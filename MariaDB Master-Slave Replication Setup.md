# MariaDB Master-Slave Replication Setup

This guide demonstrates how to deploy two MariaDB engines and set up master-to-slave replication between them using Kubernetes manifests. The replication ensures that data changes on the master database are automatically propagated to the slave database.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Deployment Manifests](#deployment-manifests)
  - [Master Deployment (master.yaml)](#master-deployment-masteryaml)
  - [Slave Deployment (slave.yaml)](#slave-deployment-slaveyaml)
  - [Replication Monitor (cronjob.yaml)](#replication-monitor-cronjobyaml)
- [Steps to Deploy](#steps-to-deploy)
- [Verification](#verification)
- [Replication Explained](#replication-explained)

## Prerequisites

- Kubernetes cluster configured and accessible
- `kubectl` command-line tool installed
- Basic understanding of Kubernetes concepts

## Deployment Manifests

### Master Deployment (master.yaml)

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mariadb-master-config
data:
  my.cnf: |
    [mariadb]
    # Master Server ID
    server-id = 1

    # Replication Configurations
    binlog-format = ROW
    binlog-ignore-db = information_schema
    binlog-ignore-db = mysql
    binlog-ignore-db = performance_schema
    binlog-ignore-db = sys
    expire-logs-days = 7
    log-bin = mariadb-bin
    log_warnings = 3
    max_binlog_size = 100M

  replication.sql: |
    CREATE USER 'replication_user'@'%' IDENTIFIED BY 'replsecret';
    GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';
    FLUSH PRIVILEGES;
---
apiVersion: v1
kind: Secret
metadata:
  name: mariadb-secrets
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: c2VjcmV0
---
apiVersion: v1
kind: Service
metadata:
  name: mariadb-master-service
spec:
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
  selector:
    app: mariadb-master
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb-master
spec:
  serviceName: mariadb-master-service
  replicas: 1
  selector:
    matchLabels:
      app: mariadb-master
  template:
    metadata:
      labels:
        app: mariadb-master
    spec:
      containers:
        - name: mariadb-master
          image: mariadb:latest
          ports:
            - containerPort: 3306
          envFrom:
            - configMapRef:
                name: mariadb-master-config
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mariadb-secrets
                  key: MYSQL_ROOT_PASSWORD
          readinessProbe:
            tcpSocket:
              port: 3306
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          livenessProbe:
            tcpSocket:
              port: 3306
            initialDelaySeconds: 20
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          startupProbe:
            tcpSocket:
              port: 3306
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 5
          volumeMounts:
            - name: mariadb-config-volume
              mountPath: /etc/mysql/conf.d/my.cnf
              subPath: my.cnf
            - name: mariadb-persistent-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mariadb-config-volume
          configMap:
            name: mariadb-master-config
        - name: mariadb-persistent-storage
          persistentVolumeClaim:
            claimName: mariadb-persistent-storage
  volumeClaimTemplates:
    - metadata:
        name: mariadb-persistent-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
---
apiVersion: batch/v1
kind: Job
metadata:
  name: mariadb-master-initialization
spec:
  completions: 1
  template:
    metadata:
      name: mariadb-master-initialization
    spec:
      initContainers:
        - name: wait-for-mariadb-master
          image: alpine:latest
          command: ["sh", "-c", "until nc -z mariadb-master-service 3306; do echo 'Waiting for MariaDB master to be ready...'; sleep 5; done"]
      containers:
        - name: mariadb-master-initialization
          image: alpine:latest
          command:
            - "sh"
            - "-c"
            - |
              # Install telnet client and MySQL client
              apk update && \
              apk add --no-cache mysql-client

              # Function to check if the MariaDB service is ready
              wait_for_mariadb() {
                  until nc -z mariadb-master-service 3306; do
                      echo "Waiting for MariaDB service to be ready..."
                      sleep 5
                  done
              }

              # Wait until the MariaDB service is ready
              wait_for_mariadb

              # Once the service is ready, proceed with initialization
              echo "Executing MySQL command to initialize MariaDB:"
              mysql_output=$(mysql -h mariadb-master-service -uroot -p"$MYSQL_ROOT_PASSWORD" < /etc/mysql/init/replication.sql 2>&1)
              if [ $? -eq 0 ]; then
                  echo "SQL query processed successfully."
              else
                  echo "Failed to process SQL query. Error message: $mysql_output"
                  exit 1
              fi
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mariadb-secrets
                  key: MYSQL_ROOT_PASSWORD
          volumeMounts:
            - name: mariadb-config-volume
              mountPath: /etc/mysql/init
      restartPolicy: Never
      volumes:
        - name: mariadb-config-volume
          configMap:
            name: mariadb-master-config

```

### Slave Deployment (slave.yaml)

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mariadb-slave-config
data:
  my.cnf: |
    [mariadb]
    # Slave Server ID
    server-id = 2

    # Replication Configurations
    binlog-format = ROW
    expire-logs-days = 7
    log-bin = mariadb-bin
    log-slave-updates = 1
    log_warnings = 3
    max_binlog_size = 100M
    relay-log = mariadb-relay-bin
    replicate-ignore-db = information_schema
    replicate-ignore-db = mysql
    replicate-ignore-db = performance_schema
    replicate-ignore-db = sys

  replication.sql: |
    CHANGE MASTER TO
      MASTER_HOST = 'mariadb-master-service',
      MASTER_USER = 'replication_user',
      MASTER_PASSWORD = 'replsecret',
      MASTER_PORT = 3306,
      MASTER_CONNECT_RETRY = 10;
    START SLAVE;
---
apiVersion: v1
kind: Service
metadata:
  name: mariadb-slave-service
spec:
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
  selector:
    app: mariadb-slave
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mariadb-slave
spec:
  serviceName: mariadb-slave-service
  replicas: 1
  selector:
    matchLabels:
      app: mariadb-slave
  template:
    metadata:
      labels:
        app: mariadb-slave
    spec:
      containers:
        - name: mariadb-slave
          image: mariadb:latest
          ports:
            - containerPort: 3306
          envFrom:
            - configMapRef:
                name: mariadb-slave-config
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mariadb-secrets
                  key: MYSQL_ROOT_PASSWORD
          readinessProbe:
            tcpSocket:
              port: 3306
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          livenessProbe:
            tcpSocket:
              port: 3306
            initialDelaySeconds: 20
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          startupProbe:
            tcpSocket:
              port: 3306
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 5
          volumeMounts:
            - name: mariadb-config-volume
              mountPath: /etc/mysql/conf.d/my.cnf
              subPath: my.cnf
            - name: mariadb-persistent-storage
              mountPath: /var/lib/mysql
      volumes:
        - name: mariadb-config-volume
          configMap:
            name: mariadb-slave-config
        - name: mariadb-persistent-storage
          persistentVolumeClaim:
            claimName: mariadb-persistent-storage
  volumeClaimTemplates:
    - metadata:
        name: mariadb-persistent-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
---
apiVersion: batch/v1
kind: Job
metadata:
  name: mariadb-slave-initialization
spec:
  completions: 1
  template:
    metadata:
      name: mariadb-slave-initialization
    spec:
      initContainers:
        - name: wait-for-mariadb-master
          image: alpine:latest
          command: ["sh", "-c", "until nc -z mariadb-master-service 3306; do echo 'Waiting for MariaDB master to be ready...'; sleep 5; done"]
        - name: wait-for-mariadb-slave
          image: alpine:latest
          command: ["sh", "-c", "until nc -z mariadb-slave-service 3306; do echo 'Waiting for MariaDB slave to be ready...'; sleep 5; done"]
      containers:
        - name: mariadb-slave-initialization
          image: alpine:latest
          command:
            - "sh"
            - "-c"
            - |
              # Install telnet client and MySQL client
              apk update && \
              apk add --no-cache mysql-client

              # Function to check if both MariaDB services are ready
              wait_for_mariadb() {
                  until nc -z mariadb-master-service 3306 && nc -z mariadb-slave-service 3306; do
                      echo "Waiting for MariaDB services to be ready..."
                      sleep 5
                  done
              }

              # Wait until both MariaDB services are ready
              wait_for_mariadb

              # Once the services are ready, proceed with initialization
              echo "Executing MySQL command to initialize MariaDB slave:"
              mysql_output=$(mysql -h mariadb-slave-service -uroot -p"$MYSQL_ROOT_PASSWORD" < /etc/mysql/init/replication.sql 2>&1)
              if [ $? -eq 0 ]; then
                  echo "SQL query processed successfully."
              else
                  echo "Failed to process SQL query. Error message: $mysql_output"
                  exit 1
              fi
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mariadb-secrets
                  key: MYSQL_ROOT_PASSWORD
          volumeMounts:
            - name: mariadb-config-volume
              mountPath: /etc/mysql/init
      restartPolicy: Never
      volumes:
        - name: mariadb-config-volume
          configMap:
            name: mariadb-slave-config

```

### Replication Monitor (cronjob.yaml)
```yaml
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mariadb-replication-monitor
spec:
  schedule: "*/5 * * * *"  # Run every 5 minutes
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 5
  jobTemplate:
    spec:
      template:
        spec:
          initContainers:
            - name: wait-for-mariadb-master
              image: alpine:latest
              command: ["sh", "-c", "until nc -z mariadb-master-service 3306; do echo 'Waiting for MariaDB master to be ready...'; sleep 5; done"]
            - name: wait-for-mariadb-slave
              image: alpine:latest
              command: ["sh", "-c", "until nc -z mariadb-slave-service 3306; do echo 'Waiting for MariaDB slave to be ready...'; sleep 5; done"]
          containers:
            - name: mariadb-replication-monitor
              image: alpine:latest
              command: ["/bin/sh", "-c"]
              args:
                - |
                  # Set variables
                  MARIADB_SLAVE_SERVICE="mariadb-slave-service"
                  MARIADB_PORT="3306"

                  # Install MySQL client
                  apk update && \
                  apk add --no-cache mysql-client

                  # Function to check replication status on slave
                  check_replication_status() {
                    local service="$1"
                    local query="$2"
                    local result=$(mysql -h "$service" -uroot -p"$MYSQL_ROOT_PASSWORD" -e "$query" 2>&1)
                    if [ $? -eq 0 ]; then
                      echo "$result"
                    else
                      echo "Failed to retrieve replication status from $service. Aborting..."
                      exit 1
                    fi
                  }

                  # Function to check if the Last_SQL_Error contains specific phrases
                  check_sql_error() {
                    local status="$1"
                    if echo "$status" | grep -q "Last_SQL_Error:"; then
                      local error=$(echo "$status" | grep "Last_SQL_Error:" | sed 's/Last_SQL_Error://' | sed 's/^[ \t]*//')
                      if echo "$error" | grep -q "Duplicate entry" && echo "$error" | grep -q "insert into FilledLots"; then
                        echo "Duplicate entry error detected: $error"
                        return 0
                      fi
                    fi
                    return 1
                  }

                  # Function to get the value of a specific parameter from the replication status
                  get_replication_status_param() {
                    local status="$1"
                    local param="$2"
                    echo "$status" | grep "$param" | awk '{ print $2 }'
                  }

                  # Function to check replication status
                  check_replication() {
                    local io_status="$1"
                    local sql_status="$2"
                    if [ "$io_status" = "Yes" ] && [ "$sql_status" = "Yes" ]; then
                      echo "Replication is running"
                    else
                      echo "Replication is not running. Attempting to restart the slave..."
                      mysql -h "$MARIADB_SLAVE_SERVICE" -uroot -p"$MYSQL_ROOT_PASSWORD" -e "START SLAVE;" 2>&1
                    fi
                  }

                  # Function to skip one error and restart slave
                  skip_error_and_restart_slave() {
                    echo "Replication is not running. Attempting to skip one error and restart the slave..."
                    mysql -h "$MARIADB_SLAVE_SERVICE" -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SET GLOBAL sql_slave_skip_counter = 1; START SLAVE;" 2>&1
                  }

                  # Query to check replication status on slave
                  QUERY="SHOW SLAVE STATUS \G;"

                  # Execute the query and parse the results
                  STATUS=$(check_replication_status "$MARIADB_SLAVE_SERVICE" "$QUERY")
                  IO_STATUS=$(get_replication_status_param "$STATUS" "Slave_IO_Running:")
                  SQL_STATUS=$(get_replication_status_param "$STATUS" "Slave_SQL_Running:")

                  # Check if Last_SQL_Error contains specific phrases and take appropriate action
                  if check_sql_error "$STATUS"; then
                    skip_error_and_restart_slave
                  else
                    check_replication "$IO_STATUS" "$SQL_STATUS"
                  fi

                  # Add logging for replication check
                  echo "Replication check completed at $(date)"

              env:
                - name: MYSQL_ROOT_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: mariadb-secrets
                      key: MYSQL_ROOT_PASSWORD
          restartPolicy: OnFailure

```

### Steps to Deploy

**1. Apply the master deployment manifest:**
```sh
kubectl apply -f master.yaml
```

**2. Apply the slave deployment manifest:**
```sh
kubectl apply -f slave.yaml
```
**2. Apply the cronjob monitor manifest:**
```sh
kubectl apply -f cronjob.yaml
```

### Verification

Once both deployments are successfully created, verify that the replication is functioning correctly:

**1. Access the master database and create a test database and table:**
```sql
mysql -h mariadb-master-service -u root -p
CREATE DATABASE testdb;
USE testdb;
CREATE TABLE testtable (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(255));
```
**2. Insert some data into the test table:**
```sql
INSERT INTO testtable (name) VALUES ('Data from Master');
```

**3. Access the slave database and check if the data is replicated:**
```sql
mysql -h mariadb-slave-service -u root -p
USE testdb;
SELECT * FROM testtable;
```

Ensure that the data inserted into the master is replicated in the slave.

### Replication Explained

MariaDB replication involves two main processes:

- `Master`: The master database server where the original data resides.
- `Slave`: The replica database server that copies and maintains an identical copy of the data from the master.

Replication works by transferring binary logs (binlogs) from the master to the slave. The slave server reads the binlog.