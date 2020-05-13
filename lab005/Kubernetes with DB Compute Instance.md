# Kubernetes with Oracle DB Compute Instance

This lab show you a seamless integration between an application deployed inside the kubernetes cluster and an oracle database.

Using the REST database interface to provisioning new database containers, a PDB, or even cloning existing one. Allows to use any Oracle Database with multitenant option in a DBaaS fashion in any environment (cloud, on-prem or hybrid)

The complexity of managing the connections to the database is abstracted by the usage of the Oracle Connection Manager so to have a "standard" kubernetes developer experience.

The operator creates a secret containing all the info for the connection, the visibility of which can be limited to the application itself so to achieve isolation of credentials.

A full architecture could be the one presented in this diagram:

![](./images/architecture.png " ")



## Deploy Oracle DB Compute Instance on OCI



1. Create a custom compute instance using Lab4 into the exist private subnet: oke-subnet-quick-...

2. Write down the database information like:

   - Private IP address: 10.0.10.6

   - DB Hostname: *dbserver.sub981952be8.mycluster.oraclevcn.com*

   - CDB service: *ORCL*

   - PDB service: *orclpdb*
   - Database password: *Ora_DB4U*



   

   


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

   - Change the container image from ```DOCKER_REPO/cman:19.3.0.0``` to ```minqiao/cman:19.3.0```. Ad

   - Add env variables PUBLIC_IP and PUBLIC_HOSTNAME to the dynamic value when the pod startup. Please refer the following file content.

   - Add env variables SCAN_NAME and SCAN_IP to the DB hostname without domain(**dbserver**) and IP address(**10.0.10.6**).

   - Add the pod service type: LoadBalancer, so the Public IP can be used by DB  to register.

     

   The file looks like this:

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
                 valueFrom:
                   fieldRef:
                     fieldPath: status.podIP
               - name: PUBLIC_HOSTNAME
                 valueFrom:
                   fieldRef:
                     fieldPath: metadata.name
               - name: SCAN_NAME
                 value: "dbserver"
               - name: SCAN_IP
                 value: "10.0.10.6"            
         imagePullSecrets:
           - name: DOCKER_SECRET
   
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: oracle-db-connection-manager-service
   spec:
     type: LoadBalancer
     ports:
       - port: 1521
         targetPort: 1521
         protocol: TCP
     selector:
       app: oracle-db-connection-manager                                    
   ```

4. To create the deployment and service, enter the following command:

   ```
   $ kubectl apply -f ./examples/cman-deployment.yaml
   ```

5. List the pods.

   ```
   [opc@oke-bastion ~]$ kubectl get pods
   NAME                                           READY   STATUS    RESTARTS   AGE
   oracle-db-connection-manager-d474757ff-rhs98   1/1     Running   0          12s
   ```

6. After the status change to Running, check the log to see if it start succes.

   ```
   $ kubectl logs -f oracle-db-connection-manager-d474757ff-rhs98
   ```

   When you see the following message, the CMAN is ready, press control-c to exist.

   ```
   03-24-2020 02:08:42 UTC :  : cman started sucessfully
   03-24-2020 02:08:42 UTC :  : ################################################
   03-24-2020 02:08:42 UTC :  :  CONNECTION MANAGER IS READY TO USE!            
   03-24-2020 02:08:42 UTC :  : ################################################
   03-24-2020 02:08:42 UTC :  : cman started sucessfully
   ```

   

7. Check the Connection Manager Service with the Extenal-IP address. 

   ```
   $ kubectl get service
   NAME                                           TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)          AGE
   service/kubernetes                             ClusterIP      10.96.0.1      <none>            443/TCP          8d
    service/oracle-db-connection-manager-service   LoadBalancer   10.96.61.86    168.138.220.146   1521:31388/TCP   5m36s
   ```
   
   - Service Name: oracle-db-connection-manager-service
   - External-IP: 168.138.220.146
   - Port: 1521
   

   

## Register Database to the CMAN

1. From bastion host, Install Oracle Instant Client in the bastion if you haven't done this:

```
   $ sudo yum install oracle-release-el7
   $ sudo yum list oracle-instantclient*
   $ sudo yum install oracle-instantclient19.3-basic.x86_64 oracle-instantclient19.3-sqlplus.x86_64 oracle-instantclient19.3-tools.x86_64
```

   

2. Login to the Database as sysdba.

   ```
   $ sqlplus sys/Ora_DB4U@10.0.10.6:1521/ORCL as sysdba
    
   SQL*Plus: Release 19.0.0.0.0 - Production on Tue Mar 24 02:54:38 2020
   Version 19.5.0.0.0

   Copyright (c) 1982, 2019, Oracle.  All rights reserved.

   Last Successful login time: Mon Mar 23 2020 12:05:08 +00:00

   Connected to:
   Oracle Database 19c EE High Perf Release 19.0.0.0.0 - Production
   Version 19.6.0.0.0

   SQL> 
   ```

   

3. Alter the *remote_listener* and register to the CMAN.

   ```
   SQL> alter system set remote_listener='(DESCRIPTION=(ADDRESS_LIST=(ADDRESS=(PROTOCOL=tcp)(HOST=168.138.220.146)(PORT=1521))))' scope=both;

   System altered.

   SQL> alter system register;

   System altered.

   ```

   

4. From bastion host, test the CMAN using the External-IP of the CMAN service.

   ```
   [opc@oke-bastion ~]$ sqlplus system/Ora_DB4U@168.138.220.146:1521/orclpdb

   SQL*Plus: Release 19.0.0.0.0 - Production on Wed Mar 25 02:51:14 2020
   Version 19.5.0.0.0

   Copyright (c) 1982, 2019, Oracle.  All rights reserved.

   Last Successful login time: Wed Mar 25 2020 02:50:12 +00:00

   Connected to:
   Oracle Database 19c EE High Perf Release 19.0.0.0.0 - Production
   Version 19.6.0.0.0

   SQL> 
   ```

   The database is successfully register to the CMAN.

   

##Deploy Oracle Rest Data Services

1. In the bastion host. Make sure you are under the right directory

   ```
   $ cd /home/opc/oracle-db-operator
   ```

2. Modify the configure files for the configmap. First edit the *apex.xml* file.

   ```
   $ vi ./examples/ords/configmaps/apex.xml
   ```

   modify the ```##ORDS_PASSWORD##``` to ```Ora_DB4U```.

   The file looks like:

   ```
   <?xml version="1.0" encoding="UTF-8" standalone="no"?>
   <!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
   <properties>
   <comment>Saved on Fri Aug 23 11:24:16 GMT 2019</comment>
   <entry key="db.password">!Ora_DB4U</entry>
   <entry key="db.username">ORDS_PUBLIC_USER</entry>

   </properties>
   ```

3. Edit the *apex_pu.xml* file.

   ```
   $ vi ./examples/ords/configmaps/apex_pu.xml
   ```

   change all ```##ORDS_PASSWORD##``` to ```Ora_DB4U```.

   The file looks like:

   ```
   <?xml version="1.0" encoding="UTF-8" standalone="no"?>
   <!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
   <properties>
   <comment>Saved on Mon Aug 26 11:22:54 GMT 2019</comment>
   <entry key="db.cdb.adminUser">C##DBAPI_CDB_ADMIN as SYSDBA</entry>
   <entry key="db.cdb.adminUser.password">!Ora_DB4U</entry>
   <entry key="db.password">!Ora_DB4U</entry>
   <entry key="db.username">ORDS_PUBLIC_USER</entry>
   </properties>
   ```

4. Edit the *defaults.xml* file.

   ```
   vi ./examples/ords/configmaps/defaults.xml
   ```

   - change ```##ORDS_PASSWORD##``` to ```Ora_DB4U```.
   - change ```##DATABASE_CDB_SERVICE_NAME##``` to ```ORCL```.
   - change db hostname from  ```oracle-db-enterprise``` to ```dbserver.sub981952be8.mycluster.oraclevcn.com```

   The file looks like:

   ```
   <?xml version="1.0" encoding="UTF-8" standalone="no"?>
   <!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
   <properties>
   <comment>Saved on Mon Aug 26 11:24:32 GMT 2019</comment>
   <entry key="database.api.admin.enabled">true</entry>
   <entry key="database.api.enabled">true</entry>
   <entry key="db.cdb.adminUser">C##DBAPI_CDB_ADMIN as SYSDBA</entry>
   <entry key="db.cdb.adminUser.password">!Ora_DB4U</entry>
   <entry key="db.hostname">dbserver.sub981952be8.mycluster.oraclevcn.com</entry>
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

   Copy the credentials content which generate in Appendix3 using password "WElcome_123#".

   The file looks like the following, You can copy and update the content directly to your credentials file.

   ```
   admin;{SSHA-512}I9yp9gOvNtML/DfGkz2XHqb/lXbmeryleCLRGPdoDG1r67dKDPxPosgeG57Y08iDbT7JMnmxXNxz3QTAZzgu95fKBqx2P/wx;SQL Administrator,System Administrator
   ```

6. Run the following command to create configmap

   ```
   $ kubectl create configmap oracle-db-ords-config --from-file=examples/ords/configmaps/
   ```

7. Edit ords-persistent-storage.yaml file

   ```
   $ vi ./examples/ords/ords-persistent-storage.yaml
   ```

   Change the *storageClassName* to *"oci"*, the file looks like:

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
   - change oracle host from ```oracle-db-enterprise``` to ```dbserver.sub981952be8.mycluster.oraclevcn.com```
   - change Oracle Service from ```ORCLCDB``` to ```ORCL```
   - change Oracle PWD from ```##DB_PASSWORD##``` to ```Ora_DB4U```
   - change ORDS PWD from ```##ORDS_PASSWORD##``` to ```Ora_DB4U```
   - change Oracle Base from ```/opt/oracle``` to ```/u01/app/oracle```
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
                 value: dbserver.sub981952be8.mycluster.oraclevcn.com
               - name: ORACLE_SERVICE
                 value: ORCL
               - name: ORACLE_PWD
                 value: Ora_DB4U
               - name: ORDS_PWD
                 value: Ora_DB4U
               - name: ORACLE_BASE
                 value: /u01/app/oracle
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
    $ kubectl get pods
    NAME                                           READY   STATUS             RESTARTS   AGE
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

15. From the bastion host, Connect to Database as sysdba:

    ```
    [opc@oke-bastion oracle-db-operator]$ sqlplus sys/Ora_DB4U@10.0.10.6:1521/ORCL as sysdba
    
    SQL*Plus: Release 19.0.0.0.0 - Production on Mon Mar 23 11:52:48 2020
    Version 19.5.0.0.0
    
    Copyright (c) 1982, 2019, Oracle.  All rights reserved.
    
    Last Successful login time: Mon Mar 23 2020 10:30:50 +00:00
    
    Connected to:
    Oracle Database 19c EE High Perf Release 19.0.0.0.0 - Production
    Version 19.5.0.0.0
    
    SQL> 
    ```

    ​    

17. Copy and run the following SQL Statement to Enable ORDS Database API.

    ```
    CREATE USER C##DBAPI_CDB_ADMIN IDENTIFIED BY Ora_DB4U;
    GRANT SYSDBA TO C##DBAPI_CDB_ADMIN CONTAINER = ALL;
    ```
    
    

22. Exit SQLPLUS. Log into the ords pod(because there is no publice ip for the ords service), test the ORDS databse API using the following command:

    ```
    [opc@oke-bastion oracle-db-operator]$ kubectl exec -it oracle-db-ords-557d787956-wkmhq bash
    [oracle@oracle-db-ords-557d787956-wkmhq ~]$ curl -u admin:WElcome_123# http://oracle-db-ords:8888/ords/_/db-api/stable/database/pdbs/orclpdb/status
    ```

    The result like this:
    
    ```
    {"inst_id":1,"con_id":3,"name":"ORCLPDB","open_mode":"READ WRITE","restricted":"NO","links":[{"rel":"collection","href":"http://oracle-db-ords:8888/ords/_/db-api/stable/database/pdbs/orclpdb/"}]}
    ```
    
    The ORDS Database API is ready. If you want to know more about the Rest API for Oracle Database Please refer to the document: https://docs.oracle.com/en/database/oracle/oracle-database/19/dbrst/index.html

 

## Deploy Oracle Database Operator

1. In the bastion host. Make sure you are under the right directory

   ```
   $ cd /home/opc/oracle-db-operator
   ```

   

2. Edit *ords-credentials.yaml* file. 

   ```
   $ vi ./examples/ords/ords-credentials.yaml
   ```

   Go to the URL: http://www.base64encode.org, encode the username as **admin**, password as **WElcome_123#**. Update the username and password entry in the file using the base64 encode result. The file looks like:
   
   ```
   kind: Secret
      apiVersion: v1
   metadata:
        name: oracle-ords-credentials
        namespace: default
      data:
        username: YWRtaW4=
        password: V0VsY29tZV8xMjMj
      type: Opaque
   ```



3. Deploy the credentials:

   ```
   $ kubectl apply -f examples/ords/ords-credentials.yaml
   ```

    

4. Modify the ```operator-k8s.yaml file```

   ```
   $ vi ./manifest/operator-k8s.yaml
   ```

   - change ```##DOCKER_REGISTY##``` to ```minqiao```
   - change port from ```8080``` to ```8888``` 
   - change DB_FILENAME_CONVERSION_PATTERN  to ```"NONE"``` because we already enable the OMF.

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
             value: "oracle-db-connection-manager-service"
           - name: OCM_SERVICE_PORT
             value: "1521"
           - name: DB_FILENAME_CONVERSION_PATTERN
             value: "NONE"
           imagePullPolicy: Always
         imagePullSecrets:
           - name: ##DOCKER_SECRET##
   ```

   

5. Deploy the Oracle Database Operator

   ```
   $ kubectl apply -f manifest/operator-k8s.yaml
   ```

6. List the pods

   ```
   $ kubectl get pods
   NAME                                           READY   STATUS             RESTARTS   AGE
   oracle-db-enterprise-85bd744d59-fg7ml          1/1     Running            0          4h28m
   oracle-db-operator-7bd7d5bfbc-w2w87            1/1     Running            0          111s
   oracle-db-ords-f666954f5-92ls4                 1/1     Running            0          149m
   ```

7. Check the log, 

   ```
   $ kubectl logs -f oracle-db-operator-7bd7d5bfbc-w2w87
   ```

   You may see the message like the following, Press **control-c** to exist.

   ```
   2020-03-21 06:28:52 INFO  Entrypoint:236 - 
   Operator has started in version 0.0.1-SNAPSHOT.
   ```

     

## Provision a new PDB through operator

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

   

3. Deploy the service to create a new pdb.

   ```
   $ kubectl apply -f examples/cr.yaml
   ```

   Wait some time, you will get the message:

   ```
   oracleservice.com.oracle/mypdb created
   ```

   

4. From the bastion host, log into the database as sysdba:

   ```
   $ sqlplus sys/Ora_DB4U@10.0.10.6:1521/ORCL as sysdba

   SQL*Plus: Release 19.0.0.0.0 - Production on Thu Mar 26 03:21:42 2020
   Version 19.5.0.0.0

   Copyright (c) 1982, 2019, Oracle.  All rights reserved.
   
   Connected to:
   Oracle Database 19c EE High Perf Release 19.0.0.0.0 - Production
   Version 19.6.0.0.0
   
   SQL>
   ```




5. You can see the a PDB named **ORACLE_MYPDB** is created.

   ```
   SQL> show pdbs

       CON_ID CON_NAME			  OPEN MODE  RESTRICTED
   ---------- ------------------------------ ---------- ----------
   	 2 PDB$SEED			  READ ONLY  NO
   	 3 ORCLPDB 			  READ WRITE NO
   	 4 ORACLE_MYPDB 		  READ WRITE NO
   
   ```

6. (**Optional**) You can drop the PDB you created before.

   ```
   $ kubectl delete -f examples/cr.yaml
   ```

   

