# Gravitee.io Kafka Gateway Lab

Here is a docker-compose to run APIM with Native Kafka enabled.

## Run docker-compose

⚠️ You need a license file to be able to run Enterprise Edition of APIM. Do not forget to add your license file into `./.license`.

Docker compose will create the following services :
- `gio_apim_mongodb` : MongoDB database
- `gio_apim_elasticsearch` : Elasticsearch database
- `gio_apim_mailhog` : Mailhog service
- `gio_apim_gateway` : Famous Gravitee Gateway with Kafka enabled
- `gio_apim_management_api` : Gravitee Management Rest-API
- `gio_apim_management_ui` : Gravitee Management Console UI
- `gio_apim_portal_ui` : Gravitee Portal UI
- `gio_apim_kafka` : Kafka broker which must be accessible via Gravitee Gateway
- `gio_apim_kafka-ui` : Simple Kafka UI to see topics and messages. Useful for testing
- `gio_apim_kafka-client` : Kafka client container to run kafka commands. This avoids to configure kafka-client on your local machine.

Docker volumes :
- `./.license` : License file
- `./.ssl` : Contains SSL certificates for this example
- `./.kafka-client-config` : Contains kafka client configuration files. To help to run kafka client commands easily

Up docker-compose :
`docker-compose up -d`

# APIM Quick Setup with Native Kafka

In this example, we will create a Kafka API and consume it with a Kafka client using the gateway.
The Gravitee gateway exposes a Kafka port 9092. It also requires an SSL configuration to use SNI.
In this example, we will use a self-signed wildcard certificate `*.kafka.local`.

## General Configuration

The gateway is already configured with the static part of the `kafka.local` certificate via the environment variable `gravitee_kafka_routingHostMode_defaultDomain`.\
To configure this in the different UI applications, it must be set in the Console UI.
- Go to the console `http://localhost:8084/` (user: admin, password: admin) 
- Then: Organization > Entrypoints & Sharding Tags > Entrypoint Configuration. 
- Set the Default Kafka Domain to `{apiHost}.kafka.local`.


## Create a APIM Kafka API
1. Go to the console `http://localhost:8084/` > API > Add API > Create API
2. Enter a name and version, e.g., `My Kafka Gateway API` and `1.0.0`
3. Select Kafka Protocol
4. Configure entrypoints  
  This allows you to create the host on which the API will be used via the gateway. The host prefix in our case will be contained in the wildcard part of the certificate `*.kafka.local`.\
  Enter `foo` or `bar`. Only these two values work in our example. To add more, you need to specify them in the docker-compose `services.gateway.networks.gateway.aliases`.\
  This limitation in our example is due to the simplified local installation and reliance on Docker DNS to resolve domain names.

> NOTE: To add other hosts, you need to add two aliases in our example:
> - `[hostPrefix].kafka.local` The address for the Kafka bootstrap server for the client
> - `broker-0-[hostPrefix].kafka.local` The address of the broker with node id 0 that the client will use. In our case, we have only one broker, so 0.  
> In a real use case with a wildcard certificate, this part is invisible. However, if you do not want to use a wildcard certificate, you need to create certificates for each Kafka broker.\
> (The prefix and delimiter are configurable if necessary)


5. Configure endpoints \
This configuration allows the connection/security between the gateway and the Kafka broker.

    - Option 1: PLAINTEXT \
      Specify the Kafka broker bootstrap. In our case, it is the service: `kafka:9091`.
      Select PLAINTEXT as the security protocol.

    - Option 2: SSL \
      Specify the Kafka broker bootstrap intended for SSL. In our case, it is the service: `kafka:9094`.    
      Select SSL as the security protocol.
      Configure a Truststore "JKS With Path" with `./ssl/kafka-client.truststore.jks` and the password `password`.

    - Other options \
      The docker-compose contains other configurations to use SASL or even SASL_SSL, but you will need to customize the configuration.\
      We can even implement a "Path Through" concept with SASL and OAUTHBEARER_TOKEN by configuring an EL for the Token, e.g., `{#context.principal.token}`. This retrieves the token from the "client connection to gateway" to pass it to the broker.

6. Security \
In our example, we keep the Keyless plan. It will always be possible to change the plan later.

7. Review your API configuration \
Validate with Save & Deploy API.

## Produce & Consume with Kafka client
Now that we have created and started an API, we can produce and consume messages with a Kafka client.
Since we have a keyless plan, no subscription is required, and we can use the `kafka-keyless-plan-ssl.properties` configuration, which enables SSL and configures the truststore.

Execute the following commands to produce and consume messages.
```bash
docker exec -it gio_apim_kgw_kafka-client bash -c "kafka-console-producer.sh --bootstrap-server foo.kafka.local:9092 --producer.config config/kafka-keyless-plan-ssl.properties --topic client-topic-1"
```
```bash
docker exec -it gio_apim_kgw_kafka-client bash -c "kafka-console-consumer.sh --bootstrap-server foo.kafka.local:9092 --consumer.config config/kafka-keyless-plan-ssl.properties --topic client-topic-1"
```

## Secure My API with an API Key

In this example, we will use the API Key Plan.
For other plans like JWT and OAuth2, you need to configure the providers, which is beyond the scope of this example.

Before starting, enable the next-gen (v2) Gravitee.io Developer Portal:

Activate the new portal in the environment settings (Settings > Settings > Enable the New Developer Portal)
Publish the `My Kafka Gateway API` API with the "Publish the API" button on the main page of the API


1. Modify the My Kafka Gateway API > Consumer > Plan > Add new Plan > API Key
2. Nothing specific to Kafka here, it's a standard plan in APIM. Add a Name and finish creating the plan
3. Publish the plan
A dialog will open to confirm the closure of the unsecured Keyless plan and the opening of the secured API Key plan.
> It is not possible to have an unsecured plan and secured plans at the same time.
4. Deploy the API now out of sync

5. Go to the next portal with the URL `http://localhost:8085/next/`

6. Subscribe to my API with the "Default Application"
Create a subscription between the application and the API via the developer portal.\

7. Complete the Kafka client configuration file `kafka-api-key-ssl.properties` with the API key as the password and an MD5 of the API key as the username, you can get that from the developer portal (My Kafka Gateway API > My Subscription > Open it > And use the information there)

8. Produce and consume messages with the Kafka client 

```bash
docker exec -it gio_apim_kgw_kafka-client bash -c "kafka-console-producer.sh --bootstrap-server foo.kafka.local:9092 --producer.config config/kafka-api-key-plan-ssl.properties --topic client-topic-1"
```
```bash
docker exec -it gio_apim_kgw_kafka-client bash -c "kafka-console-consumer.sh --bootstrap-server foo.kafka.local:9092 --consumer.config config/kafka-api-key-plan-ssl.properties --topic client-topic-1"
```

## Control access to Kafka with ACL at the Gateway level

Gravitee has multiple policies that can be applied at the gateway level to secure,control and govern access to the Kafka cluster.

In this example we will use the ACL policy to limit which topics consumers and producers can interact with and limit rights.

1. Modify the `My Kafka Gateway API`, Add a new flow in the common flows section. `My Kafka Gateway API` > Policies > Create a flow under the Common section (click on “+” next to Common Flow and give it a name or leave it blank).

2. Add an ACL policy to the flow. Stay on the Global tab of the flow and click on the "+" in the interact phase between the Client and Broker. Select the ACL policy.

3. Configure the ACL policy to only allow `read, write` to the `topic` `prefixed` with `client-topic-`.
ACL Policy > ACL > Resource > + Add

    - Select Option >  `Topic`
    - Resource Pattern Type > `Prefixed`
    - Resource Pattern > `client-topic-`
    - Topic Opearations > Select `Read, Write`

4. Configure the ACL policy (same as above) to allow `Groups` to be `read` (this is necessary for consumers otherwise they won't be able to get Consumer Groups)
ACL Policy > ACL > Resource > + Add

    - Select Option >  `Group`
    - Resource Pattern Type > `ANY`
    - Group Opearations > Select `Read`

5. Save the ACL Policy

6. Deploy the API (out of sync)

7. Produce and consume messages with the Kafka client to test that the ACL policy is working correctly

    - The following will drop Unauthorized errors when trying to produce or consume messages (note that the topic prefix doesn't match):

      ```bash
      docker exec -it gio_apim_kgw_kafka-client bash -c "kafka-console-producer.sh --bootstrap-server foo.kafka.local:9092 --producer.config config/kafka-api-key-plan-ssl.properties --topic client-1"
      ```

      ```bash
      docker exec -it gio_apim_kgw_kafka-client bash -c "kafka-console-consumer.sh --bootstrap-server foo.kafka.local:9092 --consumer.config config/kafka-api-key-plan-ssl.properties --topic client-1"

    - The following will be go through when trying to produce or consume messages (except for the initial UNKNOWN_TOPIC_OR_PARTITION message at the first produced message):

    ```bash
    docker exec -it gio_apim_kgw_kafka-client bash -c "kafka-console-producer.sh --bootstrap-server foo.kafka.local:9092 --producer.config config/kafka-api-key-plan-ssl.properties --topic client-topic-2"
    ```

    ```bash
    docker exec -it gio_apim_kgw_kafka-client bash -c "kafka-console-consumer.sh --bootstrap-server foo.kafka.local:9092 --consumer.config config/kafka-api-key-plan-ssl.properties --topic client-topic-2"


## Create customer friendly topic naming with the Topic Mapping policy

In this example we will use the Topic Mapping policy to change the name of the topic exposed through the gateway to the client without changing the topic name in the Kafka cluster. This can be useful to avoid client modifications during a migration or to harmonize.

1. Modify the `My Kafka Gateway API`, use the previous flow created in the above example for the ACL policy or create a new one if you skipped the previous section. Add a Kafka `Topic Mapping policy` in the Interact Phase
`My Kafka Gateway API > Policies > Common Flow > Interact Phase > + Add a Topic Mapping Policy`

2. Configure the Kafka Topic Mapping policy as follow:
      - Client-side-name: `customer`
      - Broker-side-name: `client-topic-1`
  Save the policy.

3. If you already have a Kafka ACL Policy in the flow, you'll need to either disable it ('...' on the Kafka ACL policy icon and `disable`) or edit it and add a new ACL resource to match the topic name `customer`.
ACL Policy > ACL > Resource > + Add

    - Select Option >  `Topic`
    - Resource Pattern Type > `Match`
    - Resource Pattern > `customer`
    - Topic Opearations > Select `Read, Write`
  Save the policy.

4. Save the flow changes and deploy the API.

5. Produce and consume messages with the Kafka client to test that the Topic Mapping and new ACL policy is working correctly

    - The following will drop Unauthorized errors when trying to produce or consume messages (the topic client-topic-1 doesn't exist anymore when topic mapping is used ):

      ```bash
      docker exec -it gio_apim_kgw_kafka-client bash -c "kafka-console-producer.sh --bootstrap-server foo.kafka.local:9092 --producer.config config/kafka-api-key-plan-ssl.properties --topic client-topic-1"
      ```

      ```bash
      docker exec -it gio_apim_kgw_kafka-client bash -c "kafka-console-consumer.sh --bootstrap-server foo.kafka.local:9092 --consumer.config config/kafka-api-key-plan-ssl.properties --topic client-topic-1"

    - The following will be go through when trying to produce or consume messages:

    ```bash
    docker exec -it gio_apim_kgw_kafka-client bash -c "kafka-console-producer.sh --bootstrap-server foo.kafka.local:9092 --producer.config config/kafka-api-key-plan-ssl.properties --topic customer"
    ```

    ```bash
    docker exec -it gio_apim_kgw_kafka-client bash -c "kafka-console-consumer.sh --bootstrap-server foo.kafka.local:9092 --consumer.config config/kafka-api-key-plan-ssl.properties --topic customer"

