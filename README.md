# Spring Cloud Config - Configuration Repository

> **GitHub:** https://github.com/spring-config-demo/config-file

This repository serves as the **Git-backed configuration store** for a Spring Cloud Config Server. It holds externalized configuration files for multiple services across different environments.

## Repository Structure

```
config-file/
├── global/                         # Shared configuration for all services
│   ├── application.properties      # Default (no active profile)
│   ├── application-dev.properties  # Development profile
│   ├── application-qa.properties   # QA/Testing profile
│   └── application-prod.properties # Production profile
│
└── orderService/                   # Configuration specific to Order Service
    ├── order-service.properties      # Default (no active profile)
    ├── order-service-dev.properties  # Development profile
    ├── order-service-qa.properties   # QA/Testing profile
    └── order-service-prod.properties # Production profile
```

## How It Works

Spring Cloud Config Server reads configuration from this Git repository and serves it to client microservices at startup via HTTP endpoints.

### Property Resolution Order (highest to lowest priority)

1. Service-specific + active profile — e.g., `order-service-prod.properties`
2. Service-specific default — e.g., `order-service.properties`
3. Global + active profile — e.g., `application-prod.properties`
4. Global default — `application.properties`

### Config Server Endpoint Pattern

```
GET /{application}/{profile}
GET /{application}/{profile}/{label}
```

| Placeholder   | Example value         |
|---------------|-----------------------|
| `application` | `order-service`       |
| `profile`     | `dev`, `qa`, `prod`   |
| `label`       | `main` (git branch)   |

## Environments

| Profile | Purpose                    |
|---------|----------------------------|
| (none)  | Default / fallback values  |
| `dev`   | Local development          |
| `qa`    | QA / integration testing   |
| `prod`  | Production                 |

## Configuration Properties

### `custom.message`

A sample property used to verify that the correct profile and service configuration is loaded.

| File                             | Value                           |
|----------------------------------|---------------------------------|
| `global/application.properties`              | `Hello from global git`         |
| `global/application-dev.properties`          | `Hello from global dev`         |
| `global/application-qa.properties`           | `Hello from global qa`          |
| `global/application-prod.properties`         | `Hello from global prod`        |
| `orderService/order-service.properties`      | `Hello from orderService git`   |
| `orderService/order-service-dev.properties`  | `Hello from orderService dev - updated WITH refresh scope` |
| `orderService/order-service-qa.properties`   | `Hello from orderService qa`    |
| `orderService/order-service-prod.properties` | `Hello from orderService prod`  |

## Live Config Refresh with `@RefreshScope`

Spring Cloud Config supports refreshing a running client's properties **without a restart** using Spring Actuator.

### How it works

1. Push a config change to this repo.
2. Call the client's refresh endpoint:
   ```bash
   POST http://<client-host>:<port>/actuator/refresh
   ```
3. Any bean annotated with `@RefreshScope` on the client reloads its injected properties.

### Client-side requirements

```java
// In the client microservice
@RefreshScope
@RestController
public class MessageController {
    @Value("${custom.message}")
    private String message;
    // ...
}
```

The `spring-boot-actuator` dependency and the following config must be present in the client:

```properties
# application.properties on the client
management.endpoints.web.exposure.include=refresh
```

> Without `@RefreshScope`, the bean retains the value it had at startup even after `/actuator/refresh` is called. This was demonstrated in commits `ec38660` (without) and `9e87130` (with refresh scope).



## Adding a New Service

1. Create a new folder named after the service (e.g., `paymentService/`).
2. Add `<service-name>.properties` for defaults.
3. Add `<service-name>-<profile>.properties` for each environment.
4. Commit and push — the Config Server picks up changes automatically on the next refresh.

## Adding a New Property

Edit the relevant `.properties` file and commit. Clients pick up the change on the next `/actuator/refresh` call or restart.
