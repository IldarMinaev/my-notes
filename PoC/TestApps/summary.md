The main objective is to find an application that most closely meets the following criteria:

- Easy deployment in Kubernetes (K8s)
- Minimal complex business logic
- Simplicity in extending and adding new logic to the application
- Multilingual support (Mandatory: Java, Go; Optional: Python)
- Use of two Java frameworks (Spring Boot, Quarkus)
- Use of message queues (Kafka, RabbitMQ)
- Use of databases (PostgreSQL, Cassandra, MongoDB)
- Use of indexing engines (OpenSearch)
- Interaction between microservices via REST and gRPC

The next table represents the results of test application research:

| **App name** | [ftgo](https://github.com/microservices-patterns/ftgo-application) | [save](https://github.com/saveourtool/save-cloud) | [Google ms-demo](https://github.com/GoogleCloudPlatform/microservices-demo) | [cool-tools](https://github.com/Zhykos/cool-tools) | [magda](https://github.com/magda-io/magda) | [robot-shop](https://github.com/instana/robot-shop) |
|---|---|---|---|---|---|---|
| Easy k8s deploy |  | [Helm](https://github.com/saveourtool/save-cloud/tree/f651465b403a89bf34cde9af973d318cfd11cffb/save-cloud-charts/save-cloud) | `scaffold run` | docker-compose only |  |  |
| Complex busines logic |  |  |  |  |  |  |
| Easy to extend |  |  |  |  |  |  |
| Java Quarkus |  | ➖ | ➖ |  |  |  |
| Java Springboot |  | ➕ | ➖ |  |  |  |
| Golang |  | ➖ | ➕ |  |  |  |
| Python |  | ➖ | ➕ |  |  |  |
| Kafka |  | ➕ | ➖ |  |  |  |
| RabbitMQ | | ➖  | ➖ |  |  |  |
| PostgreSQL | | ➖ | ➖ |  |  |  |
| MySQL | | ➕ | ➖ |  |  |  |
| Neo4j | | ➕ | ➖ |  |  |  |
| Redis | | ➖ | ➕ |  |  |  |
| Cassandra | | ➖ | ➖ |  |  |  |
| MongoDB |  | ➖ | ➖ |  |  |  |
| OpenSearch |  | ➖ | ➖ |  |  |  |
| REST |  | ➕ | ❓ |  |  |  |
| gRPC |  | ➖ | ➕ |  |  |  |
