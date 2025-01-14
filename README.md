# Nussknacker Scenario Examples Library

The project provides:
1. A tool for bootstrapping, running example scenarios, mocking external services (currently DB and OpenAPI service) and generating example 
   data for the example scenario (it's in the `scenario-examples-bootstrapper` dir) - let's call it Scenario Examples Bootstrapper
2. Scenario examples definitions (it's in the `scenario-examples-library` dir) - it's Scenario Examples Library

The main purpose is to provide a simple way for creating Nu scenarios that can be treated as examples. 
An example scenario has the following capabilities:
1. It can be deployed using the Scenario Examples Bootstrapper Docker image in any Nu environment 
2. It contains mocks of external services the example scenario depends on
3. it contains data generators to generate some data, to show users the deployed example scenario in action

## Building the docker image locally

```bash
docker buildx build --tag touk/nussknacker-example-scenarios-library:latest .
```

## Publishing

The publish process is done by the Github Actions. See `.github/workflows/publish.yml`. It's going to publish docker image with two tags:
`latest` and with version taken from the `version` file.

## Testing

The Library can be tested with Nu Quickstart. You can locally test your changes with the following script:

```bash
.github/workflows/scripts/test-with-nu-quickstart.sh
```

## Running the library

To illustrate how to use the image, we're going to use a docker-compose.yml snippet:

```yaml
services:

  nu-example-scenarios-library:
    image: touk/nussknacker-example-scenarios-library:latest
    environment:
      NU_DESIGNER_ADDRESS: "designer:8080"
      NU_REQUEST_RESPONSE_OPEN_API_SERVICE_ADDRESS: "designer:8181"
      KAFKA_ADDRESS: "kafka:9092"
      SCHEMA_REGISTRY_ADDRESS: "schema-registry:8081"
    volumes:
      - nussknacker_designer_shared_configuration:/opt/nussknacker/conf/

  [...]

  designer:
    image: touk/nussknacker:latest_scala-2.12
    environment:
      EXAMPLE_SCENARIOS_LIBRARY_SERVICE_NAME: nu-example-scenarios-library
      CONFIG_FILE: "/opt/nussknacker/conf/application.conf,/opt/nussknacker/conf/additional-configuration.conf"
    [...]
    volumes:
      - nussknacker_designer_shared_configuration:/opt/nussknacker/conf
    
  [...]

volumes:
  nussknacker_designer_shared_configuration:
    name: nussknacker_designer_shared_configuration
```

### ENVs:

#### Used by the `nu-example-scenarios-library` service

- `NU_DESIGNER_ADDRESS` - it contains the address (with port) of the Designer API. It's used to import and deploy scenarios and for Nu 
  configuration reloading. You should always configure one. 
- `NU_REQUEST_RESPONSE_OPEN_API_SERVICE_ADDRESS` - it contains the address (with port) of the server which exposes Request-Response 
  scenarios. You will need it when you want to run Request-Response scenario example using the library with requests generator.
- `KAFKA_ADDRESS` - it contains the address (With port) of a Kafka service. You will need it when you want to run streaming examples 
  with Kafka sources. It's used to create topics and by generator to generate example messages.
- `SCHEMA_REGISTRY_ADDRESS` - it contains the address (with port) of a Schema Registry service. You will need it when you want to run 
  streaming examples with Kafka sources. It's used to create schemas for Kafka topics.
- `FLINK_SQL_GATEWAY_ADDRESS` - it contains the address (with port) of the [Flink SQL Gateway](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/table/sql-gateway/overview/). You will need it when you want to run batch examples. It's used to create Flink tables and insert data.

#### Used by the `designer` service 

- `EXAMPLE_SCENARIOS_LIBRARY_SERVICE_NAME` - it's the address (without port) of the Example Scenarios Library service. You can use this env
  in designer custom configurations (required by your example). Eg. `${EXAMPLE_SCENARIOS_LIBRARY_SERVICE_NAME}:5432` is the mock Postgres service, `${EXAMPLE_SCENARIOS_LIBRARY_SERVICE_NAME}:8080` is the mock HTTP service exposed by the Library. 

### Nu Configuration

Some example scenarios require to provide a custom Nussknacker configuration (e.g. DB definition or Open API service definitions). 
The Scenario Examples Bootstrapper is able to customize Nu service configuration and reload it using Nu API. But it has to have
an access to the shared `additional-configuration.conf` file. The Bootstrapper is going to put each scenario example configuration file close
to the `additional-configuration.conf` and add proper "[include](https://github.com/lightbend/config/blob/main/HOCON.md#includes)" in this file.
In the docker compose case (see the example above) to achieve it, you should: 
1. create a shared configuration volume and mount it in `nu-example-scenarios-library` and `designer` services
2. include `/opt/nussknacker/conf/additional-configuration.conf` in the `CONFIG_FILE` ENV value

### Other configuration

#### Using Scenario Examples Bootstrapper with scenario defined outside

You don't have to put your scenario in the Library to be able to use it with the Scenario Examples Bootstapper. All you need to do is to mount
your example from e.g. `my-scenario` folder to `/scenario-examples/my-scenario` in the container. Your scenario will be read and bootstrapped.

#### Disabling specific example scenarios

You can disable a specific example by setting e.g. `LOAN_REQUEST_DISABLED: true` - this ENV disables `loan-request` scenario example. 
So, all you need to do is to take name of your example (the example's folder name), change `-` to `_`, uppercase it and add `_DISABLED` - 
it gives you the name of the ENV. Don't forget to set `true` as its value.

#### Disabling all (embedded) example scenarios

You can disable all examples embedded into library by setting `DISABLE_EMBEDDED_EXAMPLES: true`.

#### Disabling data generation for all scenarios

You can disable only data generation for all the examples from the library by setting `DISABLE_DATA_GENERATION: true`

#### Disabling scenario deployment

You can disable scenario deployment (in fact, the scenario will be deployed but then it will be canceled) for a specific
example by setting e.g. `LOAN_REQUEST_DEPLOY: false` - this ENV ensures that the `loan-request' scenario example is not 
active when the data data generation is started.

### Additional outside requirements

#### Flink SQL Gateway

Currently, Bootstrapper communicates with Flink using the Flink SQL Gateway. 
You can run it like this:

```bash
./bin/sql-gateway.sh start-foreground -Dsql-gateway.endpoint.rest.address=flink-jobmanager -Drest.address=flink-jobmanager
```

<details>
  <summary>SQL Gateway docker compose example </summary>
  
    ```yaml
    services:

      flink-jobmanager:
        [...]

      flink-sql-gateway:
        image: flink:1.19.1-scala_2.12-java11
        restart: unless-stopped
        entrypoint: >
          /bin/sh -c "./bin/sql-gateway.sh start-foreground -Dsql-gateway.endpoint.rest.address=flink-jobmanager -Drest.address=flink-jobmanager"
        environment:
          JOB_MANAGER_RPC_ADDRESS: "flink-jobmanager"
        depends_on:
          flink-jobmanager:
            condition: service_healthy
        deploy:
          resources:
            limits:
              memory: 1024M
    ```
  </details>

## What's underneath and how it works

### Scenario Examples Library

In the `scenario-examples-library` you will find definition of Nu scenarios, their mocks and example data generators (for showcase purposes). 
We want to build and develop the library and we hope new examples will be added to it in the future. If you want to create an example of Nu 
scenario, don't hesitate to put it here. 

### Scenario Examples Bootstrapper

In the `scenario-examples-bootstrapper` you will find the code of the service which is responsible for:
* creating mocks required by the example scenarios (DB mocks and OpenAPI mocks)
* configuring Nu Designer, importing, and deploying the example scenarios
* configuring Nu stack dependencies (like Kafka and Schema Registry)
* running data generators for the example scenarios

#### Mocks

There is one PostgreSQL-based database which can be used by Nu's database components. Each example service can define DDL scripts that will be 
imported by the DB instance.

The second type of mock is an OpenAPI service. It's based on the Wiremock. You can deliver OpenAPI definition that will be served by 
the Wiremock server. Moreover, you can create Wiremock mappings to instruct the mock on how to respond when your mocked OpenAPI service 
is called during scenario execution.

#### Scenario setup

You are supposed to deliver the example in the Nu JSON format. Some examples needs to customize the Nu Designer configuration, so you can
provide the custom parts of configuration too. And obviously you can instruct the Bootstrapper what topics and JSON schemas are required
by the scenario - it'll configure them too.

#### Data generators

To see the example in action, some data has to be provided. This is why Bootstrapper allows you to deliver these example data by defining:
- them statically - in files with listed Kafka messages or HTTP request bodies (in case of the Request-Response scenarios). It's run only 
  on the Boostrapper service starts.
- generator - it's a bash script that produces Kafka messages of HTTP request bodies. It's used by the Bootstapper to continuously generate
  and send the example data to the scenarios sources.

## Creating an example scenario

Structure for a folder with a Scenario Example definition:

```bash
scenario-examples-library
├── {scenario-example-1} # folder with all things needed by the example scenario
│   ├── {name-of-scenario-example-1}.json # file with scenario (required)
│   ├── data # static data and data generator scripts (optional)
│   │   ├── kafka
│   │   │   ├── generated
│   │   │   │   └── {topic-01-name}.sh # script to generate message which will be sent to the topic "topic-01-name" (it will be called continuously)
│   │   │   └── static
│   │   │       └── {topic-01-name}.txt # list of messages which will be sent to topic "topic-01-name" (to send only once)
│   │   ├── http
│   │   │   ├── generated
│   │   │   │   └── {open-api-service-slug}.sh # script to generate request body which will be sent with POST request to /scenarios/{open-api-service-slug} service (it will be called continuously)
│   │   │   └── static
│   │   │       └── {open-api-service-slug}.txt # list of request bodies which will be sent with POST request to /scenarios/{open-api-service-slug} service (to send only once)
|   |   └── flink
│   │       └── static
│   │           └── {dml-script}.sql # file with Flink SQL scripts that provides some data to test
│   ├── mocks # mock definitions (optional)
│   │   └── db 
│   │       ├── {db-schema-01-name}.sql # script with DDLs to import 
│   │       └── {db-schema-02-name}.sql
│   │   └── http-service
│   │       └── {external-open-api-service-name} # name of an external Open API service
│   │           ├── __files
│   │           │   └── {external-open-api-service-name}
│   │           │       ├── openapi
│   │           │       │   └── {api-name}.yaml # it contains the external Open API service definitions. Exposed as Wiremock's static files
│   │           │       └── responses
│   │           │           ├── {some-response-01-name}.json # contains mock response - it can be used in the mapping definition
│   │           │           └── {some-response-02-name}.json
│   │           └── mappings
│   │               └── {external-open-api-service-name}
│   │                   ├── {endpoint-1-mapping}.json # definition of Wiremock's mappings - it describes how the mock service should respond
│   │                   └── {endpoint-2-mapping}.json
│   └── setup # setup Nu Designer configuration, Kafka's topics ans JSON schemas (optional)
│       ├── kafka
│       │   └── topics.txt # it contains list of topics name which should be created (topic per line)
│       ├── nu-designer
│       │   ├── {some-configuration-01-name}.conf # it contains part of Nu configuration (it's HOCON file)
│       │   └── {some-configuration-02-name}.conf
│       ├── schema-registry
│       │   ├── {topic-01-name}.schema.json # it contains JSON schema definition for topic "topic-01-name"
│       │   └── {topic-02-name}.schema.json
|       └── flink 
│           └── {ddl-script}.sql # it contains Flink SQL script to create Flink tables
└── {scenario-example-2} # the next scenario
    ├── [...]
```

### Scenario JSON

It's a representation of a Scenario in form of JSON. Nu Designer should be able to import it. The name of the file with the scenario 
is going to be used by the Bootstrapper during empty scenario creation. It should be unique.

>❗ Bootstrapper is able to determine the scenario processing mode based on the scenario metadata from the JSON in case of a Streaming and 
Request-Response scenario. But it cannot distinguish between a Streaming and Batch scenario. So, to give the Bootstrapper a hint about 
processing mode, you can add `batch` part at the end of JSON file name (eg. `my-example-batch.json`). You can do the same in case of 
a Streaming (by adding `streaming`) and Request-Response (by adding `request-response`), but it's not required.

### Scenario setup

#### Nu Designer configuration

If you want to add custom configuration you can create a HOCON file and add it in `{scenario-name}/setup/nu-designer` folder. 
In the file you can refer to mock services using the `EXAMPLE_SCENARIOS_LIBRARY_SERVICE_NAME` variable. 
E.g. PostgresSQL DB address is `${EXAMPLE_SCENARIOS_LIBRARY_SERVICE_NAME}:5432` and the Wiremock server address is 
`${EXAMPLE_SCENARIOS_LIBRARY_SERVICE_NAME}:8080`. 

>❗ Your configuration files will be included in the `additional-configuration.conf`. You have to be sure that the `additional-configuration.conf` can be writeable by the Library service and readable by the Nu Designer service.

#### Kafka topics

You list all topics which are used by the scenario in the `{scenario-name}/setup/kafka/topics.txt` file. All the topics will be created before deploying the scenario. The format of the file looks like the following:

```text
CustomerEvents
OfferProposalsBasedOnCustomerEvents

```

#### JSON Schemas for Kafka topics

For each defined topic you should provide a JSON schema. You add schemas in the `{scenario-name}/setup/schema-registry` folder. Name format 
for a schema file is `{topic-01-name}.schema.json`. It means that the schema will be added for a topic `{topic-01-name}. 

#### Flink tables

To create Flink tables you can use Flink SQL scripts places in the `{scenario-name}/setup/flink` folder. Each script has to have the `*.sql` extension.

<details>
  <summary>Example</summary>
  
  `tables.sql` file:
  ```sql
    CREATE CATALOG nessie_catalog WITH (
      'type'='iceberg',
      'catalog-impl'='org.apache.iceberg.nessie.NessieCatalog',
      'io-impl'='org.apache.iceberg.aws.s3.S3FileIO',
      'uri'='http://nessie:19120/api/v2',
      'warehouse' = 's3://warehouse',
      's3.endpoint'='http://minio:9000'
    );

    USE CATALOG nessie_catalog;

    CREATE DATABASE IF NOT EXISTS `default`;

    CREATE TABLE IF NOT EXISTS orders
    (
      order_timestamp TIMESTAMP,
      order_date      STRING,
      id              BIGINT,
      customer_id     BIGINT
    ) PARTITIONED BY (order_date);

    CREATE TABLE IF NOT EXISTS products
    (
      id            BIGINT,
      name          STRING,
      product_category TINYINT,
      current_prise DECIMAL(15, 2)
    );

    CREATE TABLE IF NOT EXISTS order_items
    (
      id         BIGINT,
      order_id   BIGINT,
      product_id BIGINT,
      quantity   BIGINT,
      unit_prise DECIMAL(15, 2)
    );

    CREATE TABLE IF NOT EXISTS product_orders_report
    (
      report_id            BIGINT,
      report_timestamp     TIMESTAMP,
      report_year_month    STRING,
      product_id           BIGINT,
      product_quantity_sum BIGINT
    ) PARTITIONED BY (report_year_month);
  ```
</details>


### Scenario external services mocks

Some scenarios can use components that call external services like database or Open API HTTP service. In this case you need to provide
mocks which will pretend to be real services.

#### DB mocks

DB mocks should be added to the `{scenario-name}/mocks/db` folder. The mocks has a form of PostgreSQL DDL scripts. Name of the script will 
be used as a schema in the database (all scripts will be run in context of the same PostgreSQL db instance). 

Assuming that your db mock is `{scenario-name}/mocks/db/example01.sql`, you should be able to refer to it like that:

```hocon
# Nu Designer configuration
db {
  driverClassName: "org.postgresql.Driver"
  url: "jdbc:postgresql://"${EXAMPLE_SCENARIOS_LIBRARY_SERVICE_NAME}":5432/mocks"
  username: "mocks"
  password: "mocks_pass"
  schema: "example01"
}
```

See the `scenario-examples-library/rtm-client-near-pos` example. 

#### OpenAPI mocks

OpenAPI mocks should be added in the `{scenario-name}/mocks/http-service` folder. Mock for single API contains the service OpenAPI definition (placed in the `{scenario-name}/mocks/http-service/{service-name}/__files/{service-name}/openapi` folder) and Wiremock's mappings (placed in the 
`{scenario-name}/mocks/http-service/{service-name}/mappings/{service-name}` folder). Sometimes in the mappings you can refer to static files. These files can be added to the `{scenario-name}/mocks/http-service/{service-name}/__files/{service-name}/responses` folder.

Assuming that your OpenAPI mock is `{scenario-name}/mocks/http-service/{service-name}`, you should be able to refer to it like that:

```hocon
# Nu Designer configuration

        # OpenAPI enricher
        "customerProfileOffers" {
          providerType: "openAPI"
          url: "http://"${EXAMPLE_SCENARIOS_LIBRARY_SERVICE_NAME}":8080/__admin/files/customer-api/openapi/CustomerApi.yaml"
          rootUrl: "http://"${EXAMPLE_SCENARIOS_LIBRARY_SERVICE_NAME}":8080/"
          namePattern: "get.*"
          allowedMethods: ["GET"]
        }
```

See the `scenario-examples-library/offer-customer-proposal-based-on-activity-event` example.

Check out the following resources to see how to create Wiremock mappings:
https://github.com/wiremock/wiremock-faker-extension/blob/main/docs/reference.md

https://docs.wiremock.io/response-templating/basics/

https://docs.wiremock.io/response-templating/dates-and-times/

### Example data for scenario showcase

To see how scenario works, you need to provide some data that will be interpreted/processed by the scenario. You can provide the data for Streaming and Request-Response scenarios in static and dynamic form. The dynamic data will be generated continuously since the Library container started. 

#### Streaming scenario

##### Dynamic data

Dynamic Kafka messages are provided by generator scripts. The scripts should be placed in the `{scenario-name}/data/kafka/generated` folder.
The topic name, the data generated by the script will be sent to, is taken from the name of the script file (e.g. script `transactions.sh` generates messages that will be sent to `Transactions` topic). A script should echo a string (e.g. stringified JSON). 

> 💡 You can use `/app/utils/lib.sh` script to import helpers that contains set of functions that will help you to create the data. Please,
> don't hesitate to add more util functions to it where you need them but there is no any. 

<details>
  <summary>Example</summary>
  
  `Transactions.sh` script:
  ```bash
  #!/bin/bash -e
  source /app/utils/lib.sh

  ID=$((1 + $(random_4digit_number) % 5))
  AMOUNT=$((1 + $(random_4digit_number) % 30))
  TIME=$(($(now) - $(random_4digit_number) % 20))

  echo "{ \"clientId\": \"Client$ID\", \"amount\": $AMOUNT, \"eventDate\": $TIME}"
  ```
</details>

##### Static data

Static Kafka messages are provided with text file placed in the `{scenario-name}/data/kafka/static` folder. The topic name is taken 
from the name of the file (e.g. `transactions.txt` file contains messages that will be sent to `Transactions` topic). The file contains 
message per line. 

<details>
  <summary>Example</summary>
  
  `Transactions.txt` file:
  ```text
  # Example messages below (message per line) 
  { "clientId": "Client1", "amount": 100, "eventDate": 1720166429}"
  { "clientId": "Client2", "amount": 1000, "eventDate": 1720166429}"
  ```
</details>

#### Request-Response scenario

##### Dynamic data

Dynamic HTTP requests are provided by generator scripts (the scripts basically provide request's body payload, because at the moment 
we support only POST requests). The scripts should be placed in the `{scenario-name}/data/http/generated` folder. The URL, 
the request generated by the script will be sent to, consists of a static path and a dynamic part taken from the name of the script file 
(e.g. script `loan.sh` generates requests that will be sent to `http://$NU_REQUEST_RESPONSE_OPEN_API_SERVICE_ADDRESS/scenario/loan`).
A script should echo a string (e.g. stringified JSON). 

> 💡 You can use `/app/utils/lib.sh` script to import helpers that contains set of functions that will help you to create the data. Please,
> don't hesitate to add more util functions to it where you need them but there is no any. 

<details>
  <summary>Example</summary>
  
  `Loan.sh` script:
  ```bash
  #!/bin/bash -e
  source /app/utils/lib.sh

  ID="$(random_4digit_number)"
  AMOUNT="$(random_4digit_number)"
  REQUEST_TYPE="$(pick_randomly "loan" "mortgage" "insurance")"
  CITY="$(pick_randomly "Warszawa" "Berlin" "Gdańsk" "Kraków", "Poznań", "Praga")"

  echo "{\"customerId\": \"$ID\", \"requestedAmount\": $AMOUNT, \"requestType\": \"$REQUEST_TYPE\", \"location\": { \"city\": \"$CITY\", \"street\": \"\" }}"
  ```
</details>

##### Static data

Static HTTP requests (payloads) are provided with text file placed in the `{scenario-name}/data/http/static` folder. The URL 
consists of a static path and a dynamic part taken from the name of the file  (e.g. `loan.txt` contains requests that will be 
sent to `http://$NU_REQUEST_RESPONSE_OPEN_API_SERVICE_ADDRESS/scenario/loan`). The file contains request payload per line. 

<details>
  <summary>Example</summary>
  
  `Loan.txt` file:
  ```text
  # Example Request-Response OpenAPI service requests (request payload per line)
  {"customerId": "anon", "requestedAmount": 1555, "requestType": "mortgage", "location": { "city": "Warszawa", "street": "Marszałkowska" }}
  {"customerId": "anon", "requestedAmount": 86, "requestType": "loan", "location": { "city": "Lublin", "street": "Głęboka" }}
  {"customerId": "1", "requestedAmount": 1000, "requestType": "loan", "location": { "city": "Warszawa", "street": "Marszałkowska" }}
  {"customerId": "1", "requestedAmount": 500, "requestType": "savings", "location": { "city": "London", "street": "Kensington" }}
  {"customerId": "4", "requestedAmount": 2000, "requestType": "mortgage", "location": { "city": "Lublin", "street": "Lipowa" }}
  {"customerId": "3", "requestedAmount": 2000, "requestType": "loan", "location": { "city": "Lublin", "street": "Głęboka" }}
  ```
</details>

#### Batch scenario

##### Dynamic data

Not supported yet.

##### Static data

We can upload static data to Flink tables by providing Flink SQL DML scripts in the `{scenario-name}/data/flink/static` folder. Each 
file with scripts has the `*.sql` extension.

<details>
  <summary>Example</summary>
  
  `inserts.sql` file:
  ```sql
      CREATE CATALOG nessie_catalog WITH (
      'type'='iceberg',
      'catalog-impl'='org.apache.iceberg.nessie.NessieCatalog',
      'io-impl'='org.apache.iceberg.aws.s3.S3FileIO',
      'uri'='http://nessie:19120/api/v2',
      'warehouse' = 's3://warehouse',
      's3.endpoint'='http://minio:9000'
    );

    USE CATALOG nessie_catalog;

    INSERT INTO products (id, name, product_category, current_prise)
    VALUES (1, 'Basic Widget', 0, 9.99),
          (2, 'Premium Widget', 1, 19.99),
          (3, 'Basic Gadget', 0, 14.99),
          (4, 'Premium Gadget', 1, 29.99),
          (5, 'Basic Tool', 0, 24.99),
          (6, 'Premium Tool', 1, 49.99);

    INSERT INTO orders (order_timestamp, order_date, id, customer_id)
    VALUES (TO_TIMESTAMP('2024-09-12 14:25:30'), '2024-09-12', 1001, 2001),
          (TO_TIMESTAMP('2024-09-12 15:10:45'), '2024-09-12', 1002, 2002),
          (TO_TIMESTAMP('2024-09-13 09:20:10'), '2024-09-13', 1003, 2003),
          (TO_TIMESTAMP('2024-09-13 10:30:05'), '2024-09-13', 1004, 2004),
          (TO_TIMESTAMP('2024-09-13 12:15:22'), '2024-09-13', 1005, 2001),
          (TO_TIMESTAMP('2024-09-14 14:50:00'), '2024-09-14', 1006, 2002);

    INSERT INTO order_items (id, order_id, product_id, quantity, unit_prise)
    VALUES (10001, 1001, 1, 2, 9.99), -- Order 1001 contains 2 Basic Widgets
          (10002, 1001, 3, 1, 14.99), -- Order 1001 also contains 1 Basic Gadget
          (10003, 1002, 2, 1, 19.99), -- Order 1002 contains 1 Premium Widget
          (10004, 1003, 3, 3, 14.99), -- Order 1003 contains 3 Basic Gadgets
          (10005, 1004, 4, 1, 29.99), -- Order 1004 contains 1 Premium Gadget
          (10006, 1005, 5, 4, 24.99), -- Order 1005 contains 4 Basic Tools
          (10007, 1006, 6, 2, 49.99);
  ```
</details>
