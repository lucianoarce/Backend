# FiinSoft Core V2

FiinSoft Core solution (v2).

## Technologies used

- [.NET 6.0](https://dot.net)
  - [Minimal API](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis?view=aspnetcore-6.0)
  - [Entity Framework Core](https://docs.microsoft.com/en-us/ef/core)
- [MediatR](https://github.com/jbogard/MediatR)
- [Mapster](https://github.com/MapsterMapper/Mapster)
- [FluentValidation](https://fluentvalidation.net)
- [xUnit](https://xunit.net)
- [Docker](https://www.docker.com)
- [Microsoft SQL Server](https://www.microsoft.com/en-us/sql-server)
- Observability:
  - [Open Telemetry Collector](https://opentelemetry.io/docs/collector)MapsterMapster
  - [Prometheus](https://grafana.com/oss/prometheus)
  - [Grafana Tempo](https://grafana.com/oss/tempo)
  - [Grafana Loki](https://grafana.com/oss/loki)
  - [Grafana](https://grafana.com)

## Implementation description

The implementation follows the principles described on [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html), the layers are isolated in projects:

- **Domain** _layer_: The heart of the application, and responsible of the core models (which must be persistence ignorant), must abstract the expected behavior of the application, so it only declares interfaces with permitted actions. So it is splitted in two parts:
  - `FiinSoft.*.Domain.Entities`: The database/services models (entities, value objects, aggregates, enumerations), must be declared here. Only can reference `FiinSoft.Abstractions` project.
  - `FiinSoft.*.Domain.Ports`: The ports (interfaces: repository, unit of work, third party services) of the application, must be declared here. Only can reference `FiinSoft.Abstractions` and `FiinSoft.*.Domain.Entities` projects.
- **Application** _layer_: Implements the use cases of the business. Provides mapping from domain to DTOs and vice versa. Validation goes into this layer. And implements [CQRS](https://docs.microsoft.com/en-us/azure/architecture/patterns/cqrs)/[Mediator pattern](https://sourcemaking.com/design_patterns/mediator) to handle all application requests. Also additional behavior like logging, caching, monitoring can be performed here.
  > ⚠️  The _Application Layer_ **ONLY** references the _Domain Layer_, it knows nothing of databases, third party services, etc., it only uses the abstractions (interfaces) declared in the `FiinSoft.*.Domain.Ports` project.
  - `FiinSoft.*.Application`: Mapping is implemented by [MapsterMapsterMapste library](https://github.com/MapsterMapper/Mapster). Validation is implemented by [FluentValidation library](https://fluentvalidation.net). CQRS/Mediator is implemented by [MediatR library](https://github.com/jbogard/MediatR). Only can reference `FiinSoft.Abstractions`, `FiinSoft.*.Domain.Entities` and `FiinSoft.*.Domain.Ports` projects.
- **Infrastructure** _layer_: This will provide functionality to access external systems. These will be hooked up by the IoC container, usually in the _Presentation Layer_.
  > ⚠️  The _Infrastructure Layer_ **ONLY** references the _Domain Layer_, as it implements the behaviors described there.
  - `FiinSoft.*.Infrastructure.Adapters`: Implements interfaces from the `FiinSoft.*.Domain.Ports` project, additonally can reference `FiinSoft.Abstractions`, `FiinSoft.*.Domain.Entities` and `FiinSoft.Base.Infrastructure` projects.
  - `FiinSoft.*.Infrastructure.Data`: Implements the database configuration ([Entity Framework Core](https://docs.microsoft.com/en-us/ef/core)), so maps entities to persistance model using [Code-First approach](https://entityframework.net/ef-code-first). It can reference only `FiinSoft.Abstractions`, `FiinSoft.*.Domain.Entities` and `FiinSoft.Base.Infrastructure` projects.
- **Presentation** _layer_: Is the entry point to the system from the user’s point of view. Its primary concerns are routing requests to the _Application Layer_ and registering all dependencies in the IoC container.
  > ⚠️  The Presentation Layer_ references all the other layers.
  - `*.WebApi`: As the solution is currently implemented as _API First approach_, the entry point is a REST API implemented in this project. So this references all `FiinSoft.Abstractions`, `FiinSoft.*.Domain.Entities`, `FiinSoft.*.Domain.Ports`, `FiinSoft.*.Application`, `FiinSoft.*.Infrastructure.Adapters`, `FiinSoft.*.Infrastructure.Data` and `FiinSoft.Base.WebApi` projects.

![Clean Architecture - FiinSoft Implementation](./docs/diagrams/clean-architecture.drawio.svg)

### Interactions

- FiinSoft services:

  - Security: Used to generate access token (JWT) to invoke all other FiinSoft services.
  - Tenants: Used to manage tenants and their options (configuration), e.g. tenant identification, tenant name, connection string(s), etc.
  - Core: Used to manage financial core services: products, accounts, transactions, etc.

  ![Interactions - FiinSoft services](./docs/diagrams/interactions-fiinsoft.drawio.svg)

- Third party services:

  - Microsoft SQL Server (**mandatory**): Database
  - Open Telemetry Collector (_optional_): Traces & Metrics collector
  - Prometheus (_optional_): Metrics datasource
  - Grafana Tempo (_optional_): Traces datasource
  - Grafana Loki (_optional_): Logs collector and datasource
  - Grafana (_optional_): Observability (Traces, Metrics & Logs) visualization

  ![Interactions - Thrid party services](./docs/diagrams/interactions-3rd-party.drawio.svg)

## Running locally

- Prerrequisites
  - [Git](https://git-scm.com/downloads)
  - [.NET 6.0 SDK](https://dotnet.microsoft.com/en-us/download/dotnet/6.0)
  - [SQL Server](https://www.microsoft.com/en-us/sql-server/sql-server-downloads) ([docker image](https://hub.docker.com/_/microsoft-mssql-server) also works)

1. Clone current repository:

    ```bash
    git clone ssh://<your-user>@source.developers.google.com:2022/p/fiinsoft-desarrollo/r/fiinsoft-core-V2
    ```

2. Move to cloned directory:

    ```bash
    cd fiinsoft-core-V2
    ```

    > ⚠️ Remember to edit `ConnectionStrings:Default` to connect to SQL Server:
    >
    > - `src/Security/FiinSoft.Security.WebApi/appsettings.json`
    > - `src/Tenants/FiinSoft.Tenants.WebApi/appsettings.json`

3. Restore dependencies:

    ```bash
    # Security application
    dotnet restore FiinSoft.Security.sln

    # Tenants application
    dotnet restore FiinSoft.Tenants.sln

    # Core application
    dotnet restore FiinSoft.Core.sln
    ```

4. Build:

    ```bash
    # Security application
    dotnet build FiinSoft.Security.sln

    # Tenants application
    dotnet build FiinSoft.Tenants.sln

    # Core application
    dotnet build FiinSoft.Core.sln
    ```

5. Run APIs:

    ```bash
    # Security API
    dotnet run --project ./src/Security/FiinSoft.Security.WebApi/FiinSoft.Security.WebApi.csproj

    # Tenants API
    dotnet run --project ./src/Tenants/FiinSoft.Tenants.WebApi/FiinSoft.Tenants.WebApi.csproj

    # Core API
    dotnet run --project ./src/Core/FiinSoft.Core.WebApi/FiinSoft.Core.WebApi.csproj
    ```

6. Execute REST requests, the APIs will be available at:

    - Security API: `http://localhost:6001` or `https://localhost:7001`. To display swagger just go to route [/swagger](http://localhost:6001/swagger).
    - Tenants API: `http://localhost:6002` or `https://localhost:7002`. To display swagger just go to route [/swagger](http://localhost:6002/swagger).
    - Core API: or `http://localhost:6003` or `https://localhost:7003`. To display swagger just go to route [/swagger](http://localhost:6003/swagger).

## Docker setup

To have a full environment running on Docker, including observability tools (metrics, tracing, logs) and [Grafana](https://grafana.com)). Execute following steps in cloned root directory (default `fiinsoft-core-V2`):

  > ⚠️ To configure observability services is required to copy `/observability/*.yml` files into `$HOME/.observability/` directory:

1. Create a Network

    ```bash
    docker network create fiinsoft-net
    ```

    > 💡 When a container is already running, attach network:
    >
    > ```bash
    > docker network connect fiinsoft-net <container-name>
    > ```

2. Microsoft SQL Server

    ```bash
    # Download Docker image
    docker pull mcr.microsoft.com/mssql/server

    # Generate volume to persist data
    docker volume create fiinsoft-mssql-data

    # Execute Docker container with volume and network attached
    docker run \
      --name fiinsoft-mssql \
      -p 1433:1433 \
      -e ACCEPT_EULA=Y \
      -e SA_PASSWORD="<your-super-secure-password>" \
      -v fiinsoft-mssql-data:/var/opt/mssql \
      --net fiinsoft-net \
      -d mcr.microsoft.com/mssql/server
    ```

    > 🔗 Microsoft SQL Server will be available on `localhost:1433`, accessible for other containers as `fiinsoft-mssql`.

3. Grafana Loki

    ```bash
    # Download Docker image
    docker pull grafana/loki

    # Execute Docker container
    docker run \
      --name fiinsoft-loki \
      -p 3100:3100 \
      -v ~/.observability/loki-config.yml:/etc/config/loki-config.yml:ro \
      --net fiinsoft-net \
      -d grafana/loki
    ```

    > 🔗 Grafana Loki will be available on [http://localhost:3100](http://localhost:3100), accessible for other containers as `http://fiinsoft-loki:3100`.

4. Grafana Tempo

    ```bash
    # Download Docker image
    docker pull grafana/tempo

    # Generate volume to persist data
    docker volume create fiinsoft-tempo-data

    # Execute Docker container
    docker run \
      --name fiinsoft-tempo \
      -p 55681:55681 \
      -p 55682:55682 \
      -p 3200:3200 \
      -p 9095:9095 \
      -v fiinsoft-tempo-data:/tmp/tempo \
      -v ~/.observability/tempo.yml:/etc/tempo.yml:ro \
      --net fiinsoft-net \
      -d grafana/tempo \
      -config.file=/etc/tempo.yml
    ```

    > 🔗 Grafana Tempo will be available on `localhost:55681`, accessible for other containers as `fiinsoft-tempo:55681`.

5. OpenTelemetry Collector

    ```bash
    # Download Docker image
    docker pull otel/opentelemetry-collector

    # Execute Docker container
    docker run \
      --name fiinsoft-otel \
      -p 8888:8888 \
      -p 8889:8889 \
      -p 4317:4317 \
      -p 4318:4318 \
      -v ~/.observability/otel-collector.yml:/etc/otel-collector.yml:ro \
      --net fiinsoft-net \
      -d otel/opentelemetry-collector \
      --config /etc/otel-collector.yml;
    ```

    > 🔗 OpenTelemetry Collector will be available on `localhost:4317`, [http://localhost:8888/metrics](http://localhost:8888/metrics), [http://localhost:8889/metrics](http://localhost:8889/metrics), accessible for other containers as `fiinsoft-otel:4317`, `http://fiinsoft-otel:8888/metrics`, `http://fiinsoft-otel:8889/metrics`.

6. Prometheus

    ```bash
    # Download Docker image
    docker pull prom/prometheus

    # Generate volume to persist data
    docker volume create fiinsoft-prometheus-data

    # Execute Docker container
    docker run \
      --name fiinsoft-prometheus \
      -p 9090:9090 \
      -v fiinsoft-prometheus-data:/prometheus \
      -v ~/.observability/prometheus.yml:/etc/prometheus/prometheus.yml:ro \
      --net fiinsoft-net \
      -d prom/prometheus
    ```

    > 🔗 Prometheus will be available at: [http://localhost:9090](http://localhost:9090), accessible for other containers as `http://fiinsoft-prometheus:9090`.

7. Grafana

    ```bash
    # Download Docker image
    docker pull grafana/grafana

    # Execute Docker container
    docker run \
      --name fiinsoft-grafana \
      -p 3000:3000 \
      -e GF_AUTH_ANONYMOUS_ENABLED=true \
      -e GF_AUTH_ANONYMOUS_ORG_ROLE=Admin \
      -e GF_AUTH_DISABLE_LOGIN_FORM=true \
      -v ~/.observability/grafana-datasources.yml:/etc/grafana/provisioning/datasources/datasources.yaml:ro \
      --net fiinsoft-net \
      -d grafana/grafana
    ```

    > 🔗 Grafana Viewer will be available at: [http://localhost:3000](http://localhost:3000), accessible for other containers as `http://fiinsoft-grafana:3000`.

8. Security API

    ```bash
    # Build Docker image
    docker build -t fiinsoft-security-api -f ./src/Security/Dockerfile .

    # Execute Docker container with network attached
    docker run \
      --rm -it \
      --name fiinsoft-security-api \
      -p 8001:80 \
      -e LEGACY_DB_SERVER="<legacy-security-database-server>" \
      -e LEGACY_DB_NAME="<legacy-security-database-name>" \
      -e LEGACY_DB_USR="<legacy-security-database-user>" \
      -e LEGACY_DB_PWD="<legacy-security-database-password>" \
      -e DB_SERVER="<security-database-server>" \
      -e DB_NAME="<security-database-name>" \
      -e DB_USR="<security-database-user>" \
      -e DB_PWD="<security-database-password>" \
      -e Jwt__Issuer="<jwt-issuer-name>" \
      -e Jwt__Audience="<jwt-audience-name>" \
      -e Jwt__Expiration="<jwt-expiration-in-seconds>" \
      -e Jwt__RefreshExpiration="<jwt-refresh-expiration-in-seconds>" \
      -e Jwt__Key="<jwt-signing-key>" \
      --net fiinsoft-net \
      -d fiinsoft-security-api \
      --environment=Docker
    ```

    > 🔗 The API will be available at: [http://localhost:8001](http://localhost:8001/swagger), accessible for other containers as `http://fiinsoft-security-api`.

9. Tenants API

    ```bash
    # Build Docker image
    docker build -t fiinsoft-tenants-api -f ./src/Tenants/Dockerfile .

    # Execute Docker container with network attached
    docker run \
      --rm -it \
      --name fiinsoft-tenants-api \
      -p 8002:80 \
      -e DB_SERVER="<tenants-database-server>" \
      -e DB_NAME="<tenants-database-name>" \
      -e DB_USR="<tenants-database-user>" \
      -e DB_PWD="<tenants-database-password>" \
      -e Jwt__Issuer="<jwt-issuer-name>" \
      -e Jwt__Audience="<jwt-audience-name>" \
      -e Jwt__Key="<jwt-signing-key>" \
      --net fiinsoft-net \
      -d fiinsoft-tenants-api \
      --environment=Docker
    ```

    > 🔗 The API will be available at: [http://localhost:8002](http://localhost:8002/swagger), accessible for other containers as `http://fiinsoft-tenants-api`.

10. Core API

    ```bash
    # Build Docker image
    docker build -t fiinsoft-core-api -f ./src/Core/Dockerfile .

    # Execute Docker container with network attached
    docker run \
      --rm -it \
      --name fiinsoft-core-api \
      -p 8003:80 \
      -e Jwt__Issuer="<jwt-issuer-name>" \
      -e Jwt__Audience="<jwt-audience-name>" \
      -e Jwt__Key="<jwt-signing-key>" \
      --net fiinsoft-net \
      -d fiinsoft-core-api \
      --environment=Docker
    ```

    > 🔗 The API will be available at: [http://localhost:8003](http://localhost:8003/swagger), accessible for other containers as `http://fiinsoft-core-api`.

> ℹ️  To cleanup docker objects created:
>
> - Containers: `docker rm -f $(docker container ls -f name=fiinsoft- --all -q)`
> - Images (only FiinSoft): `docker rmi -f $(docker images -f "reference=fiinsoft-*" -q)`
> - Volumes: `docker volume rm $(docker volume ls -f name=fiinsoft- --format "{{.Name}}")`

### `docker-compose`

To setup services (Third-party and FiinSoft), the `docker-compose` configuration is generated. So, the services will be available as follows:

- Third-party:
  - Microsoft SQL Server on `localhost:1433`, accessible for other docker containers as `mssql`.
  - Grafana Loki on [http://localhost:3100](http://localhost:3100), accessible for other docker containers as `http://loki:3100`.
  - Grafana Tempo on `localhost:55681`, accessible for other docker containers as `tempo:55681`.
  - OpenTelemetry Collector on `localhost:4317`, [http://localhost:8888/metrics](http://localhost:8888/metrics), [http://localhost:8889/metrics](http://localhost:8889/metrics), accessible for other docker containers as `otel:4317`, `http://otel:8888/metrics`, `http://otel:8889/metrics`.
  - Prometheus on [http://localhost:9090](http://localhost:9090), accessible for other docker containers as `http://prometheus:9090`.
  - Grafana on [http://localhost:3000](http://localhost:3000), accessible for other docker containers as `http://grafana:3000`.
- FiinSoft:
  - Security API on [http://localhost:8001](http://localhost:8001/swagger), accessible for other docker containers as `http://security-api`.
  - Tenants API on [http://localhost:8002](http://localhost:8002/swagger), accessible for other docker containers as `http://tenants-api`.
  - Core API on [http://localhost:8003](http://localhost:8003/swagger), accessible for other docker containers as `http://core-api`.

#### Only Third-party

- Create an `.env` file with following content:

  ```properties
  MSSQL_SA_PWD="<your-super-secure-password>"
  # Security API: JWT
  JWT_ISSUER="<jwt-issuer-name>"
  JWT_AUDIENCE="<jwt-audience-name>"
  JWT_EXPIRATION="<jwt-expiration-in-seconds>"
  JWT_REFRESH="<jwt-refresh-expiration-in-seconds>"
  JWT_KEY="<jwt-signing-key>"
  ```

- Execute:

  ```bash
  docker-compose --env-file .env -f docker-compose.services.yml -p fiinsoft up --build
  ```

- To cleanup containers, volumes and images created by `docker-compose`:

  ```bash
  # Option 1:
  docker compose --env-file '.env' -f 'docker-compose.services.yml' -p 'fiinsoft' down -v

  # Option 2:
  docker rm -f $(docker container ls -f name=fiinsoft_ --all -q)
  docker volume rm $(docker volume ls -f name=fiinsoft_ --format "{{.Name}}")
  ```

#### All services

- Create an `.env` file with following content:

  ```properties
  # Microsoft SQL Server
  MSSQL_SA_PWD="<your-super-secure-password>"
  # Security API: JWT
  JWT_ISSUER="<jwt-issuer-name>"
  JWT_AUDIENCE="<jwt-audience-name>"
  JWT_EXPIRATION="<jwt-expiration-in-seconds>"
  JWT_REFRESH="<jwt-refresh-expiration-in-seconds>"
  JWT_KEY="<jwt-signing-key>"
  # Security API: Legacy Database
  DB_SEC_SERVER="<security-database-server>"
  DB_SEC_NAME="<security-database-name>"
  DB_SEC_USR="<security-database-user>"
  DB_SEC_PWD="<security-database-password>"
  ```

- Execute:

  ```bash
  docker-compose --env-file .env -f docker-compose.yml -p fiinsoft up --build
  ```

- To cleanup containers, volumes and images created by `docker-compose`:

  ```bash
  # Option 1:
  docker compose --env-file '.env' -f 'docker-compose.yml' -p 'fiinsoft' down -v
  docker rmi -f $(docker images -f "reference=fiinsoft_*" -q)

  # Option 2:
  docker rm -f $(docker container ls -f name=fiinsoft_ --all -q)
  docker rmi -f $(docker images -f "reference=fiinsoft_*" -q)
  docker volume rm $(docker volume ls -f name=fiinsoft_ --format "{{.Name}}")
  ```

## Running from VSCode

There are some tasks/launch configurations included to make it easier to run from VSCode even locally or in docker.

- Prerrequisites
  - [Git](https://git-scm.com/downloads)
  - [Visual Studio Code](https://code.visualstudio.com)
  - [.NET 6.0 SDK](https://dotnet.microsoft.com/en-us/download/dotnet/6.0)
  - [Docker Desktop](https://www.docker.com/products/docker-desktop)
  - [SQL Server](https://www.microsoft.com/en-us/sql-server/sql-server-downloads) ([docker image](https://hub.docker.com/_/microsoft-mssql-server) also works)

1. Clone current repository:

    ```bash
    git clone ssh://<your-user>@source.developers.google.com:2022/p/fiinsoft-desarrollo/r/fiinsoft-core-V2
    ```

2. Open solution in VSCode:

    ```bash
    code fiinsoft-core-V2
    ```

3. Run locally:

    - Restore dependencies and build: Menu → _Terminal_ → _Run Task..._ or _Run Build Task..._
      - **All** → `build-all`
      - **`Security API`** → `build-security`
      - **`Tenants API`** → `build-tenants`
      - **`Core API`** → `build-core`
    - Run **`Security API`**:
      - Edit `.vscode/launch.json` file and set security databases connection information on lines 19-26, Loki/OpenTelemetry URLs on lines 27-28, JWT information on lines 29-33
      - Side Bar → _Run and Debug_ → `Security API` → Menu → Run → _Start Debugging_ or _Run Without Debugging_
    - Run **`Tenants API`**:
      - Edit `.vscode/launch.json` file and set tenants database connection information on lines 52-55, Loki/OpenTelemetry URLs on lines 56-57, JWT information on lines 58-60
      - Side Bar → _Run and Debug_ → `Tenants API` → Menu → Run → _Start Debugging_ or _Run Without Debugging_
    - Run **`Core API`**:
      - Edit `.vscode/launch.json` file and set Loki/OpenTelemetry URLs on lines 79-80, JWT information on lines 81-83, Security API information on lines 84-86, Tenants API information on lines 87-89
      - Side Bar → _Run and Debug_ → `Core API` → Menu → Run → _Start Debugging_ or _Run Without Debugging_

4. Run in docker:

    - Build docker image: Menu → _Terminal_ → _Run Task..._
      - **`Security API`** → `docker.build-security`
      - **`Tenants API`** → `docker.build-tenants`
      - **`Core API`** → `docker.build-core`
    - Run docker container:
      - **`Security API`**:
        - Edit `.vscode/tasks.json` file and set security databases connection information on lines 456, 458, 460, 462, 464, 466, 468, 470, JWT information on lines 472, 474, 476, 478, 480
        - Menu → _Terminal_ → _Run Task..._ → `docker.run-security`
      - **`Tenants API`**:
        - Edit `.vscode/tasks.json` file and set tenants database connection information on lines 510, 512, 514, 516, JWT information on lines 518, 520, 522
        - Menu → _Terminal_ → _Run Task..._ → `docker.run-tenants`
      - **`Core API`**:
        - Edit `.vscode/tasks.json` file and set JWT information on lines 552, 554, 556
        - Menu → _Terminal_ → _Run Task..._ → `docker.run-core`

5. `docker-compose` _services_:

    1. Create `.env` file as described in [docker-compose _"Only Third-party"_](#only-third-party)
    2. **_Up_**: Menu → _Terminal_ → _Run Task..._ → `docker-compose-up.services`
    3. **_Down_**: Menu → _Terminal_ → _Run Task..._ → `docker-compose-down.services`
