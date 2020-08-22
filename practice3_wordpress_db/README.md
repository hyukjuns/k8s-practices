# Wordpress - DB in Kubernetes Cluster
### Environment
PC OS: Ubuntu 18.04
Hypervisor: KVM
    - MasterNode 1 (Centos7)
    - WorkerNode 3 (Centos7)
Memory: 16G
CPU: 5core
### Config
# Practice2: Wordpress-DB Connect

git: [https://github.com/namhj94/k8s/tree/master/practice3_wordpress_db](https://github.com/namhj94/k8s/tree/master/practice3_wordpress_db)

Wordpress-DB on Kubernetes cluster

wordpress는 deployment를 쓰고 db는 statefulset을 쓰는 이유

web은 stateless 이고, 뷸륨에 있는 파일은 읽기 작업만 수행하면되므로(콘텐츠 파일이기때문) 파드간에 볼륨을 공유해도 된다. 

반면 DB는 statefull을 유지해야하고, 볼륨에 새로운 데이터를 쓰기 작업을 해야하므로 볼륨을 공유하게되면 쓰기 작업시 충돌이 일어나게 된다. 이를 방지하려면 각각의 파드는 각자의 볼륨을 가져야하고, 마스터의 볼륨을 슬레이브의 볼륨에 동기화 해주는 방식으로 동작한다.

### Architecture

![Practice2%20Wordpress-DB%20Connect%202f2679430ce4428bb49649c21847e7b1/prac2.svg](Practice2%20Wordpress-DB%20Connect%202f2679430ce4428bb49649c21847e7b1/prac2.svg)

[prac2.svg.drawio](Practice2%20Wordpress-DB%20Connect%202f2679430ce4428bb49649c21847e7b1/prac2.svg.drawio)

### Handbook

1. 서비스 생성

    각 서비스의 포트를 다르게 해줘야 충돌을 사전에 예방할 수 있다.

    - Headless Service: wordpress web 파드에서 DB파드의 이름으로 통신을 가능하게 해준다.

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: nam-svc-headless
    spec:
      clusterIP: None
      selector:
        app: db
      ports:
        - port: 8080
    ```

    - ClusterIP Service: DB 파드끼리의 통신을 가능하게 해준다.

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: nam-cip
    spec:
      type: ClusterIP
      selector:
        app: db
      ports:
        - port: 3306
    ```

    - LoadBalancer Service

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: nam-svc-lb
    spec:
      type: LoadBalancer
      selector:
        app: wp
      ports:
        - port: 80
          targetPort: 80
    ```

2. StatefulSet (MySQL)생성

    Wordpress를 생성하기전에 DB를 띄우는 이유는 DB 마스터 파드의 호스트이름을 미리 알고있어야 하기 때문에 먼저 DB를 띄운다.

    Kubernetes 문서의 MySQL APP 문서를 바탕으로 하지만 내부 설정 값을 찾아 적절한 요소에 값을 교체해준다.(xtrabackup ⇒ DB HA 솔루션(마스터가 무조건 1개)

    특히 환경변수를 워드프레스 database 정보에 맞게 셋팅해준다, 또한 'MYSQL_ALLOW_EMPTY_PASSWORD'을 1로 셋팅해주어야 문서에 설정된 livenessProbe에 조건을 충족시켜줄 수 있다.

    - Statefulset.spec.selector.matchLabels.
    - Statefulset.spec.serviceName (Headless)
    - Statefulset.spec.template.metadata.labels
    - Statefulset.spec.containers.env
    - Statefulset.spec.volumeClaimTemplates (volumeClaimTemplates 에 storageClassName을 rook-ceph-block으로 셋팅)

    ```yaml
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: mysql
    spec:
      selector:
        matchLabels:
          app: db
      serviceName: nam-svc-headless
      replicas: 3
      template:
        metadata:
          labels:
            app: db
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
          - name: clone-mysql
            image: gcr.io/google-samples/xtrabackup:1.0
            command:
            - bash
            - "-c"
            - |
              set -ex
              # Skip the clone if data already exists.
              [[ -d /var/lib/mysql/mysql ]] && exit 0
              # Skip the clone on master (ordinal index 0).
              [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
              ordinal=${BASH_REMATCH[1]}
              [[ $ordinal -eq 0 ]] && exit 0
              # Clone data from previous peer.
              ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
              # Prepare the backup.
              xtrabackup --prepare --target-dir=/var/lib/mysql
            volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
              subPath: mysql
            - name: conf
              mountPath: /etc/mysql/conf.d
          containers:
          - name: mysql
            image: mysql:5.7
            env:
              - name: MYSQL_ALLOW_EMPTY_PASSWORD
                value: "1"
              - name: MYSQL_DATABASE
                value: "wordpress"
              - name: MYSQL_USER
                value: "wp-admin"
              - name: MYSQL_PASSWORD
                value: "1234"
            ports:
            - name: mysql
              containerPort: 3306
            volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
              subPath: mysql
            - name: conf
              mountPath: /etc/mysql/conf.d
            resources:
              requests:
                cpu: 500m
                memory: 1Gi
            livenessProbe:
              exec:
                command: ["mysqladmin", "ping"]
              initialDelaySeconds: 30
              periodSeconds: 10
              timeoutSeconds: 5
            readinessProbe:
              exec:
                # Check we can execute queries over TCP (skip-networking is off).
                command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
              initialDelaySeconds: 5
              periodSeconds: 2
              timeoutSeconds: 1
          - name: xtrabackup
            image: gcr.io/google-samples/xtrabackup:1.0
            ports:
            - name: xtrabackup
              containerPort: 3307
            command:
            - bash
            - "-c"
            - |
              set -ex
              cd /var/lib/mysql

              # Determine binlog position of cloned data, if any.
              if [[ -f xtrabackup_slave_info && "x$(<xtrabackup_slave_info)" != "x" ]]; then
                # XtraBackup already generated a partial "CHANGE MASTER TO" query
                # because we're cloning from an existing slave. (Need to remove the tailing semicolon!)
                cat xtrabackup_slave_info | sed -E 's/;$//g' > change_master_to.sql.in
                # Ignore xtrabackup_binlog_info in this case (it's useless).
                rm -f xtrabackup_slave_info xtrabackup_binlog_info
              elif [[ -f xtrabackup_binlog_info ]]; then
                # We're cloning directly from master. Parse binlog position.
                [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
                rm -f xtrabackup_binlog_info xtrabackup_slave_info
                echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                      MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
              fi

              # Check if we need to complete a clone by starting replication.
              if [[ -f change_master_to.sql.in ]]; then
                echo "Waiting for mysqld to be ready (accepting connections)"
                until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done

                echo "Initializing replication from clone position"
                mysql -h 127.0.0.1 \
                      -e "$(<change_master_to.sql.in), \
                              MASTER_HOST='mysql-0.mysql', \
                              MASTER_USER='root', \
                              MASTER_PASSWORD='', \
                              MASTER_CONNECT_RETRY=10; \
                            START SLAVE;" || exit 1
                # In case of container restart, attempt this at-most-once.
                mv change_master_to.sql.in change_master_to.sql.orig
              fi

              # Start a server to send backups when requested by peers.
              exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
                "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"
            volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
              subPath: mysql
            - name: conf
              mountPath: /etc/mysql/conf.d
            resources:
              requests:
                cpu: 100m
                memory: 100Mi
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
              storage: 10Gi
          storageClassName: rook-ceph-block
    ```

3. Deploymnet에 사용될 pvc 생성

    Deployment가 생성될때 참조할 pvc를 미리 생성 해줘야한다.(storageClass는 csi-cephfs 사용)

    StatefulSet은 volumeClaimTemplates사용하므로 pvc를 따로 생성하지 않아도 template에 의해 pvc가 자동생성되고, 따라서 pv도 자동생성된다.

    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: wp-pvc
    spec:
      accessModes: ["ReadWriteMany"]
      resources:
        requests:
          storage: 1Gi
      storageClassName: csi-cephfs
    ```

4. Deployment (Wordpress)생성

    DB에 접근할때 사용할 계정 정보를 환경변수로 셋팅해준다. 특히 

    'WORDPRESS_DB_HOST' 를 DB 파드의 호스트 이름(파드이름.서비스이름)으로 셋팅해준다

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nam-deploy
    spec:
      replicas: 5
      selector:
        matchLabels:
          app: wp
      strategy:
        type: RollingUpdate
        rollingUpdate:
          maxUnavailable: 2
          maxSurge: 2
      minReadySeconds: 20
      template:
        metadata:
          name: nam-pod-wordpress
          labels:
            app: wp
        spec:
          containers:
            - name: nam-con-wordpress
              image: wordpress:latest
              env:
                - name: WORDPRESS_DB_HOST
                  value: "mysql-0.nam-svc-headless"
                - name: WORDPRESS_DB_USER
                  value: "wp-admin"
                - name: WORDPRESS_DB_PASSWORD
                  value: "1234"
                - name: WORDPRESS_DB_NAME
                  value: "wordpress"
              ports:
                - containerPort: 80
              volumeMounts:
                - name: wp-vol
                  mountPath: /var/www/html
          volumes:
            - name: wp-vol
              persistentVolumeClaim:
                claimName: wp-pvc
    ```

### Result

![Practice2%20Wordpress-DB%20Connect%202f2679430ce4428bb49649c21847e7b1/Untitled.png](Practice2%20Wordpress-DB%20Connect%202f2679430ce4428bb49649c21847e7b1/Untitled.png)

![Practice2%20Wordpress-DB%20Connect%202f2679430ce4428bb49649c21847e7b1/Untitled%201.png](Practice2%20Wordpress-DB%20Connect%202f2679430ce4428bb49649c21847e7b1/Untitled%201.png)

![Practice2%20Wordpress-DB%20Connect%202f2679430ce4428bb49649c21847e7b1/Untitled%202.png](Practice2%20Wordpress-DB%20Connect%202f2679430ce4428bb49649c21847e7b1/Untitled%202.png)

![Practice2%20Wordpress-DB%20Connect%202f2679430ce4428bb49649c21847e7b1/Untitled%203.png](Practice2%20Wordpress-DB%20Connect%202f2679430ce4428bb49649c21847e7b1/Untitled%203.png)

