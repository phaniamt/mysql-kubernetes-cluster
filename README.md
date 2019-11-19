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
