# mysql-kubernetes-cluster

### Create configmap for mysql DB ###

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: mysql
      labels:
        app: mysql
    data:
      master.cnf: |
        # Apply this config only on first pod .
        [mysqld]
        log-bin="mysql-bin"
        binlog-ignore-db=information_schema
        replicate-ignore-db=information_schema
        relay-log="mysql-relay-log"
      slave.cnf: |
        # Apply this config only on second pod .
        [mysqld]
        log-bin="mysql-bin"
        binlog-ignore-db=information_schema
        replicate-ignore-db=information_schema
        relay-log="mysql-relay-log"
### Create the configmap ###

    kubectl apply -f https://raw.githubusercontent.com/phaniamt/mysql-kubernetes-cluster/master/mysql-configmap.yaml

### Create two services for mysql.### 
### services for stable DNS entries of StatefulSet members and connect the applications to db. ###
    # Headless service for stable DNS entries of StatefulSet members.
    apiVersion: v1
    kind: Service
    metadata:
      name: mysql
      labels:
        app: mysql
    spec:
      ports:
      - name: mysql
        port: 3306
      clusterIP: None
      selector:
        app: mysql
    # Connect to DB from applications 
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: mysql-db
      labels:
        app: mysql
    spec:
      ports:
      - name: mysql
        port: 3306
      selector:
        app: mysql
### Create the services ###

    kubectl apply -f https://raw.githubusercontent.com/phaniamt/mysql-kubernetes-cluster/master/mysql-services.yaml
