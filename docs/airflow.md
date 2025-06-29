## Tutorials

[marclamberti](https://marclamberti.com/)

## Run with Docker

[Airflow Installlation with docker images](https://airflow.apache.org/docs/apache-airflow/stable/installation/index.html#using-production-docker-images)

### docker compose file (w/ postgres + mongodb + rabbitmq + celery)

```
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.
#

# Basic Airflow cluster configuration for CeleryExecutor with Redis and PostgreSQL.
#
# WARNING: This configuration is for local development. Do not use it in a production deployment.
#
# This configuration supports basic configuration using environment variables or an .env file
# The following variables are supported:
#
# AIRFLOW_IMAGE_NAME           - Docker image name used to run Airflow.
#                                Default: apache/airflow:3.0.1
# AIRFLOW_UID                  - User ID in Airflow containers
#                                Default: 50000
# AIRFLOW_PROJ_DIR             - Base path to which all the files will be volumed.
#                                Default: .
# Those configurations are useful mostly in case of standalone testing/running Airflow in test/try-out mode
#
# _AIRFLOW_WWW_USER_USERNAME   - Username for the administrator account (if requested).
#                                Default: airflow
# _AIRFLOW_WWW_USER_PASSWORD   - Password for the administrator account (if requested).
#                                Default: airflow
# _PIP_ADDITIONAL_REQUIREMENTS - Additional PIP requirements to add when starting all containers.
#                                Use this option ONLY for quick checks. Installing requirements at container
#                                startup is done EVERY TIME the service is started.
#                                A better way is to build a custom image or extend the official image
#                                as described in https://airflow.apache.org/docs/docker-stack/build.html.
#                                Default: ''
#
# Feel free to modify this file to suit your needs.
---
x-airflow-common: &airflow-common
  # In order to add custom dependencies or upgrade provider distributions you can use your extended image.
  # Comment the image line, place your Dockerfile in the directory where you placed the docker-compose.yaml
  # and uncomment the "build" line below, Then run `docker-compose build` to build the images.
  image: ${AIRFLOW_IMAGE_NAME:-apache/airflow:3.0.1}
  # build: .
  environment: &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: CeleryExecutor
    AIRFLOW__CORE__AUTH_MANAGER: airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
    # AIRFLOW__CELERY__BROKER_URL: redis://:@redis:6379/0
    AIRFLOW__CELERY__BROKER_URL: amqp://:@rabbitmq:5672/airflow-vhost
    AIRFLOW__CORE__FERNET_KEY: ""
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: "true"
    AIRFLOW__CORE__LOAD_EXAMPLES: "true"
    AIRFLOW__CORE__EXECUTION_API_SERVER_URL: "http://airflow-apiserver:8080/execution/"
    # yamllint disable rule:line-length
    # Use simple http server on scheduler for health checks
    # See https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/logging-monitoring/check-health.html#scheduler-health-check-server
    # yamllint enable rule:line-length
    AIRFLOW__SCHEDULER__ENABLE_HEALTH_CHECK: "true"
    # WARNING: Use _PIP_ADDITIONAL_REQUIREMENTS option ONLY for a quick checks
    # for other purpose (development, test and especially production usage) build/extend Airflow image.
    _PIP_ADDITIONAL_REQUIREMENTS: ${_PIP_ADDITIONAL_REQUIREMENTS:-}
    # The following line can be used to set a custom config file, stored in the local config folder
    AIRFLOW_CONFIG: "/opt/airflow/config/airflow.cfg"
  volumes:
    - ${AIRFLOW_PROJ_DIR:-.}/dags:/opt/airflow/dags
    - ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs
    - ${AIRFLOW_PROJ_DIR:-.}/config:/opt/airflow/config
    - ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins
    - "/var/run/docker.sock:/var/run/docker.sock" # We will pass the Docker Deamon as a volume to allow the webserver containers start docker images. Ref: https://stackoverflow.com/q/51342810/7024760
    - "/tmp:/tmp"
  user: "${AIRFLOW_UID:-50000}:0"
  depends_on: &airflow-common-depends-on
    rabbitmq:
      condition: service_healthy
    postgres:
      condition: service_healthy
    mongo:
      condition: service_healthy

services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
      TEST_APP_USER: test_app_user
      TEST_APP_PASSWORD: test_app
      TEST_APP_DB: test_app
    ports:
      - "5432:5432"
    volumes:
      - postgres-db-volume:/var/lib/postgresql/data
      - ./init_scripts/sql/init_db.sh:/docker-entrypoint-initdb.d/init_db.sh
      - ./init_scripts/sql:/sql
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "airflow"]
      interval: 10s
      retries: 5
      start_period: 5s
    restart: always

  mongo:
    image: mongo:7.0.0
    restart: always
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 5s
      timeout: 5s
      retries: 3
      start_period: 5s
    environment:
      MONGO_INITDB_ROOT_USERNAME: mongo_test_db
      MONGO_INITDB_ROOT_PASSWORD: mongo_test_db
    ports:
      - "27017:27017"

  rabbitmq:
    image: rabbitmq:4.1.0-management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      - RABBITMQ_DEFAULT_USER=guest
      - RABBITMQ_DEFAULT_PASS=guest
    healthcheck:
      test: rabbitmq-diagnostics -q ping
      interval: 10s
      timeout: 30s
      retries: 50
      start_period: 30s
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
      - ./init_scripts/rabbitmq/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
      - ./init_scripts/rabbitmq/definitions.json:/etc/rabbitmq/definitions.json
    restart: always

  airflow-apiserver:
    <<: *airflow-common
    command: api-server
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080/api/v2/version"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-scheduler:
    <<: *airflow-common
    command: scheduler
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8974/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-dag-processor:
    <<: *airflow-common
    command: dag-processor
    healthcheck:
      test:
        [
          "CMD-SHELL",
          'airflow jobs check --job-type DagProcessorJob --hostname "$${HOSTNAME}"',
        ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-worker:
    <<: *airflow-common
    command: celery worker
    healthcheck:
      # yamllint disable rule:line-length
      test:
        - "CMD-SHELL"
        - 'celery --app airflow.providers.celery.executors.celery_executor.app inspect ping -d "celery@$${HOSTNAME}" || celery --app airflow.executors.celery_executor.app inspect ping -d "celery@$${HOSTNAME}"'
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    environment:
      <<: *airflow-common-env
      # Required to handle warm shutdown of the celery workers properly
      # See https://airflow.apache.org/docs/docker-stack/entrypoint.html#signal-propagation
      DUMB_INIT_SETSID: "0"
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-apiserver:
        condition: service_healthy
      airflow-init:
        condition: service_completed_successfully

  airflow-triggerer:
    <<: *airflow-common
    command: triggerer
    healthcheck:
      test:
        [
          "CMD-SHELL",
          'airflow jobs check --job-type TriggererJob --hostname "$${HOSTNAME}"',
        ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-init:
    <<: *airflow-common
    entrypoint: /bin/bash
    # yamllint disable rule:line-length
    command:
      - -c
      - |
        if [[ -z "${AIRFLOW_UID}" ]]; then
          echo
          echo -e "\033[1;33mWARNING!!!: AIRFLOW_UID not set!\e[0m"
          echo "If you are on Linux, you SHOULD follow the instructions below to set "
          echo "AIRFLOW_UID environment variable, otherwise files will be owned by root."
          echo "For other operating systems you can get rid of the warning with manually created .env file:"
          echo "    See: https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#setting-the-right-airflow-user"
          echo
          export AIRFLOW_UID=$(id -u)
        fi
        one_meg=1048576
        mem_available=$$(($$(getconf _PHYS_PAGES) * $$(getconf PAGE_SIZE) / one_meg))
        cpus_available=$$(grep -cE 'cpu[0-9]+' /proc/stat)
        disk_available=$$(df / | tail -1 | awk '{print $$4}')
        warning_resources="false"
        if (( mem_available < 4000 )) ; then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough memory available for Docker.\e[0m"
          echo "At least 4GB of memory required. You have $$(numfmt --to iec $$((mem_available * one_meg)))"
          echo
          warning_resources="true"
        fi
        if (( cpus_available < 2 )); then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough CPUS available for Docker.\e[0m"
          echo "At least 2 CPUs recommended. You have $${cpus_available}"
          echo
          warning_resources="true"
        fi
        if (( disk_available < one_meg * 10 )); then
          echo
          echo -e "\033[1;33mWARNING!!!: Not enough Disk space available for Docker.\e[0m"
          echo "At least 10 GBs recommended. You have $$(numfmt --to iec $$((disk_available * 1024 )))"
          echo
          warning_resources="true"
        fi
        if [[ $${warning_resources} == "true" ]]; then
          echo
          echo -e "\033[1;33mWARNING!!!: You have not enough resources to run Airflow (see above)!\e[0m"
          echo "Please follow the instructions to increase amount of resources available:"
          echo "   https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#before-you-begin"
          echo
        fi
        echo
        echo "Creating missing opt dirs if missing:"
        echo
        mkdir -v -p /opt/airflow/{logs,dags,plugins,config}
        echo
        echo "Airflow version:"
        /entrypoint airflow version
        echo
        echo "Files in shared volumes:"
        echo
        ls -la /opt/airflow/{logs,dags,plugins,config}
        echo
        echo "Running airflow config list to create default config file if missing."
        echo
        /entrypoint airflow config list >/dev/null
        echo
        echo "Files in shared volumes:"
        echo
        ls -la /opt/airflow/{logs,dags,plugins,config}
        echo
        echo "Change ownership of files in /opt/airflow to ${AIRFLOW_UID}:0"
        echo
        chown -R "${AIRFLOW_UID}:0" /opt/airflow/
        echo
        echo "Change ownership of files in shared volumes to ${AIRFLOW_UID}:0"
        echo
        chown -v -R "${AIRFLOW_UID}:0" /opt/airflow/{logs,dags,plugins,config}
        echo
        echo "Files in shared volumes:"
        echo
        ls -la /opt/airflow/{logs,dags,plugins,config}

    # yamllint enable rule:line-length
    environment:
      <<: *airflow-common-env
      _AIRFLOW_DB_MIGRATE: "true"
      _AIRFLOW_WWW_USER_CREATE: "true"
      _AIRFLOW_WWW_USER_USERNAME: ${_AIRFLOW_WWW_USER_USERNAME:-airflow}
      _AIRFLOW_WWW_USER_PASSWORD: ${_AIRFLOW_WWW_USER_PASSWORD:-airflow}
      _PIP_ADDITIONAL_REQUIREMENTS: ""
    user: "0:0"

  airflow-cli:
    <<: *airflow-common
    profiles:
      - debug
    environment:
      <<: *airflow-common-env
      CONNECTION_CHECK_MAX_COUNT: "0"
    # Workaround for entrypoint issue. See: https://github.com/apache/airflow/issues/16252
    command:
      - bash
      - -c
      - airflow
    depends_on:
      <<: *airflow-common-depends-on

  # You can enable flower by adding "--profile flower" option e.g. docker-compose --profile flower up
  # or by explicitly targeted on the command line e.g. docker-compose up flower.
  # See: https://docs.docker.com/compose/profiles/
  flower:
    <<: *airflow-common
    command: celery flower
    profiles:
      - flower
    ports:
      - "5555:5555"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:5555/"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

volumes:
  postgres-db-volume:
  rabbitmq-data:

```

### Dockerfile (Running airflow with additional modules/requirements.txt)

```
FROM apache/airflow:2.7.0-python3.11

USER root
RUN apt-get update \
	&& apt-get install -y --no-install-recommends \
	vim \
	&& apt-get autoremove -yqq --purge \
	&& apt-get clean \
	&& rm -rf /var/lib/apt/lists/*
USER airflow

COPY requirements.txt /
RUN pip install --no-cache-dir "apache-airflow==${AIRFLOW_VERSION}" -r /requirements.txt
```

### Populate/SQL migration on Postgres docker start

```
      - ./init_scripts/sql/init_db.sh:/docker-entrypoint-initdb.d/init_db.sh
      - ./init_scripts/sql:/sql
```

Add SQL scripts to the `/init_scripts/sql` directory e.g.:

```SQL
-- ./init_scripts/sql/001-init_tables.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE IF NOT EXISTS departments (
    id uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NULL
);

CREATE TABLE IF NOT EXISTS employees (
    id uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
    email VARCHAR(255) UNIQUE NOT NULL,
    first_name VARCHAR(255) NOT NULL,
    last_name VARCHAR(255) NOT NULL,
    department_id uuid NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NULL,
    FOREIGN KEY (department_id) REFERENCES departments(id) ON DELETE CASCADE
);

INSERT INTO
    departments(name)
VALUES
    ('HR'), ('Finance');

INSERT INTO
    employees(email, first_name, last_name, department_id)
VALUES
    ('mary@test_co.com', 'Mary', 'Smith', (SELECT id from departments WHERE name='Finance')),
    ('dave@test_co.com', 'Dave', 'Cole', (SELECT id from departments WHERE name='Finance')),
    ('jane@test_co.com', 'Jane', 'Hills', (SELECT id from departments WHERE name='Finance')),
    ('john@test_co.com', 'John', 'Doe', (SELECT id from departments WHERE name='HR'));
```

Add DB migrations files or SQL commands to `init_db.sh`

```bash
#!/bin/bash

psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" -d "$POSTGRES_DB"  <<-EOSQL
  CREATE USER $TEST_APP_USER WITH PASSWORD '$TEST_APP_PASSWORD';
	CREATE DATABASE $TEST_APP_DB;
	GRANT ALL PRIVILEGES ON DATABASE $TEST_APP_DB TO $TEST_APP_USER;
EOSQL

psql -U "$TEST_APP_USER" -d "$TEST_APP_DB" -a -f /sql/001-init_tables.sql
```

### RabbitMQ config & definitions

rabbitmq.conf

```json
load_definitions = /etc/rabbitmq/definitions.json
```

definitions.json

```json
{
  "rabbit_version": "4.1.0",
  "rabbitmq_version": "4.1.0",
  "product_name": "RabbitMQ",
  "product_version": "4.1.0",
  "rabbitmq_definition_format": "cluster",
  "original_cluster_name": "rabbit@2fa62a014fed",
  "explanation": "Definitions of cluster 'rabbit@2fa62a014fed'",
  "users": [
    {
      "name": "guest",
      "password_hash": "fIkEZt6PCkcBXHIEpaG4GEeW/WDN9iHQJKS23eCX6v8vrvnQ",
      "hashing_algorithm": "rabbit_password_hashing_sha256",
      "tags": ["administrator"],
      "limits": {}
    }
  ],

  "vhosts": [
    {
      "name": "airflow-vhost",
      "description": "Virtual host for Airflow",
      "metadata": {
        "description": "Virtual host for Airflow",
        "tags": ["airflow"],
        "default_queue_type": "classic"
      },
      "tags": ["airflow"],
      "default_queue_type": "classic"
    },
    {
      "name": "rpc-vhost",
      "description": "Virtual host for RPC",
      "metadata": {
        "description": "Virtual host for RPC",
        "tags": ["rpc"],
        "default_queue_type": "classic"
      },
      "tags": ["rpc"],
      "default_queue_type": "classic"
    },
    {
      "name": "/",
      "description": "Default virtual host",
      "metadata": {
        "description": "Default virtual host",
        "tags": [],
        "default_queue_type": "classic"
      },
      "tags": [],
      "default_queue_type": "classic"
    }
  ],
  "permissions": [
    {
      "user": "guest",
      "vhost": "airflow-vhost",
      "configure": ".*",
      "write": ".*",
      "read": ".*"
    },
    {
      "user": "guest",
      "vhost": "rpc-vhost",
      "configure": ".*",
      "write": ".*",
      "read": ".*"
    },
    {
      "user": "guest",
      "vhost": "/",
      "configure": ".*",
      "write": ".*",
      "read": ".*"
    }
  ],
  "topic_permissions": [],
  "parameters": [],
  "global_parameters": [
    { "name": "cluster_tags", "value": [] },
    {
      "name": "internal_cluster_id",
      "value": "rabbitmq-cluster-id-UrGfZKlCFsC_mmTFjvTBUw"
    }
  ],
  "policies": [],
  "queues": [
    {
      "name": "rpc-queue",
      "vhost": "rpc-vhost",
      "durable": true,
      "auto_delete": false,
      "arguments": { "x-queue-type": "classic" }
    },
    {
      "name": "stream-queue",
      "vhost": "/",
      "durable": true,
      "auto_delete": false,
      "arguments": { "x-queue-type": "classic" }
    }
  ],
  "exchanges": [],
  "bindings": []
}
```
