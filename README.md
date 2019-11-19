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
