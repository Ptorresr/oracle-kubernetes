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

2. Edit the *init.ora* file, modify all the *<ORACLE_BASE>* to */opt/oracle*.

   ```
   $ vi examples/database/configmaps/init.ora
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
   vi examples/database/configmaps/init.sql
   ```

   It's looks like the following, save the file.

   ```
   -- ORDS USER
   CREATE USER C##DBAPI_CDB_ADMIN IDENTIFIED BY "Welcome_123#";
   GRANT SYSDBA TO C##DBAPI_CDB_ADMIN CONTAINER = ALL;
   GRANT connect  TO C##DBAPI_CDB_ADMIN CONTAINER = ALL;
   
   -- CONNECTION TO CONNECTION MANAGER
   alter system  set remote_listener='oracle-db-connection-manager-service:1521' scope=both sid='*';
   alter system register;
   ```

4. Edit the *tnsnames.ora* file

   ```
   $ vi examples/database/configmaps/tnsnames.ora
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
   $ vi examples/database/oracle-db-deployment.yaml
   ```

   - change ```##DOCKER_REGISTRY##/oracle-db:19.3.0.0 ``` to ```minqiao/database:19.3.0-ee```

   - change SID  ```value: "ORCLCDB"``` to ```value: "ORCL"```

   - change PDB ```value: "ORCLPDB1"``` to ```value: "PDB1"```

   - change PWD ```##ORACLE_SYS_PASSWORD##``` to ```Welcome_123#```

     

7. Deploy the Oracle Database

   ```
   $ kubectl apply -f examples/database/oracle-db-deployment.yaml
   ```

8. List the pod

   ```
   [opc@oke-bastion oracle-db-operator]$ kubectl get pod
   NAME                                           READY   STATUS              RESTARTS   AGE
   oracle-db-connection-manager-d474757ff-rhs98   1/1     Running             10         38m
   oracle-db-enterprise-8d67f7cbb-c78c2           0/1     ContainerCreating   0          17s
   ```

9. It's need some time to download the container image. List the pods again, when the STATUS change to Running, you can check the log.

   ```
   $ kubectl logs -f oracle-db-enterprise-8d67f7cbb-c78c2
   ```

   The inital database setup will take abourt 15 minutes if you encounter the *error: unexpected EOF*, try to re-enter the log check.

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

2. sadf

3. sdf

4. sdaf

5. sadf

6. sdaf

7. sadf

8. sadf

9. sdf

10. sdaf

11. asdf

12. sadf

13. 