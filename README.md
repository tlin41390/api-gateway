# API Gateway

The API gateway is the single entry point for the TaskFlow system. Clients send every request to `http://localhost:8082`, and the gateway forwards it to the right service based on the path. It finds the services through the Eureka discovery server, so it doesn't need to know their ports.

The gateway has no code of its own apart from the application class. All routing is set in `application.properties`.

## Tech stack

- Java 25
- Spring Boot 4.1
- Spring Cloud 2025.1 (Gateway Server WebMVC, Netflix Eureka client, LoadBalancer)
- Maven (the wrapper is included, so you don't need to install Maven)

## How the services fit together

| Service                | Port          | What it does                                                        |
|------------------------|---------------|---------------------------------------------------------------------|
| `discovery-server`     | `8761`        | Eureka registry. The other services register here.                  |
| `taskflow`             | `8080`, `61616` | Task REST API. Also runs the embedded Artemis message broker on 61616. |
| `notification-service` | `8081`        | Stores a notification each time a task is created.                  |
| `api-gateway`          | `8082`        | This project. Routes requests to the two services above.            |

```
               ┌──────────────────┐
               │ discovery-server │  :8761
               └──────────────────┘
                 ▲      ▲      ▲
       looks up  │      │      │  register
                 │      │      │
client ──► api-gateway  │      │
            :8082 │     │      │
                  ├─ /tasks/** ──────► taskflow  :8080
                  │                      │  TaskCreatedEvent (JMS, :61616)
                  │                      ▼
                  └─ /notifications/** ► notification-service  :8081
```

When you create a task through the gateway:

1. The gateway matches `/tasks/**`, asks Eureka for a `TASKFLOW` instance and forwards the request unchanged, including the `Authorization` header.
2. TaskFlow checks the login, saves the task and returns it.
3. TaskFlow publishes a `TaskCreatedEvent` to `task.created.queue` on its embedded Artemis broker.
4. The notification service receives the event and stores `"Task created: <title>"`.

## Getting started

### Prerequisites

- JDK 25 or newer
- The other three projects checked out next to this one

### Start order

Start the services in this order, each in its own terminal:

1. `discovery-server`
2. `taskflow`. It must start before the notification service, because the notification service connects to the broker that runs inside TaskFlow.
3. `notification-service`
4. `api-gateway`

In each project folder, run:

```bash
# macOS / Linux
./mvnw spring-boot:run

# Windows
mvnw.cmd spring-boot:run
```

Check the Eureka dashboard at `http://localhost:8761`. `TASKFLOW` and `NOTIFICATION-SERVICE` should both be listed before you send requests through the gateway. Registration can take up to 30 seconds after a service starts.

### Run the tests

```bash
./mvnw test        # or mvnw.cmd test on Windows
```

`ApiGatewayApplicationTests` checks that the application starts.

## Routes

| Path                | Forwarded to                  | Service ID in Eureka   |
|---------------------|-------------------------------|------------------------|
| `/tasks/**`         | `lb://TASKFLOW`               | `TASKFLOW`             |
| `/notifications/**` | `lb://NOTIFICATION-SERVICE`   | `NOTIFICATION-SERVICE` |

Paths are forwarded as they are, so `GET :8082/tasks/1` becomes `GET /tasks/1` on TaskFlow. Any other path returns `404 Not Found` from the gateway itself.

Routes are set in `application.properties` under `spring.cloud.gateway.server.webmvc.routes`. The older `spring.cloud.gateway.routes` prefix is ignored by this version, and the gateway then returns `404` for every path.

## Authentication

The gateway doesn't check logins. It passes the `Authorization` header through, and TaskFlow checks it. TaskFlow uses HTTP Basic with these accounts:

| Username | Password    | Role    |
|----------|-------------|---------|
| `admin`  | `adminpass` | `ADMIN` |
| `alice`  | `alicepass` | `USER`  |

Reading tasks needs no login, creating and updating need any login, and deleting needs `admin`. The notification service has no login at all. See the TaskFlow README for the full rules.

## Example with curl

```bash
# Create a task (-i shows the status line, so a failed login isn't silent)
curl -i -u alice:alicepass -X POST http://localhost:8082/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Routed through the gateway"}'

# List tasks
curl http://localhost:8082/tasks

# List notifications
curl http://localhost:8082/notifications
```

On Windows PowerShell, use `curl.exe`, put each command on one line, and escape the inner quotes, for example `-d '{\"title\": \"Routed through the gateway\"}'`.

## Troubleshooting

| What you see | Likely cause |
|---|---|
| `404` with `"path":"/tasks"` for every request | The routes weren't loaded. Check that they use the `spring.cloud.gateway.server.webmvc.routes` prefix. |
| `503 Service Unavailable` | The target service isn't registered in Eureka yet. Check `http://localhost:8761` and wait for it to appear. |
| `401 Unauthorized` with an empty body on `POST /tasks` | Wrong or missing login. Use `alice:alicepass` or `admin:adminpass`. |
| Tasks are created but `/notifications` stays empty | The notification service started before TaskFlow, or can't reach `localhost:61616`. Restart it after TaskFlow is up. |
| Data is gone | Both services keep data in memory. Restarting either one clears it. |

## Project structure

```
src/main/java/com/example/api_gateway/
└── ApiGatewayApplication.java      # Application entry point
src/main/resources/
└── application.properties          # Port, Eureka address and routes
```
