---
layout: post
title:  "Using Keycloak Authorization in Strimzi"
date: 2020-08-25
author: marko_strukelj
---

In the recent blog post we have described the [OPA authorization support](https://strimzi.io/blog/2020/08/05/using-open-policy-agent-with-strimzi-and-apache-kafka/) introduced in Strimzi 0.19.0 as one of the authorization options.

Another authorization option has been available in Strimzi since version 0.17.0 that uses [Keycloak](https://www.keycloak.org/) - the popular open-source SSO server - specifically, the facility called [Keycloak Authorization Services](https://www.keycloak.org/docs/latest/authorization_services/) that allows central management of security permissions for sessions protected by OAuth 2.0 access tokens.
Strimzi 0.20.0 introduces some enhancements to address issues around token timeouts, invalidation, and runtime permission changes, so it is high time we describe in more detail how to use this authorization mechanism.

<!--more-->

The [OPA blog post](https://strimzi.io/blog/2020/08/05/using-open-policy-agent-with-strimzi-and-apache-kafka/) does a great job describing all the currently supported authorization mechanisms, and basic concepts.
We'll assume that you have read it and that we have the basics covered.

Keycloak authorization works by adding another layer on top of [OAuth authentication](https://strimzi.io/blog/2019/10/25/kafka-authentication-using-oauth-2.0/).
The OAuth authentication is performed during client session initiation, when the OAuth 2.0 access token is received and validated, then stored in session context for possible later use.
Keycloak authorization relies on the stored token. It expects that the token was created by the same issuer (the same Keycloak instance or cluster, and the same realm), and that the token is valid.
The valid token is needed in order to obtain the list of grants for the current session from Keycloak Authorization Services. The valid token is also needed to regularly refresh that list of grants during the session.

## Preconditions for using 'keycloak' authorization

There are two requirements for Keycloak authorization to operate smoothly:
* A recent version of Keycloak has to be used (version 9+)
* OAuth authentication has to be configured for Strimzi Kafka cluster - using the same Keycloak backend as used for authorization
* Token validity has to be ensured at all times which is achieved by enabling re-authentication

Re-authentication is a functionality provided by Apache Kafka for token-based authentication mechanisms, which allows the client to send a valid new token when the current one expires, and to continue using the already established secure connection, by effectively starting a new authenticated session over the existing connection.
A special listener configuration option called `maxSecondsWithoutReauthentication`, introduced in Strimzi 0.20.0, has to be set in order to enable re-authentication.
If re-authentication is not activated and Keycloak authorization is used, any operation performed after the access token has expired will result in Kafka client receiving the `org.apache.kafka.common.errors.AuthorizationException`.
In that situation the client will normally try to reinitialize the `KafkaProducer` / `KafkaConsumer` and continue sending / receiving messages.
That kind of recovery is time consuming, especially with TLS connections, and can represent a highly undesired hiccup in the otherwise smooth flow of message processing.

NOTE: In earlier versions of Strimzi the re-authentication can't be enabled through Kafka CR or any other supported mechanism.


## Custom Keycloak authorizer

When enabling `keycloak` authorization in Strimzi, a custom global authorizer is installed (implemented by `io.strimzi.kafka.oauth.server.authorizer.KeycloakRBACAuthorizer`). 
It is `global` in the sense that all Kafka requests, performed through any of the listeners flow through this authorizer for access approval.
In practice that means that while you can have multiple listeners with differently configured authentication, only requests from listeners that are configured with correct 'oauth' type authentication configuration will be subject to Keycloak Authorization Services.
Requests coming through other listeners will either all be granted (when performed by super users), or will all be denied.   
The exception is if you enable delegation of authorization checks to `simple` authorizer. In that case, the simple ACL rules will be used when Keycloak Authorization Services can't be.
Keep in mind that having central management of users is one of the value propositions of 'keycloak' authorization, and mixing users between multiple stores and rulesets makes things much harder to manage.

## Installing Keycloak

For Kubernetes deployments the Keycloak cluster can be set up using the [Keycloak Operator](https://www.keycloak.org/docs/latest/server_installation/index.html#_operator).
It works by deploying a service that communicates with Kubernetes API server, and acts upon changes in Keycloak custom resource (CR) records created in the Kubernetes API server.
Based on CR records new clusters of Keycloak are deployed, and populated with realm definitions, users, clients, groups, roles ...

For the demo purposes, however, we can deploy a simple Keycloak pod, backed by a database, so it can survive restarts.

We'll use the following `postgres-pvc.yaml` containing the persistent volume claim for our `postgres` instance:

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgres-pv-claim
  labels:
    app: postgres
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

Then, we'll use a simple stateless pod as defined in the following `postgres.yaml` for creating the Postgres database instance:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  type: NodePort
  ports:
    - port: 5432
  selector:
    app: postgres

---

apiVersion: v1
kind: Pod
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  containers:
  - name: postgres
    image: postgres
    ports:
      - containerPort: 5432
    env:
    - name: POSTGRES_DB
      value: keycloak
    - name: POSTGRES_USER
      value: kcuser
    - name: POSTGRES_PASSWORD
      value: kcuserpass
    volumeMounts:
    - name: pg-volume
      mountPath: /var/lib/postgresql/data
      subPath: postgres
  volumes:
  - name: pg-volume
    persistentVolumeClaim:
      claimName: postgres-pv-claim
``` 

We'll deploy Keycloak as a stateless pod as well using the following `keycloak.yaml` for creating the Keycloak server instance:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  labels:
    app: keycloak
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: https
    port: 8443
    targetPort: 8443
  selector:
    app: keycloak
  type: NodePort

---

apiVersion: v1
kind: Pod 
metadata:
  name: keycloak
  labels:
    app: keycloak
spec:
  containers:
  - name: keycloak
    image: jboss/keycloak
    args:
    - "-b 0.0.0.0"
    - "-Dkeycloak.profile.feature.upload_scripts=enabled"
    env:
    - name: KEYCLOAK_USER
      value: admin
    - name: KEYCLOAK_PASSWORD
      value: admin
    - name: PROXY_ADDRESS_FORWARDING
      value: "true"
    - name: KEYCLOAK_LOGLEVEL
      value: INFO
    # DB_VENDOR is very important to prevent Keycloak from automatically failing over to H2
    - name: DB_VENDOR
      value: postgres
    - name: DB_ADDR
      value: postgres.myproject.svc.cluster.local:5432
    - name: DB_DATABASE
      value: keycloak
    - name: DB_USER
      value: kcuser
    - name: DB_PASSWORD
      value: kcuserpass
    ports:
    - name: http
      containerPort: 8080
    - name: https
      containerPort: 8443
    readinessProbe:
      httpGet:
        path: /auth/realms/master
        port: 8080
```

Note the DB_ADDR env variable where we assume that we're deploying to `myproject` namespace - thus the FQDN of the postgres server contains '.myproject.'.
If the target namespace was something else, we would have to fix the DB_ADDR value.

If you're using OpenShift / Minishift / OKD the current project is set to `myproject` by default.
You can thus start up everything by issuing the following commands:

Deploy 'postgres' using persistent volume:

    kubectl apply -f postgres-pvc.yaml
    kubectl apply -f postgres.yaml

You may want to wait for postgres to start up before deploying Keycloak otherwise Keycloak will fail to start.
In that case it will be automatically restarted, but the whole start up procedure will take longer as a result.

    kubectl logs postgres -f
    
Deploy Keycloak:

    kubectl apply -f keycloak.yaml

Make sure that Keycloak is using the Postgres dialect:

    kubectl logs keycloak | grep Dialect

We could now import the realm with all the client definitions, roles, groups, authorization services definitions, but we want to explain everything step-by-step so we will use the Keycloak Admin Console.


## Exposing the Keycloak Admin Console

You can use Kubernetes port-forward CLI tool to create a tunnel to the Keycloak server port.
In another Terminal window type:

    kubectl port-forward keycloak 8080

You can now point your browser to [http://localhost:8080/auth/admin] and login with `admin` / `admin`.


## Preparing the Keycloak realm

Before we can use Keycloak for authentication and authorization, we have to prepare 'the realm' - that's a Keycloak concept for a logical SSO context with a unique store of users, clients, groups, roles, active sessions, and other configurations, and persistent data. 
There's the default realm, called `master`, but we can create new realms at any time.

Let's create a new one and call it `strimzi`.

![Create Strimzi Realm](/assets/images/posts/2020-08-25-keycloak-authz-create-realm.png)
![Realm Created](/assets/images/posts/2020-08-25-keycloak-authz-realm-created.png)

### OAuth client representing the Kafka broker

Next, we'll need an oauth client definition representing the Kafka broker (a `resource server` in OAuth 2.0 parlance).

Let's create a new Client and call it `kafka`.

![Create 'kafka' Client](/assets/images/posts/2020-08-25-keycloak-authz-add-kafka-client.png)

We need to configure it with `confidential` 'Access Type' which means that this client can keep a secret and can authenticate using clientId and secret. This makes another tab available in the UI called `Credentials`.
Set 'Standard Flow Enabled' to `Off` which disables the default SSO web flow, which we don't need since Kafka Broker has no web access.
Set 'Service Accounts Enabled' to `On` which creates a special account called 'service-account-kafka' which allows the client to act in its own name, rather than in the name of some other user. This makes another tab available in the UI called `Service Account Roles`.
And, crucially, enable Authorization Services for this account by setting 'Authorization Enabled' to `On`. This makes another tab available in the UI called `Authorization`

![Configure 'kafka' Client](/assets/images/posts/2020-08-25-keycloak-authz-configure-kafka-client.png)

Having an oauth client representing the Kafka broker with 'Authorization Services' enabled is enough to configure Strimzi.

### OAuth clients for the Kafka clients

Kafka clients need to authenticate when connecting to Kafka broker. In the [OAuth Authentication blog post](https://strimzi.io/blog/2019/10/25/kafka-authentication-using-oauth-2.0/) we demonstrated how to use confidential OAuth clients to provide a 'clientId' and a 'secret' for each Kafka client application.  

This time let's follow the example from the [previous post]((https://strimzi.io/blog/2020/08/05/using-open-policy-agent-with-strimzi-and-apache-kafka/) ), where we have several users assigned to different groups, and they authenticate as users with the username and the password, then their permissions depend on the group they are a part of.

We'll create a public oauth client definition - that's a kind of client that can not authenticate in its own name.

Let's create a new Client and call it `kafka-cli`.

![Create 'kafka-cli' Client](/assets/images/posts/2020-08-25-keycloak-authz-add-kafkacli-client.png)

We need to configure it with `public` 'Access Type' which means that this client can't keep a secret and can't authenticate in its own name.
Set 'Standard Flow Enabled' to `Off` which disables the default SSO web flow, which we don't need since Kafka client has no web access.
Set 'Direct Access Grants Enabled' to `On` which allows users to use this client with username and password.

![Configure 'kafka-cli' Client](/assets/images/posts/2020-08-25-keycloak-authz-configure-kafkacli-client.png)

Then, we'll need some users.

Let's create three users: 'tom', 'jack', and 'dean'.

On the first screen just type the username and click 'Save'. 

![Create User](/assets/images/posts/2020-08-25-keycloak-authz-add-user.png)

Then click on 'Credentials' tab and set user's password, and make sure to set 'Temporary' to `Off`.

![Set User Password](/assets/images/posts/2020-08-25-keycloak-authz-set-user-pass.png)

This is enough for user to be able to authenticate.
Create the other two users as well. We will later assign these three users the roles.


## Enabling the 'keycloak' authorization in Strimzi

Here is how you enable the `keycloak` authorization in Strimzi.

Step 1: Configure 'OAuth' authentication

```yaml
apiVersion: kafka.strimzi.io/v1beta1
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    listeners:
      plain:
        # ...
        authentication:
          type: oauth
          validIssuerUri: http://keycloak:8080/auth/realms/strimzi
          jwksEndpointUri: http://keycloak:8080/auth/realms/strimzi/protocol/openid-connect/certs
          userNameClaim: preferred_username
          maxSecondsWithoutReauthentication: 3600
      
    # ...
``` 


Step 2: Configure `keycloak` authorization using the same Keycloak realm and the clientId of the configured OAuth client that has Keycloak Authorization Services enabled:

```yaml
apiVersion: kafka.strimzi.io/v1beta1
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    # ...
    authorization:
      type: keycloak
      clientId: kafka
      tokenEndpointUri: http://keycloak:8080/auth/realms/strimzi/protocol/openid-connect/token
      
    # ...
``` 

Note: We should always use `https://` rather than `http://` when connecting to Keycloak. 
In our case we could use `https://keycloak:8443` but we would then also need to configure the truststore with Keycloak's certificate which was created on the fly, and will change if `keycloak` pod is deleted and recreated.
We could also create the Keycloak server certificate and configure Keycloak with it, but that's even more steps to do.
If it's a self-signed certificate we would also have to install it into the browser in order to prevent browser from denying us access to Keycloak Admin Console due to 'invalid' certificate.
Thus, for this example, in order to keep things simple and focus on main concepts of Keycloak Authorization Services, we're not using `https://`.
Analogously, when using any kibnd of Kafka broker authentication we can only guarantee communication integrity if using TLS, so rather than using the `plain` listener, we should be using the `tls` listener.

An example of a working configuration of a simple Kafka cluster - `kafka-oauth-keycloak.yaml`:

```yaml
apiVersion: kafka.strimzi.io/v1beta1
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    version: 2.6.0
    replicas: 1
    listeners:
      plain:
        authentication:
          type: oauth
          validIssuerUri: http://keycloak:8080/auth/realms/strimzi
          jwksEndpointUri: http://keycloak:8080/auth/realms/strimzi/protocol/openid-connect/certs
          userNameClaim: preferred_username
          
          # the next option is only available since Strimzi 0.20.0
          # remove it if using an older version of Strimzi 
          maxSecondsWithoutReauthentication: 3600
      tls: {}
    authorization:
      type: keycloak
      clientId: kafka
      tokenEndpointUri: http://keycloak:8080/auth/realms/strimzi/protocol/openid-connect/token
    logging:
      type: inline
      loggers:
        # Never use TRACE in production - you may end up with secrets in your log files
        log4j.logger.io.strimzi: "TRACE"
        log4j.logger.kafka: "DEBUG"
        log4j.logger.org.apache.kafka: "DEBUG"
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      log.message.format.version: "2.6"
    storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 100Gi
        deleteClaim: false
  zookeeper:
    replicas: 1
    storage:
      type: persistent-claim
      size: 100Gi
      deleteClaim: false
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

You can apply it by running:

    kubectl apply -f kafka-oauth-keycloak.yaml
    
    
You can observer the Kafka broker starting up once its pod is created by the operator:

    kubectl logs -f my-cluster-kafka-0 -f


## Using the CLI Kafka client to produce messages

We can run a new pod of strimzi kafka image with an interactive shell:

    kubectl run -ti --attach --image strimzi/kafka:latest-kafka-2.6.0 kafka-cli -- /bin/sh

This will show an interactive prompt once its started. The first time it may take a while since the image is fetched from the remote Docker registry.

Let's create a client configuration for a client that will produce some messages.

User's passwords are sensitive information, so we don't want them to be in configuration files in clear text or poorly encoded when they can easily be recovered.
For that reason Strimzi OAuth does not support plain username and password. But we can obtain a long-lived refresh token by using username and password, and set it in configuration.

We can use `curl` command line tool for that:

```
export HISTCONTROL=ignorespace
 USERNAME=tom
 PASSWORD=toms-password
 TOKEN_RESPONSE=$(curl http://keycloak:8080/auth/realms/strimzi/protocol/openid-connect/token -H "Content-Type: application/x-www-form-urlencoded" -d "grant_type=password&username=$USERNAME&password=$PASSWORD&client_id=kafka-cli&scope=offline_access" -s)
 REFRESH_TOKEN=$(echo $TOKEN_RESPONSE | awk -F "refresh_token\":\"" '{printf $2}' | awk -F "\"" '{printf $1}')
```

Or we can use a little script to do the same for us:

```
curl -o /tmp/oauth.sh https://raw.githubusercontent.com/strimzi/strimzi-kafka-oauth/master/examples/docker/kafka-oauth-strimzi/kafka/oauth.sh
chmod +x /tmp/oauth.sh

export HISTCONTROL=ignorespace
export TOKEN_ENDPOINT=http://keycloak:8080/auth/realms/strimzi/protocol/openid-connect/token
 USERNAME=tom
 PASSWORD=toms-password
 REFRESH_TOKEN=$(/tmp/oauth.sh $USERNAME $PASSWORD)
```

With the `REFRESH_TOKEN` set as env variable we can now write out the Kafka client configuration for user 'tom':

```
cat > ~/tom.properties << EOF
security.protocol=SASL_PLAINTEXT
sasl.mechanism=OAUTHBEARER
sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
  oauth.client.id="kafka-cli" \
  oauth.refresh.token=$REFRESH_TOKEN \
  oauth.token.endpoint.uri="http://keycloak:8080/auth/realms/strimzi/protocol/openid-connect/token" ;
sasl.login.callback.handler.class=io.strimzi.kafka.oauth.client.JaasClientOauthLoginCallbackHandler
EOF
```

Before running the client we have to ensure the necessary classes are on the classpath:

    export CLASSPATH=/opt/kafka/libs/strimzi/*:$CLASSPATH


### Producing Messages

Let's try to produce some messages to topic 'my-topic':

```
bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap:9092 --topic my-topic \
  --producer.config=$HOME/tom.properties
First message
```

This gives us an error:

    org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics: [my-topic]

It means the client has successfully authenticated, but that the user has no permission to write to topic `my-topic`.

We can take a look at Kafka broker log from another console, where we should see DEBUG logging for authorization checks.

    kubectl logs my-cluster-kafka-0

At this point we may see a runtime exception:

    Failed to parse Resource: Default Resource - part doesn't follow TYPE:NAME pattern: Default Resource

We will explain in the next chapter what is going on.

 
### Using Authorization Services to limit permissions to users 

For now we only have users with username and password which by default have no explicit grants - thus 'keycloak' authorization will treat them as having no permissions.

We'll give them permissions by describing the Kafka security model in Authorization Services, and define resources, policies, and permissions assignments.

But first, we need to change a few Authorization Services settings from their defaults.

Under the `Authorization` tab of `kafka` client we have to change the 'Decision Strategy' to `Affirmative`.

![Configuring Authorization](/assets/images/posts/2020-08-25-keycloak-authz-settings.png)

Also, under `Resources` sub-tab of the `Authorization` tab we have to remove the 'Default Resource'.

![Remove the Default Resource](/assets/images/posts/2020-08-25-keycloak-authz-delete-default-resource.png)

Its existence is responsible for the error we have seen in Kafka broker log when we first ran the CLI client.
For the purposes of Strimzi `keycloak` authorization the resource names have to conform to a specific format, which the 'Default Resource' does not.

#### Importing authorization scopes

We need to create a set of possible actions that map to Kafka's security model.
We could create them by hand, but it's easy to make a typo, so let's import them.
Download the [following file](https://raw.githubusercontent.com/strimzi/strimzi-kafka-oauth/master/oauth-keycloak-authorizer/etc/authorization-scopes.json).
Under the `Authorization` tab of `kafka` client in `Settings` sub-tabs there is the `Import` option. Use 'Select file' to upload the `authorization-scopes.json` file.

If you switch now to `Authorization Scopes` tab you should see the imported scopes.

![Imported Authorization Scopes](/assets/images/posts/2020-08-25-keycloak-authz-imported-scopes.png)

#### Authorization Services core concepts

`Keycloak Authorization Services` use several concepts that together take part in defining, and applying access controls to resources.

`Resources` define _what_ we are protecting from unauthorized access.
Each resource can contain a list of `authorization scopes` - actions that are available on the resource, so that permission on a resource can be granted specifically for one or more actions.

`Policies` define the groups of users we want to target with permissions. Users can be targeted based on group membership, assigned roles, or individually.

Finally, the `permissions` tie together specific `resources`, `authorization scopes` and `policies` to define that 'specific set of users U can perform certain actions A on the resource R'.

You can read more about `Keycloak Authorization Services` on [project's web site](https://www.keycloak.org/docs/latest/authorization_services/index.html).

Before defining resources we want to protect, let's explain the resource naming format required by Strimzi 'keycloak' authorization.
The name of every resource is not only a logical name, but rather a resource filter.
It is a matching pattern used to select the set of resources to which to apply particular policies and permissions.

The format is quite simple. For example:

- `kafka-cluster:my-cluster,Topic:a_*` targets only topics in kafka cluster 'my-cluster' with names starting with 'a_'

If `kafka-cluster:XXX` segment is not present, the specifier targets any cluster.

- `Group:x_*` targets all consumer groups on any cluster with names starting with 'x_'

The possible resource types mirror the [Kafka authorization model](https://kafka.apache.org/documentation/#security_authz_primitives) (Topic, Group, Cluster, ...).

Under `Authorization Scopes` we can see the list of all the possible actions (Kafka permissions) that can be granted on resources of different types.
It requires some understanding of [Kafka permissions model](https://kafka.apache.org/documentation/#resources_in_kafka) to know which of these make sense with which resource type.
This list mirrors Kafka permissions and should be the same for any deployment.

Under the `Policies` sub-tab there are filters that match sets of users.
Users can be explicitly listed, or they can be matched based on the Roles, or Groups they are assigned.
Policies can even be programmatically defined using JavaScript where logic can take into account the context of the client session - e.g. client ip (that is client ip of the Kafka client).

Then, finally, there is the `Permissions` sub-tab, which defines 'role bindings' where `resources`, `authorization scopes` and `policies` are tied together to apply a set of permissions on specific resources for certain users.

Each `permission` definition should have a nice descriptive name to make it very clear what kind of access is granted to which users.

For example:
    
    Dev Team A can write to topics that start with x_ on cluster dev-cluster
    
Let's create some user groups and grant different permissions to different users.


#### Joining users to groups

Groups are sets of users with name assigned. Typically they are used to geographically or organisationally compartmentalize users into organisations, organisational units, departments etc.

For our example we want three groups of users: 'Invoicing', 'Marketing', and 'Orders' which we create under 'Groups' sub-section.

![Create Group](/assets/images/posts/2020-08-25-keycloak-authz-create-group.png)
![Groups](/assets/images/posts/2020-08-25-keycloak-authz-groups.png)

Let's add user 'tom' to the 'Orders' group, and 'jack' to the 'Invoicing' group.

We do that by editing each user within 'Users' section, selecting the groups under 'Groups' tab, and using 'Join' button.

![Add User to Group](/assets/images/posts/2020-08-25-keycloak-authz-join-group.png)


#### Assigning users the roles

Roles are a concept analogous to groups. In principle either can be used for grouping and tagging users. 
The roles are usually used to 'tag' users as playing organisational roles and having permissions that pertain to their roles.
 
For our example we want three types of users, each type represented by a role. 
Let's create three roles: 'Producer', 'Consumer', and 'Admin' by using 'Roles' page.

![Add Realm Role](/assets/images/posts/2020-08-25-keycloak-authz-add-role.png)

![Realm Roles](/assets/images/posts/2020-08-25-keycloak-authz-roles.png)

Let's give user 'tom' the 'Producer' and 'Consumer' roles, user 'jack' the 'Consumer' role, and user 'dean' the 'Admin' role.

We do that by editing each user within 'Users' section, selecting and adding the roles under the 'Role Mappings' tab.

![Assign Roles](/assets/images/posts/2020-08-25-keycloak-authz-assign-role.png)
![Assign Roles](/assets/images/posts/2020-08-25-keycloak-authz-assigned-roles.png)

#### Creating topics

We still need to create topics to represent the queues, and we can then start granting permissions.

In `Authorization` we create a new resource called `Topic:new-orders`.

![Create 'new-orders' Topic Resource](/assets/images/posts/2020-08-25-keycloak-authz-new-topic-resource-new-orders.png)

And another one called `Group:new-orders`.

![Create 'new-orders' Group Resource](/assets/images/posts/2020-08-25-keycloak-authz-new-group-resource-new-orders.png)

We add all the relevant `authorization scopes` to each resource. 
For topics these are `Create`, `Delete`, `Describe`, `Write`, `Read`, `Alter`, `DescribeConfigs`, `AlterConfigs`.
For groups these are `Describe`, `Read`, `Delete`, `DescribeConfigs`, `AlterConfigs`.

Ideally we could declare resource type `Topic` and the possible `authorization scopes` would automatically be added based on type definition.
Currently there is no such facility.

Let's make another pair: `Topic:processed-orders`, `Group:orders-*`


#### Targetting permissions

Imagine the following business process. An automated system enters a new order into 'new-orders' queue.
A person from Orders department takes the order from the queue, fulfills it, and writes it into 'processed-orders' queue.
Then a person from Invoices department takes an order from 'processed-orders', and prepares invoices for it.
 
We want people from Orders department to only have access to their queues, and similarly for people from Invoices department.

We've already prepared users, to model the people, and groups to model the departments.
We have prepared roles to give different levels of access to people within departments - 'Producers' can add to topics, 'Consumers' can only read from topics, and 'Admins' can create and configure the topics.    

We have also created the resources to target authorization rules, next we need to target these resources based on roles and groups.


We use `Policies` tab to map realm Groups, and realm Roles into authorization services for targeting.

The existing `Default Policy` can safely be removed.

Then, let's create a 'Role' policy called 'Has Producer role'

![Create 'Has Producer role' Policy](/assets/images/posts/2020-08-25-keycloak-authz-has-producer-role-policy.png)

And a 'Group' policy called 'Is member of Orders group'

![Create 'Is member of Orders group' Policy](/assets/images/posts/2020-08-25-keycloak-authz-group-policy.png)

We can now create a scope-based permission under 'Permissions' tab, and give it a very descriptive name: 'Accounts with Producer role and Orders group can produce to 'new-orders' topic'.

![Add permission to produce](/assets/images/posts/2020-08-25-keycloak-authz-add-permissions.png)

User 'tom' should now be able to produce to 'new-orders' topic.

Let's test that.

```
bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap:9092 --topic new-orders \
  --producer.config=$HOME/tom.properties
First message
```


TODO (find a place): 
But if you think about it - this way of 'tagging' users through groups and roles is a very simple model.
Adding a role or a group to the user is just like adding a tag without any logical operators.
As a result we can't make the user a 'Producer' in the 'Orders' group, while only a 'Consumer', but not a 'Producer' in the 'Invoices' group without a special 'tag' for it.
For that we would need 'tags' like 'orders_producer', 'invoices_consumer'. But that quickly becomes hard to maintain.

This is where Authorization Services come to the rescue, since they allow creating composite conditional rules to target permissions to sets of users. 







 
        

Clients `team-a-client`, and `team-b-client` are confidential clients representing services with partial access to certain Kafka topics.


## Targeting Permissions - Clients and Roles vs. Users and Groups
  
In Keycloak, confidential clients with 'service accounts' enabled can authenticate to the server in their own name using a clientId and a secret.
This is convenient for microservices which typically act in their own name, and not as agents of a particular user (like a web site would, for example).
Service accounts can have roles assigned like regular users.
They can not, however, have groups assigned.
As a consequence, if you want to target permissions to microservices using service accounts, you can't use Group policies, but are forced to use Role policies.
Or, thinking about it another way, if you want to limit certain permissions only to regular user accounts where authentication with username and password is required, you should use Group policies, rather than Role policies.

That's what we see used in `permissions` that start with 'ClusterManager'.
Performing cluster management is usually done interactively - in person - using CLI tools. 
It makes sense to require the user to log-in, before using the resulting access token to authenticate to Kafka Broker.
In this case the access token represents the specific user, rather than the client application.




`team-a-client` has a `Dev Team A` role which gives it permissions to do anything on topics that start with 'a_', and only write to topics that start with 'x_'.
The topic named `my-topic` matches neither of those.

Use CTRL-C to exit the CLI application, and let's try to write to topic `a_messages`.

```
bin/kafka-console-producer.sh --broker-list kafka:9092 --topic a_messages \
  --producer.config ~/team-a-client.properties
First message
Second message
```

Although we can see some unrelated warnings, looking at the Kafka container log there is DEBUG level output saying 'Authorization GRANTED'.

Use CTRL-C to exit the CLI application.


### Consuming Messages

Let's now try to consume the messages we have produced.

    bin/kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic a_messages \
      --from-beginning --consumer.config ~/team-a-client.properties

This gives us an error like: `Not authorized to access group: console-consumer-55841`.

The reason is that we have to override the default consumer group name - `Dev Team A` only has access to consumer groups that have names starting with 'a_'.
Let's set custom consumer group name that starts with 'a_'

    bin/kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic a_messages \
      --from-beginning --consumer.config ~/team-a-client.properties --group a_consumer_group_1

We should now receive all the messages for the 'a_messages' topic, after which the client blocks waiting for more messages.

Use CTRL-C to exit.


### Using Kafka's CLI Administration Tools

Let's now list the topics:

    bin/kafka-topics.sh --bootstrap-server kafka:9092 --command-config ~/team-a-client.properties --list
    
We get one topic listed: `a_messages`.

Let's try and list the consumer groups:

    bin/kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
      --command-config ~/team-a-client.properties --list

Similarly to listing topics, we get one consumer group listed: `a_consumer_group_1`.

There are more CLI administrative tools. For example we can try to get the default cluster configuration:

    bin/kafka-configs.sh --bootstrap-server kafka:9092 --command-config ~/team-a-client.properties \
      --entity-type brokers --describe --entity-default

But that will fail with `Cluster authorization failed.` error, because this operation requires cluster level permissions which `team-a-client` does not have.


### Client with Different Permissions

Let's prepare a configuration for `team-b-client`:

```
cat > ~/team-b-client.properties << EOF
security.protocol=SASL_PLAINTEXT
sasl.mechanism=OAUTHBEARER
sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
  oauth.client.id="team-b-client" \
  oauth.client.secret="team-b-client-secret" \
  oauth.token.endpoint.uri="http://keycloak:8080/auth/realms/kafka-authz/protocol/openid-connect/token" ;
sasl.login.callback.handler.class=io.strimzi.kafka.oauth.client.JaasClientOauthLoginCallbackHandler
EOF
```

If we look at `team-b-client` client configuration in Keycloak, under `Service Account Roles` we can see that it has `Dev Team B` realm role assigned.
Looking in Keycloak Console at the `kafka` client's `Authorization` tab where `Permissions` are listed, we can see the permissions that start with 'Dev Team B ...'.
These match the users and service accounts that have the `Dev Team B` realm role assigned to them. 
The `Dev Team B` users have full access to topics beginning with 'b_' on Kafka cluster `cluster2` (which is the designated cluster name of the demo cluster we brought up), and read access on topics that start with 'x_'.

Let's try produce some messages to topic `a_messages` as `team-b-client`:

```
bin/kafka-console-producer.sh --broker-list kafka:9092 --topic a_messages \
  --producer.config ~/team-b-client.properties
Message 1
```

We get `Not authorized to access topics: [a_messages]` error as we expected. Let's try to produce to topic `b_messages`:

```
bin/kafka-console-producer.sh --broker-list kafka:9092 --topic b_messages \
  --producer.config ~/team-b-client.properties
Message 1
Message 2
Message 3
```

This should work fine.

What about producing to topic `x_messages`. `team-b-client` is only supposed to be able to read from such a topic.

```
bin/kafka-console-producer.sh --broker-list kafka:9092 --topic x_messages \
  --producer.config ~/team-b-client.properties
Message 1
```

We get a `Not authorized to access topics: [x_messages]` error as we expected. 
Client `team-a-client`, on the other hand, should be able to write to such a topic:

```
bin/kafka-console-producer.sh --broker-list kafka:9092 --topic x_messages \
  --producer.config ~/team-a-client.properties
Message 1
```

However, we again receive `Not authorized to access topics: [x_messages]`. What's going on?
The reason for failure is that while `team-a-client` can write to `x_messages` topic, it does not have a permission to create a topic if it does not yet exist.

We now need a power user that can create a topic with all the proper settings - like the right number of partitions and replicas.


### Power User Can Do Anything

Let's create a configuration for user `bob` who has full ability to manage everything on Kafka cluster `cluster2`.

First, `bob` will authenticate to Keycloak server with his username and password and get a refresh token.

```
export TOKEN_ENDPOINT=http://keycloak:8080/auth/realms/kafka-authz/protocol/openid-connect/token
REFRESH_TOKEN=$(./oauth.sh -q bob)
```

This will prompt you for a password. Type 'bob-password'.

We can inspect the refresh token:

    ./jwt.sh $REFRESH_TOKEN

By default this is a long-lived refresh token that does not expire.

Now we will create the configuration file for `bob`:

```
cat > ~/bob.properties << EOF
security.protocol=SASL_PLAINTEXT
sasl.mechanism=OAUTHBEARER
sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
  oauth.refresh.token="$REFRESH_TOKEN" \
  oauth.client.id="kafka-cli" \
  oauth.token.endpoint.uri="http://keycloak:8080/auth/realms/kafka-authz/protocol/openid-connect/token" ;
sasl.login.callback.handler.class=io.strimzi.kafka.oauth.client.JaasClientOauthLoginCallbackHandler
EOF
```

Note that we use the `kafka-cli` public client for the `oauth.client.id` in the `sasl.jaas.config`. 
Since that is a public client it does not require any secret.
We can use the public client because we configure the token directly (in this case a refresh token is used to request an access token behind the scenes which is then sent to Kafka broker for authentication, and we already did the authentication when obtaining the token in the first place).


Let's now try to create the `x_messages` topic:

    bin/kafka-topics.sh --bootstrap-server kafka:9092 --command-config ~/bob.properties \
      --topic x_messages --create --replication-factor 1 --partitions 1

The operation should succeed (you can ignore the warning about periods and underscores).
We can list the topics:

    bin/kafka-topics.sh --bootstrap-server kafka:9092 --command-config ~/bob.properties --list

If we try the same as `team-a-client` or `team-b-client` we will get different responses.

    bin/kafka-topics.sh --bootstrap-server kafka:9092 --command-config ~/team-a-client.properties --list
    bin/kafka-topics.sh --bootstrap-server kafka:9092 --command-config ~/team-b-client.properties --list

Roles `Dev Team A`, and `Dev Team B` both have `Describe` permission on topics that start with 'x_', but they can't see the other team's topics as they don't have `Describe` permissions on them.

We can now again try to produce to the topic as `team-a-client`.

```
bin/kafka-console-producer.sh --broker-list kafka:9092 --topic x_messages \
  --producer.config ~/team-a-client.properties
Message 1
Message 2
Message 3
```

This works.

If we try the same as `team-b-client` it should fail.

```
bin/kafka-console-producer.sh --broker-list kafka:9092 --topic x_messages \
  --producer.config ~/team-b-client.properties
Message 4
Message 5
```

We get an error - `Not authorized to access topics: [x_messages]`.

But `team-b-client` should be able to consume messages from the `x_messages` topic:

    bin/kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic x_messages \
      --from-beginning --consumer.config ~/team-b-client.properties --group x_consumer_group_b

Whereas `team-a-client` does not have permission to read, even though they can write:

    bin/kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic x_messages \
      --from-beginning --consumer.config ~/team-a-client.properties --group x_consumer_group_a

We get a `Not authorized to access group: x_consumer_group_a` error.
What if we try to use a consumer group name that starts with 'a_'?

    bin/kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic x_messages \
      --from-beginning --consumer.config ~/team-a-client.properties --group a_consumer_group_a
    
We now get a different error: `Not authorized to access topics: [x_messages]`

It just won't work - `Dev Team A` has no `Read` access on topics that start with 'x_'.

User `bob` should have no problem reading from or writing to any topic:

    bin/kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic x_messages \
      --from-beginning --consumer.config ~/bob.properties

