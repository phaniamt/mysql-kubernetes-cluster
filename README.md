# mysql-kubernetes-cluster

### Reference URLs

https://www.howtoforge.com/tutorial/how-to-setup-mysql-master-master-replication/

https://www.ryadel.com/en/mysql-master-master-replication-setup-in-5-easy-steps/

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
 
### Deploy the mysql statefulset with two master replicas ###

    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: mysql
    spec:
      selector:
        matchLabels:
          app: mysql
      serviceName: mysql
      replicas: 2
      template:
        metadata:
          labels:
            app: mysql
        spec:
          initContainers:
          - name: init-mysql
            image: mysql:5.7
            command:
            - bash
            - "-c"
            - |
              set -ex
              # Generate mysql server-id from pod ordinal index.
              [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
              ordinal=${BASH_REMATCH[1]}
              echo [mysqld] > /mnt/conf.d/server-id.cnf
              # Add an offset to avoid reserved server-id=0 value.
              echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
              # Copy appropriate conf.d files from config-map to emptyDir.
              if [[ $ordinal -eq 0 ]]; then
                cp /mnt/config-map/master.cnf /mnt/conf.d/
              else
                cp /mnt/config-map/slave.cnf /mnt/conf.d/
              fi
            volumeMounts:
            - name: conf
              mountPath: /mnt/conf.d
            - name: config-map
              mountPath: /mnt/config-map
          containers:
          - name: mysql
            image: mysql:5.7
            ports:
            - name: mysql
              containerPort: 3306
            volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
              subPath: mysql
            - name: conf
              mountPath: /etc/mysql/conf.d
            env:
              # Use secret in real usage
            - name: MYSQL_ROOT_PASSWORD
              value: phani
          volumes:
          - name: conf
            emptyDir: {}
          - name: config-map
            configMap:
              name: mysql
      volumeClaimTemplates:
      - metadata:
          name: data
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 1Gi

### Deploy the statefulset ###

    kubectl apply -f https://raw.githubusercontent.com/phaniamt/mysql-kubernetes-cluster/master/mysql-sts.yaml

### Create the master to master replication ### 

    ### Connect to  mysql-0 pod ###

    kubectl exec -it mysql-0 bash


    root@mysql-0:/# mysql -u root -p


    # Crete a user replicator and grant full permissions to that user 


    mysql> CREATE USER 'replicator'@'%' IDENTIFIED BY 'phani123';

    mysql> GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%' IDENTIFIED BY 'phani123';


    mysql> SHOW MASTER STATUS;
    +------------------+----------+--------------+--------------------+-------------------+
    | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB   | Executed_Gtid_Set |
    +------------------+----------+--------------+--------------------+-------------------+
    | mysql-bin.000003 |      694 |              | information_schema |                   |
    +------------------+----------+--------------+--------------------+-------------------+
    1 row in set (0.00 sec)




    ### Connect to mysql-1 pod ###

    kubectl exec -it mysql-1 bash

    root@mysql-1:/# mysql -u root -p


    # Crete a user replicator and grant full permissions to that user


    mysql> CREATE USER 'replicator'@'%' IDENTIFIED BY 'phani123';

    mysql> GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%' IDENTIFIED BY 'phani123';

    ## Sync the data from mysql-0 to mysql-1 

    mysql> STOP SLAVE;

    # Get the MASTER_LOG_FILE and MASTER_LOG_POS from mysql-0 pod master status

    mysql> CHANGE MASTER TO MASTER_HOST = 'mysql-0.mysql', MASTER_USER = 'replicator', MASTER_PASSWORD = 'phani123', MASTER_LOG_FILE = 'mysql-bin.000003', MASTER_LOG_POS = 694; 

    mysql> START SLAVE;

    ## Now data can able to sync from mysql-0 to mysql-1




    mysql> SHOW MASTER STATUS;
    +------------------+----------+--------------+--------------------+-------------------+
    | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB   | Executed_Gtid_Set |
    +------------------+----------+--------------+--------------------+-------------------+
    | mysql-bin.000003 |      694 |              | information_schema |                   |
    +------------------+----------+--------------+--------------------+-------------------+
    1 row in set (0.00 sec)





    ### Connect to  mysql-0 pod ###

    kubectl exec -it mysql-0 bash


    root@mysql-0:/# mysql -u root -p

    ## Sync the data from mysql-1 to mysql-0 


    # Get the MASTER_LOG_FILE and MASTER_LOG_POS from mysql-1 pod master status
    mysql> STOP SLAVE;
    
    mysql> CHANGE MASTER TO MASTER_HOST = 'mysql-1.mysql', MASTER_USER = 'replicator', MASTER_PASSWORD = 'phani123', MASTER_LOG_FILE = 'mysql-bin.000003', MASTER_LOG_POS = 694; 

    mysql> START SLAVE;

    ## Now data can able to sync from mysql-0 to mysql-1

    #### Test the Data sync ###

    mysql>  SHOW DATABASES;


    kubectl run mysql-client --image=mysql:5.7 -i --rm --restart=Never --\
      mysql -h mysql-db  -u root -pphani <<EOF
    CREATE DATABASE test;
    CREATE TABLE test.messages (message VARCHAR(250));
    INSERT INTO test.messages VALUES ('hello');
    EOF


    kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
      mysql -h mysql-db -u -pphani -e "SELECT * FROM test.messages"
