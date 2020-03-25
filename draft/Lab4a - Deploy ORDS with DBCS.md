# Deploy ORDS with DBCS

This lab show you a seamless integration between an application deployed inside the kubernetes cluster and an oracle database.

Using the REST database interface to provisioning new database containers, a PDB, or even cloning existing one. Allows to use any Oracle Database with multitenant option in a DBaaS fashion in any environment (cloud, on-prem or hybrid)

The complexity of managing the connections to the database is abstracted by the usage of the Oracle Connection Manager so to have a "standard" kubernetes developer experience.

The operator creates a secret containing all the info for the connection, the visibility of which can be limited to the application itself so to achieve isolation of credentials.

A full architecture could be the one presented in this diagram:

![architecture](img/architecture.png)



## Deploy Oracle DBCS on OCI

1. ------In the OCI Console, Create a new private subnet 10.0.30.0/24 in the same VCN of the kubernetes cluster, with new security list, add ingress rule to open port 22, 1521.

   
   
2. ------Provisioning a DBCS named ORCL with PDB named pdb1.

   
   
   Write down the DBCS private ip address like *10.0.30.2*

   DB Hostname: *dbhost.dbprivatesubnet.mycluster.oraclevcn.com*

   CDB service: *ORCL_nrt1b7.dbprivatesubnet.mycluster.oraclevcn.com*
   
   PDB service: *pdb1.dbprivatesubnet.mycluster.oraclevcn.com*
   
   

1. Create a DBCS into the exist private subnet: oke-subnet-quick-mycluster...

2. Write down the DBCS private ip address like *10.0.10.5*

   DB Hostname: *dbcs.sub981952be8.mycluster.oraclevcn.com*

   CDB service: *ORCL_nrt1dz.sub981952be8.mycluster.oraclevcn.com*

   PDB service: *pdb1.sub981952be8.mycluster.oraclevcn.com*

3. jk

   

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
   - Change PUBLIC_IP and PUBLIC_HOSTNAME to the dynamic value when the pod startup.
   - ------ Change SCAN_NAME and SCAN_IP to the DBCS hostname and IP address.
   - add the pod service type: LoadBalancer, so the DBCS can be used to register.

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
                 value: "dbcs"
               - name: SCAN_IP
                 value: "10.0.10.5"            
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

6. Check the log to see if it start succes.

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

   - Service Name: oracle-db-connection-manager-service
   - External-IP: 168.138.220.146

   ```
   $ kubectl get all
   NAME                                               READY   STATUS    RESTARTS   AGE
   pod/oracle-db-connection-manager-9cf4bc654-vpmbd   1/1     Running   0          5m36s
   
   NAME                                           TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)          AGE
   service/kubernetes                             ClusterIP      10.96.0.1      <none>            443/TCP          8d
   service/oracle-db-connection-manager-service   LoadBalancer   10.96.61.86    168.138.220.146   1521:31388/TCP   5m36s
   
   NAME                                           READY   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/oracle-db-connection-manager   1/1     1            1           5m36s
   
   NAME                                                     DESIRED   CURRENT   READY   AGE
   replicaset.apps/oracle-db-connection-manager-9cf4bc654   1         1         1       5m36s
   ```

   

## Register DBCS to the CMAN

1. Install Oracle Instant Client in the bastion host:

   ```
   $ sudo yum install oracle-release-el7
   $ sudo yum list oracle-instantclient*
   $ sudo yum install oracle-instantclient19.3-basic.x86_64 oracle-instantclient19.3-sqlplus.x86_64 oracle-instantclient19.3-tools.x86_64
   ```

   

2. From the bastion host, Login to the DBCS.

   ```
   $ sqlplus sys/WElcome_123#@10.0.10.5:1521/ORCL_nrt1dz.sub981952be8.mycluster.oraclevcn.com as sysdba
    
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

   

4. From bastion host, test the CMAN using the external-ip of the CMAN service.

   ```
   [opc@oke-bastion ~]$ sqlplus system/WElcome_123#@168.138.220.146:1521/pdb1.sub981952be8.mycluster.oraclevcn.com
   
   SQL*Plus: Release 19.0.0.0.0 - Production on Wed Mar 25 02:51:14 2020
   Version 19.5.0.0.0
   
   Copyright (c) 1982, 2019, Oracle.  All rights reserved.
   
   Last Successful login time: Wed Mar 25 2020 02:50:12 +00:00
   
   Connected to:
   Oracle Database 19c EE High Perf Release 19.0.0.0.0 - Production
   Version 19.6.0.0.0
   
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

   modify the ```##ORDS_PASSWORD##``` to ```WElcome_123#```.

   The file looks like:

   ```
   <?xml version="1.0" encoding="UTF-8" standalone="no"?>
   <!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
   <properties>
   <comment>Saved on Fri Aug 23 11:24:16 GMT 2019</comment>
   <entry key="db.password">!WElcome_123#</entry>
   <entry key="db.username">ORDS_PUBLIC_USER</entry>
   
   </properties>
   ```

3. Edit the *apex_pu.xml* file.

   ```
   $ vi ./examples/ords/configmaps/apex_pu.xml
   ```

   change all ```##ORDS_PASSWORD##``` to ```WElcome_123#```.

   The file looks like:

   ```
   <?xml version="1.0" encoding="UTF-8" standalone="no"?>
   <!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
   <properties>
   <comment>Saved on Mon Aug 26 11:22:54 GMT 2019</comment>
   <entry key="db.cdb.adminUser">C##DBAPI_CDB_ADMIN as SYSDBA</entry>
   <entry key="db.cdb.adminUser.password">!WElcome_123#</entry>
   <entry key="db.password">!WElcome_123#</entry>
   <entry key="db.username">ORDS_PUBLIC_USER</entry>
   </properties>
   ```

4. Edit the *defaults.xml* file.

   ```
   vi ./examples/ords/configmaps/defaults.xml
   ```

   - change ```##ORDS_PASSWORD##``` to ```WElcome_123#```.
   - change ```##DATABASE_CDB_SERVICE_NAME##``` to ```ORCL_nrt1dz.sub981952be8.mycluster.oraclevcn.com```.
   - change db hostname from  ```oracle-db-enterprise``` to ```dbcs.sub981952be8.mycluster.oraclevcn.com```

   The file looks like:

   ```
   <?xml version="1.0" encoding="UTF-8" standalone="no"?>
   <!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
   <properties>
   <comment>Saved on Mon Aug 26 11:24:32 GMT 2019</comment>
   <entry key="database.api.admin.enabled">true</entry>
   <entry key="database.api.enabled">true</entry>
   <entry key="db.cdb.adminUser">C##DBAPI_CDB_ADMIN as SYSDBA</entry>
   <entry key="db.cdb.adminUser.password">!WElcome_123#</entry>
   <entry key="db.hostname">dbcs.sub981952be8.mycluster.oraclevcn.com</entry>
   <entry key="db.port">1521</entry>
   <entry key="db.servicename">ORCL_nrt1dz.sub981952be8.mycluster.oraclevcn.com</entry>
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

   Copy the credentials content generate in Append3 using password "WElcome_123#".

   The file looks like:

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
   - change oracle host from ```oracle-db-enterprise``` to ```dbcs.sub981952be8.mycluster.oraclevcn.com```
   - change Oracle Service from ```ORCLCDB``` to ```ORCL_nrt1dz.sub981952be8.mycluster.oraclevcn.com```
   - change Oracle PWD from ```##DB_PASSWORD##``` to ```WElcome_123#```
   - change ORDS PWD from ```##ORDS_PASSWORD##``` to ```WElcome_123#```
   - change Oracle Base from ```/opt/oracle/``` to ```/u01/app/oracle/product/19.0.0/dbhome_1```
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
                 value: dbcs.sub981952be8.mycluster.oraclevcn.com
               - name: ORACLE_SERVICE
                 value: ORCL_nrt1dz.sub981952be8.mycluster.oraclevcn.com
               - name: ORACLE_PWD
                 value: WElcome_123#
               - name: ORDS_PWD
                 value: WElcome_123#
               - name: ORACLE_BASE
                 value: /u01/app/oracle/product/19.0.0/dbhome_1
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

    
15. Test the ORDS. From the bastion host, Connect to PDB1:

    ```
    [opc@oke-bastion oracle-db-operator]$ sqlplus system/WElcome_123#@10.0.10.5:1521/pdb1.sub981952be8.mycluster.oraclevcn.com
    
    SQL*Plus: Release 19.0.0.0.0 - Production on Mon Mar 23 11:52:48 2020
    Version 19.5.0.0.0
    
    Copyright (c) 1982, 2019, Oracle.  All rights reserved.
    
    Last Successful login time: Mon Mar 23 2020 10:30:50 +00:00
    
    Connected to:
    Oracle Database 19c EE High Perf Release 19.0.0.0.0 - Production
    Version 19.6.0.0.0
    
    SQL> 
    ```

    ​    

17. Create a test user, and grant the priviledge 

    ```
    CREATE USER testuser1 IDENTIFIED BY WElcome_123#
      DEFAULT TABLESPACE users QUOTA UNLIMITED ON users;
      
    GRANT CREATE SESSION, CREATE TABLE TO testuser1;
    ```

    

18. Connect database with testuser1.

    ```
    connect testuser1/WElcome_123#@10.0.10.5:1521/pdb1.sub981952be8.mycluster.oraclevcn.com
    ```
    
    

19. Create a test emp table and insert some records.

    ```
    CREATE TABLE EMP (
      EMPNO NUMBER(4,0), 
      ENAME VARCHAR2(10 BYTE), 
      JOB VARCHAR2(9 BYTE), 
      MGR NUMBER(4,0), 
      HIREDATE DATE, 
      SAL NUMBER(7,2), 
      COMM NUMBER(7,2), 
      DEPTNO NUMBER(2,0), 
      CONSTRAINT PK_EMP PRIMARY KEY (EMPNO)
      );
    
    insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,SAL,COMM,DEPTNO) values (7369,'SMITH','CLERK',7902,to_date('17-DEC-80','DD-MON-RR'),800,null,20);
    insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,SAL,COMM,DEPTNO) values (7499,'ALLEN','SALESMAN',7698,to_date('20-FEB-81','DD-MON-RR'),1600,300,30);
    insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,SAL,COMM,DEPTNO) values (7521,'WARD','SALESMAN',7698,to_date('22-FEB-81','DD-MON-RR'),1250,500,30);
    insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,SAL,COMM,DEPTNO) values (7566,'JONES','MANAGER',7839,to_date('02-APR-81','DD-MON-RR'),2975,null,20);
    insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,SAL,COMM,DEPTNO) values (7654,'MARTIN','SALESMAN',7698,to_date('28-SEP-81','DD-MON-RR'),1250,1400,30);
    insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,SAL,COMM,DEPTNO) values (7698,'BLAKE','MANAGER',7839,to_date('01-MAY-81','DD-MON-RR'),2850,null,30);
    insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,SAL,COMM,DEPTNO) values (7782,'CLARK','MANAGER',7839,to_date('09-JUN-81','DD-MON-RR'),2450,null,10);
    insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,SAL,COMM,DEPTNO) values (7788,'SCOTT','ANALYST',7566,to_date('19-APR-87','DD-MON-RR'),3000,null,20);
    insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,SAL,COMM,DEPTNO) values (7839,'KING','PRESIDENT',null,to_date('17-NOV-81','DD-MON-RR'),5000,null,10);
    insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,SAL,COMM,DEPTNO) values (7844,'TURNER','SALESMAN',7698,to_date('08-SEP-81','DD-MON-RR'),1500,0,30);
    insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,SAL,COMM,DEPTNO) values (7876,'ADAMS','CLERK',7788,to_date('23-MAY-87','DD-MON-RR'),1100,null,20);
    insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,SAL,COMM,DEPTNO) values (7900,'JAMES','CLERK',7698,to_date('03-DEC-81','DD-MON-RR'),950,null,30);
    insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,SAL,COMM,DEPTNO) values (7902,'FORD','ANALYST',7566,to_date('03-DEC-81','DD-MON-RR'),3000,null,20);
    insert into EMP (EMPNO,ENAME,JOB,MGR,HIREDATE,SAL,COMM,DEPTNO) values (7934,'MILLER','CLERK',7782,to_date('23-JAN-82','DD-MON-RR'),1300,null,10);
    commit;
    ```

    

20. Enable the schema for ORDS

    ```
    BEGIN
    
        ORDS.ENABLE_SCHEMA(p_enabled => TRUE,
                           p_schema => 'TESTUSER1',
                           p_url_mapping_type => 'BASE_PATH',
                           p_url_mapping_pattern => 'testuser1',
                           p_auto_rest_auth => FALSE);
    
        commit;
    
    END;
    /
    ```

    

21. Enable the object.

    ```
    BEGIN
    
        ORDS.ENABLE_OBJECT(p_enabled => TRUE,
                           p_schema => 'TESTUSER1',
                           p_object => 'EMP',
                           p_object_type => 'TABLE',
                           p_object_alias => 'emp',
                           p_auto_rest_auth => FALSE);
    
        commit;
    
    END;
    /
    ```

    

22. Exit sqlplus and test the ORDS using the following command:

    ```
    [opc@oke-bastion oracle-db-operator]$ kubectl exec -it oracle-db-ords-557d787956-wkmhq bash
    [oracle@oracle-db-ords-557d787956-wkmhq ~]$ curl http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/
```
    
The result like this:
    
    ```
    {"items":[{"empno":7369,"ename":"SMITH","job":"CLERK","mgr":7902,"hiredate":"1980-12-17T00:00:00Z","sal":800,"comm":null,"deptno":20,"links":[{"rel":"self","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/7369"}]},{"empno":7499,"ename":"ALLEN","job":"SALESMAN","mgr":7698,"hiredate":"1981-02-20T00:00:00Z","sal":1600,"comm":300,"deptno":30,"links":[{"rel":"self","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/7499"}]},{"empno":7521,"ename":"WARD","job":"SALESMAN","mgr":7698,"hiredate":"1981-02-22T00:00:00Z","sal":1250,"comm":500,"deptno":30,"links":[{"rel":"self","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/7521"}]},{"empno":7566,"ename":"JONES","job":"MANAGER","mgr":7839,"hiredate":"1981-04-02T00:00:00Z","sal":2975,"comm":null,"deptno":20,"links":[{"rel":"self","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/7566"}]},{"empno":7654,"ename":"MARTIN","job":"SALESMAN","mgr":7698,"hiredate":"1981-09-28T00:00:00Z","sal":1250,"comm":1400,"deptno":30,"links":[{"rel":"self","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/7654"}]},{"empno":7698,"ename":"BLAKE","job":"MANAGER","mgr":7839,"hiredate":"1981-05-01T00:00:00Z","sal":2850,"comm":null,"deptno":30,"links":[{"rel":"self","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/7698"}]},{"empno":7782,"ename":"CLARK","job":"MANAGER","mgr":7839,"hiredate":"1981-06-09T00:00:00Z","sal":2450,"comm":null,"deptno":10,"links":[{"rel":"self","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/7782"}]},{"empno":7788,"ename":"SCOTT","job":"ANALYST","mgr":7566,"hiredate":"1987-04-19T00:00:00Z","sal":3000,"comm":null,"deptno":20,"links":[{"rel":"self","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/7788"}]},{"empno":7839,"ename":"KING","job":"PRESIDENT","mgr":null,"hiredate":"1981-11-17T00:00:00Z","sal":5000,"comm":null,"deptno":10,"links":[{"rel":"self","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/7839"}]},{"empno":7844,"ename":"TURNER","job":"SALESMAN","mgr":7698,"hiredate":"1981-09-08T00:00:00Z","sal":1500,"comm":0,"deptno":30,"links":[{"rel":"self","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/7844"}]},{"empno":7876,"ename":"ADAMS","job":"CLERK","mgr":7788,"hiredate":"1987-05-23T00:00:00Z","sal":1100,"comm":null,"deptno":20,"links":[{"rel":"self","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/7876"}]},{"empno":7900,"ename":"JAMES","job":"CLERK","mgr":7698,"hiredate":"1981-12-03T00:00:00Z","sal":950,"comm":null,"deptno":30,"links":[{"rel":"self","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/7900"}]},{"empno":7902,"ename":"FORD","job":"ANALYST","mgr":7566,"hiredate":"1981-12-03T00:00:00Z","sal":3000,"comm":null,"deptno":20,"links":[{"rel":"self","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/7902"}]},{"empno":7934,"ename":"MILLER","job":"CLERK","mgr":7782,"hiredate":"1982-01-23T00:00:00Z","sal":1300,"comm":null,"deptno":10,"links":[{"rel":"self","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/7934"}]}],"hasMore":false,"limit":25,"offset":0,"count":14,"links":[{"rel":"self","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/"},{"rel":"edit","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/"},{"rel":"describedby","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/metadata-catalog/emp/"},{"rel":"first","href":"http://oracle-db-ords:8888/ords/pdb1/testuser1/emp/"}]}
```
    

    
The ORDS is ready.
    
  
    
      

## Deploy Oracle Database Operator

1. In the bastion host. Make sure you are under the right directory

   ```
   $ cd /home/opc/oracle-db-operator
   ```

2. Edit *ords-credentials.yaml* file. Use base64 encode the username and password.

   ```
   $ echo C##DBAPI_CDB_ADMIN as SYSDBA|base64
   QyMjREJBUElfQ0RCX0FETUlOIGFzIFNZU0RCQQo=
$ echo ORDS_PUBLIC_USER|base64
   T1JEU19QVUJMSUNfVVNFUgo=
   $ echo C##DBAPI_CDB_ADMIN|base64
   QyMjREJBUElfQ0RCX0FETUlOCg==
   $ echo admin|base64
YWRtaW4K
   
$ echo WElcome_123#|base64
   V0VsY29tZV8xMjMjCg==
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
     username: QyMjREJBUElfQ0RCX0FETUlOIGFzIFNZU0RCQQo=
     password: V0VsY29tZV8xMjMjCg==
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
   - change ```##PDBNAME##``` to ```mypdb```.
   - change port from ```8080``` to ```8888```
   - -----change ```oracle-db-connection-manager-service``` to ```10.0.30.2``` (**Currently my CMAN image not work in kubernetes**)
   - change DB_FILENAME_CONVERSION_PATTERN from ```/opt/oracle/oradata/ORCLCDB``` to ```/u02/app/oracle/oradata/ORCL_nrt1dz```

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
             value: "('/u02/app/oracle/oradata/ORCL_nrt1dz/pdbseed/','/u02/app/oracle/oradata/ORCL_nrt1dz/mypdb/')"
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

   

8. sadf

9. asdf

10. asd

     



## Provision your database through operator

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

   ```
   $ kubectl apply -f examples/cr.yaml
   ```

   

4. sdaf

5. sadf

6. asdf

7. asdf

8. asdf





















