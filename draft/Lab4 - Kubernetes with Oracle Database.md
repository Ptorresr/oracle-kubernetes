# Kubernetes with Oracle Database

This lab show you a seamless integration between an application deployed inside the kubernetes cluster and an oracle database.

Using the REST database interface to provisioning new database containers, a PDB, or even cloning existing one. Allows to use any Oracle Database with multitenant option in a DBaaS fashion in any environment (cloud, on-prem or hybrid)

The complexity of managing the connections to the database is abstracted by the usage of the Oracle Connection Manager so to have a "standard" kubernetes developer experience.

The operator creates a secret containing all the info for the connection, the visibility of which can be limited to the application itself so to achieve isolation of credentials.

A full architecture could be the one presented in this diagram:

![architecture](img/architecture.png)

## Deploy Oracle Connection Manager

Deploy the Oracle Connection Manager, You will use a CMAN image in the docker hub that create in advance. You can refer the Appendix to build your own version of CMAN v19.3 image.

1. In the bastion host, Install git

   ```
   $ sudo yum install git
   ```

2. You will clone some setup scripts from git.

   ```
   $ git clone https://github.com/minqiaowang/oracle-db-operator.git
   ```

3. Edit the cman deployment scripts.

   ```
   $ cd oracle-db-operator
   $ vi ./examples/cman-deployment.yaml
   ```

4. Change the container image from ```DOCKER_REPO/cman:19.3.0.0``` to ```minqiao/cman:19.3.0```. Add the environment variables:

   - Domain
   - PUBLIC_IP
   - PUBLIC_HOSTNAME

   ```
   apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
   kind: Deployment
   metadata:
     name: oracle-db-connection-manager
   spec:
     replicas: 1
     minReadySeconds: 30
     selector:
       matchLabels:
         app: oracle-db-connection-manager
     template:
       metadata:
         labels:
           app: oracle-db-connection-manager
       spec:
         containers:
           - name: oracle-db-connection-manager
             image: minqiao/cman:19.3.0
             ports:
               - containerPort: 1521
             livenessProbe:
               tcpSocket:
                 port: 1521
               initialDelaySeconds: 60
               periodSeconds: 30
             env:
               - name: DOMAIN
                 value: "default.svc.cluster.local"
               - name: PUBLIC_IP
                 value: "10.96.254.118"
               - name: PUBLIC_HOSTNAME
                 value: "oracle-db-connection-manager"              
         imagePullSecrets:
           - name: DOCKER_SECRET
   
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: oracle-db-connection-manager-service
   spec:
     ports:
       - port: 1521
         targetPort: 1521
         protocol: TCP
     selector:
       app: oracle-db-connection-manager                                    
   ```

5. To create the deployment and service, enter the following command:

   ```
   $ kubectl apply -f ./examples/cman-deployment.yaml
   ```

6. List the pods.

   ```
   [opc@oke-bastion ~]$ kubectl get pod
   NAME                                           READY   STATUS    RESTARTS   AGE
   oracle-db-connection-manager-d474757ff-rhs98   1/1     Running   0          12s
   ```

7. Check the log to see if it start succes.

   ```
   $ kubectl logs -f oracle-db-connection-manager-d474757ff-rhs98
   ```

8. You can log into the pod, check the status

   ```
   [opc@oke-bastion ~]$ kubectl exec -it oracle-db-connection-manager-d474757ff-rhs98 bash
   [oracle@oracle-db-connection-manager-d474757ff-rhs98 ~]$ 
   ```

9. sadf

   

## Deploy Oracle Database

1. In the bastion host. Make sure you are under the right directory

   ```
   $ cd /home/opc/oracle-db-operator
   ```

2. Modify the configure files for the configmap. First edit the *init.ora* file, modify all the *<ORACLE_BASE>* to */opt/oracle*.

   ```
   $ vi ./examples/database/configmaps/init.ora
   ```

   It's looks like the following, save the file.

   ```
   db_name='ORCL'
   memory_target=1G
   processes = 150
   audit_file_dest='/opt/oracle/admin/orcl/adump'
   audit_trail ='db'v:
   db_block_size=8192
   db_domain=''
   db_recovery_file_dest='/opt/oracle/fast_recovery_area'
   db_recovery_file_dest_size=2G
   diagnostic_dest='/opt/oracle'
   dispatchers='(PROTOCOL=TCP) (SERVICE=ORCLXDB)'
   open_cursors=300
   remote_login_passwordfile='EXCLUSIVE'
   undo_tablespace='UNDOTBS1'
   # You may want to ensure that control files are created on separate physical
   # devices
   control_files = (ora_control1, ora_control2)
   compatible ='11.2.0'
   
   REMOTE_LISTENER=listener_cman
   DISPATCHERS="(PROTOCOL=tcp)(MULTIPLEX=on)"
   ```

3. Edit the *init.ora* file, modify the *\##ORDS_PASSWORD* to *Welcome_123#*.

   ```
   vi ./examples/database/configmaps/init.sql
   ```

   - change ```##ORDS_PASSWORD``` to ```Welcome_123#```.
   - change remote_listener from ```oracle-db-connection-manager-servic:1521``` to ```listener_cman```

   It's looks like the following, save the file.

   ```
   -- ORDS USER
   CREATE USER C##DBAPI_CDB_ADMIN IDENTIFIED BY "Welcome_123#";
   GRANT SYSDBA TO C##DBAPI_CDB_ADMIN CONTAINER = ALL;
   GRANT connect  TO C##DBAPI_CDB_ADMIN CONTAINER = ALL;
   
   -- CONNECTION TO CONNECTION MANAGER
   alter system  set remote_listener='listener_cman' scope=both sid='*';
   alter system register;
   ```

4. Edit the *tnsnames.ora* file

   ```
   $ vi ./examples/database/configmaps/tnsnames.ora
   ```

   - change ORCLCDB to ORCL
   - change ORCLPDB1 to PDB1

   The file looks like this:

   ```
   ORCL=127.0.0.1:1521/ORCL
   PDB1=
   (DESCRIPTION =
     (ADDRESS = (PROTOCOL = TCP)(HOST = 0.0.0.0)(PORT = 1521))
     (CONNECT_DATA =
       (SERVER = DEDICATED)
       (SERVICE_NAME = PDB1)
     )
   )
   
   listener_cman=(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=tcp)(HOST=oracle-db-connection-manager-service)(PORT=1521))))
   ```

5. Create the configmap using the command:

   ```
   $ kubectl create configmap oracle-db-config --from-file=./examples/database/configmaps/
   ```

6. Edit the *oracle-db-deployment.yaml* file, 

   ```
   $ vi ./examples/database/oracle-db-deployment.yaml
   ```

   - change ```##DOCKER_REGISTRY##/oracle-db:19.3.0.0 ``` to ```minqiao/database:19.3.0-ee```
   - change SID  ```value: "ORCLCDB"``` to ```value: "ORCL"```
   - change PDB ```value: "ORCLPDB1"``` to ```value: "PDB1"```
   - change PWD ```##ORACLE_SYS_PASSWORD##``` to ```Welcome_123#```
   - change service name from ```oracle-db-enterprise-1``` to ```oracle-db-enterprise```

   The file looks like:

   ```
   apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
   kind: Deployment
   metadata:
     name: oracle-db-enterprise
   spec:
     replicas: 1
     minReadySeconds: 30
     selector:
       matchLabels:
         app: oracle-db-enterprise
     template:
       metadata:
         labels:
           app: oracle-db-enterprise
       spec:
         hostname: oracle-db-enterprise
         containers:      
         - name: oracle-db-enterprise
           image: minqiao/database:19.3.0-ee
           env:
           - name: ORACLE_SID
             value: "ORCL"
           - name: ORACLE_PDB
             value: "PDB1"
           - name: ORACLE_PWD
             value: "Welcome_123#"
           volumeMounts:
           - name: oracle-db-config
             mountPath: /opt/oracle/scripts/setup
           ports:
           - containerPort: 1521
           livenessProbe:
             tcpSocket:
               port: 1521
             initialDelaySeconds: 300
             periodSeconds: 30
         imagePullSecrets:
           - name: ##DOCKER_SECRET##
         volumes:
           - name: oracle-db-config
             configMap:
               name: oracle-db-config
   
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: oracle-db-enterprise
   spec:
     ports:
     - port: 1521
       targetPort: 1521
       protocol: TCP
     selector:
       app: oracle-db-enterprise
   ```

   

7. Deploy the Oracle Database

   ```
   $ kubectl apply -f examples/database/oracle-db-deployment.yaml
   ```

8. List the pods

   ```
   $ kubectl get pod
   NAME                                           READY   STATUS              RESTARTS   AGE
   oracle-db-connection-manager-d474757ff-rhs98   1/1     Running             10         38m
   oracle-db-enterprise-8d67f7cbb-c78c2           0/1     ContainerCreating   0          17s
   ```

9. It's need some time to download the container image for the first time. List the pods again, when the STATUS change to Running, you can check the log.

   ```
   $ kubectl logs -f oracle-db-enterprise-8d67f7cbb-c78c2
   ```

   The database setup will take about 15 minutes, if you encounter the *error: unexpected EOF*, try to re-enter the log check.

10. When the database creationg is complete, you may see **The Database is Ready to Use**.  Ignore the ERROR: ORA-01920. Press **control-c** to continue. 

    ```
    The Oracle base remains unchanged with value /opt/oracle
    #########################
    DATABASE IS READY TO USE!
    #########################
    
    Executing user defined scripts
    /opt/oracle/runUserScripts.sh: running /opt/oracle/scripts/startup/init.sql
    CREATE USER C##DBAPI_CDB_ADMIN IDENTIFIED BY "Welcome_123#"
                *
    ERROR at line 1:
    ORA-01920: user name 'C##DBAPI_CDB_ADMIN' conflicts with another user or role
    name
    
    
    
    Grant succeeded.
    
    
    Grant succeeded.
    
    
    System altered.
    
    
    System altered.
    
    
    
    DONE: Executing user defined scripts
    ```

11. Log into the pod.

    ```
    $ [opc@oke-bastion oracle-db-operator]$ kubectl exec -it oracle-db-enterprise-6c4988c887-lnttf bash
    [oracle@oracle-db-enterprise ~]$ 
    ```

12. Test the database is ready.

    ```
    [oracle@oracle-db-enterprise ~]$ lsnrctl status
    
    LSNRCTL for Linux: Version 19.0.0.0.0 - Production on 20-MAR-2020 04:26:02
    
    Copyright (c) 1991, 2019, Oracle.  All rights reserved.
    
    Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=IPC)(KEY=EXTPROC1)))
    STATUS of the LISTENER
    ------------------------
    Alias                     LISTENER
    Version                   TNSLSNR for Linux: Version 19.0.0.0.0 - Production
    Start Date                20-MAR-2020 04:05:30
    Uptime                    0 days 0 hr. 20 min. 32 sec
    Trace Level               off
    Security                  ON: Local OS Authentication
    SNMP                      OFF
    Listener Parameter File   /opt/oracle/product/19c/dbhome_1/network/admin/listener.ora
    Listener Log File         /opt/oracle/diag/tnslsnr/oracle-db-enterprise/listener/alert/log.xml
    Listening Endpoints Summary...
      (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1)))
      (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=0.0.0.0)(PORT=1521)))
      (DESCRIPTION=(ADDRESS=(PROTOCOL=tcps)(HOST=oracle-db-enterprise)(PORT=5500))(Security=(my_wallet_directory=/opt/oracle/admin/ORCL/xdb_wallet))(Presentation=HTTP)(Session=RAW))
    Services Summary...
    Service "ORCL" has 1 instance(s).
      Instance "ORCL", status READY, has 1 handler(s) for this service...
    Service "ORCLXDB" has 1 instance(s).
      Instance "ORCL", status READY, has 1 handler(s) for this service...
    Service "a142a3352f9d0a3ce0530901f40ad3e3" has 1 instance(s).
      Instance "ORCL", status READY, has 1 handler(s) for this service...
    Service "pdb1" has 1 instance(s).
      Instance "ORCL", status READY, has 1 handler(s) for this service...
    The command completed successfully
    [oracle@oracle-db-enterprise ~]$ sqlplus system/Welcome_123#@oracle-db-enterprise:1521/PDB1
    
    SQL*Plus: Release 19.0.0.0.0 - Production on Fri Mar 20 04:26:31 2020
    Version 19.3.0.0.0
    
    Copyright (c) 1982, 2019, Oracle.  All rights reserved.
    
    
    Connected to:
    Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
    Version 19.3.0.0.0
    
    SQL> 
    ```

    



##Deploy Oracle Rest Data Services

1. In the bastion host. Make sure you are under the right directory

   ```
   $ cd /home/opc/oracle-db-operator
   ```

2. Modify the configure files for the configmap. First edit the *apex.xml* file.

   ```
   $ vi ./examples/ords/configmaps/apex.xml
   ```

   modify the ```##ORDS_PASSWORD##``` to ```Welcome_123#```.

   The file looks like:

   ```
   <?xml version="1.0" encoding="UTF-8" standalone="no"?>
   <!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
   <properties>
   <comment>Saved on Fri Aug 23 11:24:16 GMT 2019</comment>
   <entry key="db.password">!Welcome_123#</entry>
   <entry key="db.username">ORDS_PUBLIC_USER</entry>
   
   </properties>
   ```

3. Edit the *apex_pu.xml* file.

   ```
   $ vi ./examples/ords/configmaps/apex_pu.xml
   ```

   change all ```##ORDS_PASSWORD##``` to ```Welcome_123#```.

   The file looks like:

   ```
   <?xml version="1.0" encoding="UTF-8" standalone="no"?>
   <!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
   <properties>
   <comment>Saved on Mon Aug 26 11:22:54 GMT 2019</comment>
   <entry key="db.cdb.adminUser">C##DBAPI_CDB_ADMIN as SYSDBA</entry>
   <entry key="db.cdb.adminUser.password">!Welcome_123#</entry>
   <entry key="db.password">!Welcome_123#</entry>
   <entry key="db.username">ORDS_PUBLIC_USER</entry>
   </properties>
   ```

4. Edit the *defaults.xml* file.

   ```
   vi ./examples/ords/configmaps/defaults.xml
   ```

   - change ```##ORDS_PASSWORD##``` to ```Welcome_123#```.
   - change ```##DATABASE_CDB_SERVICE_NAME##``` to ```ORCL```.

   The file looks like:

   ```
   <?xml version="1.0" encoding="UTF-8" standalone="no"?>
   <!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
   <properties>
   <comment>Saved on Mon Aug 26 11:24:32 GMT 2019</comment>
   <entry key="database.api.admin.enabled">true</entry>
   <entry key="database.api.enabled">true</entry>
   <entry key="db.cdb.adminUser">C##DBAPI_CDB_ADMIN as SYSDBA</entry>
   <entry key="db.cdb.adminUser.password">!Welcome_123#</entry>
   <entry key="db.hostname">oracle-db-enterprise</entry>
   <entry key="db.port">1521</entry>
   <entry key="db.servicename">ORCL</entry>
   <entry key="jdbc.DriverType">thin</entry>
   <entry key="jdbc.InactivityTimeout">1800</entry>
   <entry key="jdbc.InitialLimit">3</entry>
   <entry key="jdbc.MaxConnectionReuseCount">1000</entry>
   <entry key="jdbc.MaxLimit">10</entry>
   <entry key="jdbc.MaxStatementsLimit">10</entry>
   <entry key="jdbc.MinLimit">1</entry>
   <entry key="jdbc.auth.enabled">true</entry>
   <entry key="jdbc.statementTimeout">900</entry>
   <entry key="log.logging">false</entry>
   <entry key="log.maxEntries">50</entry>
   <entry key="debug.debugger">true</entry>
   <entry key="debug.printDebugToScreen">true</entry>
   <entry key="misc.compress"/>
   <entry key="misc.defaultPage">apex</entry>
   <entry key="security.disableDefaultExclusionList">false</entry>
   <entry key="security.maxEntries">2000</entry>
   <entry key="security.validationFunctionType">plsql</entry>
   </properties>
   ```

5. Edit the *credentials* file.

   ```
   vi ./examples/ords/configmaps/credentials
   ```

   Copy the credentials content generate in Append4 using password "Welcome_123#".

   The file looks like:

   ```
   admin;{SSHA-512}6gdCe3LYEBl7KLozs6ERYctHGpj5pp/xDCm6nL5gFlTL6rkZNa2cZEJhiNgG/ODFEC28yfRdwectwnbukSzkpkybDHBpatyT;SQL Administrator,System Administrator
   ```

6. Run the following command to create configmap

   ```
   $ kubectl create configmap oracle-db-ords-config --from-file=examples/ords/configmaps/
   ```

7. Edit ords-persistent-storage.yaml file

   ```
   $ vi ./examples/ords/ords-persistent-storage.yaml
   ```

   Change the *storageClassName* to *oci*, the file looks like:

   ```
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: oracleclaim-ords-config
   spec:
     storageClassName: "oci"
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 1Gi
   ```

8. Deploy the Persistent Volume.

   ```
   $ kubectl apply -f examples/ords/ords-persistent-storage.yaml
   ```

9. Edit *ords-deployment.yaml* file.

   ```
   $ vi ./examples/ords/ords-deployment.yaml
   ```

   - change image from ```##DOCKER_REGISTRY##/restdataservices:19.2.0.2``` to ``` minqiao/restdataservices:19.2.0```
   - change Oracle Service from ```ORCLCDB``` to ```ORCL```
   - change Oracle PWD from ```##DB_PASSWORD##``` to ```Welcome_123#```
   - change ORDS PWD from ```##ORDS_PASSWORD##``` to ```Welcome_123#```
   - change port from ```8080``` to ```8888```

   The file looks like:

   ```
   apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
   kind: Deployment
   metadata:
     name: oracle-db-ords
   spec:
     replicas: 1
     minReadySeconds: 30
     selector:
       matchLabels:
         app: oracle-db-ords
     template:
       metadata:
         labels:
           app: oracle-db-ords
       spec:
         containers:
           - name: oracle-db-ords
             image: minqiao/restdataservices:19.2.0
             ports:
               - containerPort: 8888
             livenessProbe:
               tcpSocket:
                 port: 8888
               initialDelaySeconds: 60
               periodSeconds: 30
             env:
               - name: ORACLE_HOST
                 value: oracle-db-enterprise
               - name: ORACLE_SERVICE
                 value: ORCL
               - name: ORACLE_PWD
                 value: Welcome_123#
               - name: ORDS_PWD
                 value: Welcome_123#
               - name: ORACLE_BASE
                 value: /opt/oracle/
             volumeMounts:
               - name: oracle-db-ords-config-persistent
                 mountPath: "/opt/oracle/ords/config/ords"
   
         initContainers:
           - name: setup-configs
             command: ["sh", "-c", "cp /conf/apex_pu.xml /apex_pu.xml && mkdir -p /opt/oracle/ords/config/ords/conf/ && cp /conf/apex_pu.xml /opt/oracle/ords/config/ords/conf/apex_pu.xml && cp /conf/apex.xml /opt/oracle/ords/config/ords/conf/apex.xml && cp /conf/credentials /opt/oracle/ords/config/ords/credentials && cp /conf/defaults.xml /opt/oracle/ords/config/ords/defaults.xml && chown -R 54321:54321 /opt/oracle/ords/config/ords"]
             image: busybox
             volumeMounts:
               - name: oracle-db-ords-config
                 mountPath: "/conf"
               - name: oracle-db-ords-config-persistent
                 mountPath: "/opt/oracle/ords/config/ords"
         imagePullSecrets:
           - name: ##DOCKER_SECRET##
   
         volumes:
           - name: oracle-db-ords-config
             configMap:
               name: oracle-db-ords-config
           - name: oracle-db-ords-config-persistent
             persistentVolumeClaim:
               claimName: oracleclaim-ords-config
   
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: oracle-db-ords
   spec:
     ports:
       - port: 8888
         targetPort: 8888
         protocol: TCP
     selector:
       app: oracle-db-ords
   ```

10. Deploy the ORDS.

    ```
    $ kubectl apply -f examples/ords/ords-deployment.yaml
    ```

11. List the pods.

    ```
    $ kubectl get pod
    NAME                                           READY   STATUS             RESTARTS   AGE
    oracle-db-enterprise-bc7dc67cd-g45mc           1/1     Running            0          171m
    oracle-db-ords-f666954f5-cq9mk                 1/1     Running            0          115s
    ```

12. Check the log. 

    ```
    $ kubectl logs -f oracle-db-ords-f666954f5-cq9mk
    ```

    When you see the following message, the ORDS is ready, press **control-c** to exist.

    ```
    2020-03-21 04:02:25.668:INFO:oejsh.ContextHandler:main: Started o.e.j.s.ServletContextHandler@1d057a39{/ords,null,AVAILABLE}
    2020-03-21 04:02:25.673:INFO:oejsh.ContextHandler:main: Started o.e.j.s.h.ContextHandler@464bee09{/,null,AVAILABLE}
    2020-03-21 04:02:25.673:INFO:oejsh.ContextHandler:main: Started o.e.j.s.h.ContextHandler@f6c48ac{/i,null,AVAILABLE}
    2020-03-21 04:02:25.683:INFO:oejs.RequestLogWriter:main: Opened /tmp/ords_log/ords_2020_03_21.log
    2020-03-21 04:02:25.705:INFO:oejs.AbstractConnector:main: Started ServerConnector@20fa2d13{HTTP/1.1,[http/1.1, h2c]}{0.0.0.0:8888}
    2020-03-21 04:02:25.710:INFO:oejs.Server:main: Started @9251ms
    ```

    

13. Edit *ords-credentials.yaml* file. Use base64 encode the username and password.

    ```
    $ echo admin|base64
    YWRtaW4K
    $ echo Welcome_123#|base64
    V2VsY29tZV8xMjMjCg==
    $ vi ./examples/ords/ords-credentials.yaml
    ```

    Change the username and password using the base64 result. The file looks like:

    ```
    ---
    kind: Secret
    apiVersion: v1
    metadata:
      name: oracle-ords-credentials
      namespace: default
    data:
      username: YWRtaW4K
      password: V2VsY29tZV8xMjMjCg==
    type: Opaque
    ```

14. Deploy the credentials:

    ```
    $ kubectl apply -f examples/ords/ords-credentials.yaml
    ```

    

15. sadf

    

    

    

## Deploy Oracle Database Operator

1. In the bastion host. Make sure you are under the right directory

   ```
   $ cd /home/opc/oracle-db-operator
   ```

2. Modify the ```operator-k8s.yaml file```

   ```
   $ vi ./manifest/operator-k8s.yaml
   ```

   - change ```##DOCKER_REGISTY##``` to ```minqiao```
   - change ```##PDBNAME##``` to ```mypdb```.
   - change port from ```8080``` to ```8888```
   - change ```oracle-db-connection-manager-service``` to ```oracle-db-enterprise``` (Currently my CMAN not work in kubernetes)

   The file looks like:

   ```
   # for creating these resources it requires the user to be logged in as system admin
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: oracle-db-operator
   ---
   apiVersion: rbac.authorization.k8s.io/v1beta1
   kind: ClusterRoleBinding
   metadata:
     name: oracle-db-operator-edit-resources
   roleRef:
     kind: ClusterRole
     name: cluster-admin
     apiGroup: ""
   subjects:
     - kind: ServiceAccount
       name: oracle-db-operator
       namespace: default
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: oracle-db-operator
     labels: &default-labels
       app.kubernetes.io/name: oracle-db-operator
       app.kubernetes.io/version: v0.0.1-v1alpha1
   spec:
     replicas: 1
     selector:
       matchLabels: *default-labels
     strategy:
       type: Recreate
     template:
       metadata:
         labels: *default-labels
       spec:
         serviceAccountName: oracle-db-operator
         containers:
         - name: oracle-db-operator
           image: minqiao/oracle-db-operator:1.0.4
           env:
           - name: CRD
             value: "true"
           - name: ORDS_HOST
             value: "oracle-db-ords"
           - name: ORDS_BASEPATH
             value: "/ords"
           - name: ORDS_PORT
             value: "8888"
           - name: ORDS_PROTOCOL
             value: "http"
           - name: ORDS_CREDENTIAL_SECRET_NAME
             value: "oracle-ords-credentials"
           - name: OCM_SERVICE_NAME
             value: "oracle-db-enterprise"
           - name: OCM_SERVICE_PORT
             value: "1521"
           - name: DB_FILENAME_CONVERSION_PATTERN
             value: "('/opt/oracle/oradata/ORCLCDB/pdbseed/','/opt/oracle/oradata/ORCLCDB/mypdb/')"
           imagePullPolicy: Always
         imagePullSecrets:
           - name: ##DOCKER_SECRET##
   ```

   

3. Deploy the Oracle Database Operator

   ```
   $ kubectl apply -f manifest/operator-k8s.yaml
   ```

4. List the pod

   ```
   $ kubectl get pod
   NAME                                           READY   STATUS             RESTARTS   AGE
   oracle-db-enterprise-85bd744d59-fg7ml          1/1     Running            0          4h28m
   oracle-db-operator-7bd7d5bfbc-w2w87            1/1     Running            0          111s
   oracle-db-ords-f666954f5-92ls4                 1/1     Running            0          149m
   ```

5. check the log, 

   ```
   $ kubectl logs -f oracle-db-operator-7bd7d5bfbc-w2w87
   ```

   You may see the message like the following, Press **control-c** to exist.

   ```
   2020-03-21 06:28:52 INFO  Entrypoint:236 - 
   Operator has started in version 0.0.1-SNAPSHOT.
   ```

   

6. sadf

   



## Provision your database

1. In the bastion host. Make sure you are under the right directory

   ```
   $ cd /home/opc/oracle-db-operator
   ```

2. Modify the *cr.yaml* file.

   ```
   $ vi ./examples/cr.yaml
   ```

   Change name from ```my-db``` to ```mypdb``` The file looks like:

   ```
   apiVersion: com.oracle/v1
   kind: OracleService
   metadata:
     name: mypdb
   spec:
     storage: 1000000000
     tempStorage: 10000000
   ```

   

3. asdf

4. sdaf

5. sadf

6. asdf

7. asdf

8. asdf





















