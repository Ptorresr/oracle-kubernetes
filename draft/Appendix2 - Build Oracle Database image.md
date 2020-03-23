# Create the Oracle Database image

## Prerequisites

You need a Docker Account (hub.docker.com) before push the docker image to the remote repository.

## Build the image

1. Install docker engine.

   ```
   $ sudo yum install docker-engine
   $ sudo usermod -aG docker opc
   $ sudo systemctl enable docker
   $ sudo systemctl start docker
   ```

2. Next, we are going to install git.

   ```
   $ sudo yum install git
   ```

3. Make sure you are in the /home/opc directory. You will clone some setup scripts from git.

   ```
   $ git clone https://github.com/oracle/docker-images
   ```

4. Download DB 19.3 binary from https://www.oracle.com/technetwork/database/enterprise-edition/downloads

5. Move the file LINUX.X64_193000_db_home.zip to the Dockerfile directory

   ```
   $ mv LINUX.X64_193000_db_home.zip docker-images/OracleDatabase/SingleInstance/dockerfiles/19.3.0/.
   ```

6. Run the following command to build the docker image.

   ```
   $ cd docker-images/OracleDatabase/SingleInstance/dockerfiles
   $ ./buildDockerImage.sh -v 19.3.0 -e -i
   ```

7. List the docker image that you just build.

   ```
   [opc@oke-bastion dockerfiles]$ docker image ls
   REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
   oracle/database     19.3.0-ee           cb126059b5d6        4 minutes ago       6.51GB
   oraclelinux         7-slim              fd84774952b5        7 days ago          118MB
   [opc@oke-bastion dockerfiles]$
   ```

8. Log into dock hub with your own account.

   ```
   $ docker login
   ```

9. Change the image tag to *your-dockerhub-account/image-name:tag*

   ```
   [opc@oke-bastion dockerfiles]$ docker tag cb126059b5d6 minqiao/database:19.3.0-ee
   [opc@oke-bastion dockerfiles]$ docker image ls
   REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
   oracle/database     19.3.0-ee           cb126059b5d6        13 minutes ago      6.51GB
   minqiao/database    19.3.0-ee           cb126059b5d6        13 minutes ago      6.51GB
   oraclelinux         7-slim              fd84774952b5        7 days ago          118MB
   [opc@oke-bastion dockerfiles]$
   ```

10. Push this image to the remote repository with your docker hub account *your-dockerhub-account/image-name:tag*

   ```
   [opc@oke-bastion dockerfiles]$ docker push minqiao/database:19.3.0-ee
   The push refers to repository [docker.io/minqiao/database]
   86748e1c9908: Pushed 
   2e303059bca7: Pushed 
   5653a67424c9: Pushed 
   11a62013f56c: Pushed 
   45705ae23c69: Pushed 
   12a9cd7d069e: Layer already exists 
   19.3.0-ee: digest: sha256:f8f97c050fa31e8b25af8835cf943250267acca1c50ca4b64b869459683c9197 size: 1581
   [opc@oke-bastion dockerfiles]$ 
   ```

Now, the database image is ready.

## Test the docker image

1. Before creating container, back to the **/home/opc** directory, and create a **oradata** directory which map to the directory in container.

   ```
   [opc@oke-bastion dockerfiles]$ cd ~
   [opc@oke-bastion ~]$ mkdir oradata
   [opc@oke-bastion ~]$ chmod 777 oradata 
   [opc@oke-bastion ~]$
   ```
   
3. Execute following command  to create database container using your own repository: *your-dockerhub-account/image-name:tag*. Use the network same as the Connection Manager container. If the port is already allocate to the other container. try to map the port with other number, like:* -p 1522:1521*

   ```
   docker run -d --network=db_pub1_nw --hostname dbhost --name db19c \
   -p 1521:1521 -p 5500:5500 \
   -e ORACLE_SID=ORCL \
   -e ORACLE_PDB=PDB1 \
   -e ORACLE_PWD=WElcome_123# \
   -e ORACLE_CHARACTERSET=AL32UTF8 \
   -v /home/opc/oradata:/opt/oracle/oradata \
   minqiao/database:19.3.0-ee
   ```

4. To watch the progress type the following command passing the name of the container: db19c.

   ```
   docker logs --follow db19c
   ```

5. When the database creationg is complete, you may see **The Database is Ready to Use**.  Press **control-c** to continue.

   ```
   #########################
   DATABASE IS READY TO USE!
   #########################
   The following output is now a tail of the alert.log:
   QPI: qopiprep.bat file present
   2020-03-18T01:22:27.735438+00:00
   PDB1(3):Opening pdb with no Resource Manager plan active
   2020-03-18T01:22:27.853059+00:00
   PDB1(3):joxcsys_required_dirobj_exists: directory object exists with required path /opt/oracle/product/19c/dbhome_1/javavm/admin/, pid 105 cid 3
   Pluggable database PDB1 opened read write
   Starting background process CJQ0
   2020-03-18T01:22:28.150184+00:00
   CJQ0 started with pid=48, OS id=277 
   Completed: ALTER DATABASE OPEN
   2020-03-18T01:22:32.633977+00:00
   ...
   ```

6. Now the container is running.

   ```
   [opc@oke-bastion ~]$ docker container ls
   CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS                    PORTS                                            NAMES
   ab3c12754b19        minqiao/database:19.3.0-ee   "/bin/sh -c 'exec $O…"   27 minutes ago      Up 27 minutes (healthy)   0.0.0.0:1521->1521/tcp, 0.0.0.0:5500->5500/tcp   db19c
   [opc@oke-bastion ~]$
   ```

7. Login to the container:

   ```
   [opc@oke-bastion ~]$ docker exec -it db19c bash
   [oracle@dbhost ~]$ 
   ```

8. Check the Database status, try to connect with sqlplus, and exit.

   ```
   [oracle@dbhost ~]$ lsnrctl status
   
   LSNRCTL for Linux: Version 19.0.0.0.0 - Production on 17-MAR-2020 12:32:58
   
   Copyright (c) 1991, 2019, Oracle.  All rights reserved.
   
   Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=IPC)(KEY=EXTPROC1)))
   STATUS of the LISTENER
   ------------------------
   Alias                     LISTENER
   Version                   TNSLSNR for Linux: Version 19.0.0.0.0 - Production
   Start Date                17-MAR-2020 12:04:19
   Uptime                    0 days 0 hr. 28 min. 38 sec
   Trace Level               off
   Security                  ON: Local OS Authentication
   SNMP                      OFF
   Listener Parameter File   /opt/oracle/product/19c/dbhome_1/network/admin/listener.ora
   Listener Log File         /opt/oracle/diag/tnslsnr/dbhost/listener/alert/log.xml
   Listening Endpoints Summary...
     (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1)))
     (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=0.0.0.0)(PORT=1521)))
     (DESCRIPTION=(ADDRESS=(PROTOCOL=tcps)(HOST=dbhost)(PORT=5500))(Security=(my_wallet_directory=/opt/oracle/admin/ORCL/xdb_wallet))(Presentation=HTTP)(Session=RAW))
   Services Summary...
   Service "ORCL" has 1 instance(s).
     Instance "ORCL", status READY, has 1 handler(s) for this service...
   Service "ORCLXDB" has 1 instance(s).
     Instance "ORCL", status READY, has 1 handler(s) for this service...
   Service "a10cff9c5a7e0c4de053020011ac3924" has 1 instance(s).
     Instance "ORCL", status READY, has 1 handler(s) for this service...
   Service "pdb1" has 1 instance(s).
     Instance "ORCL", status READY, has 1 handler(s) for this service...
   The command completed successfully
   [oracle@dbhost ~]$ sqlplus system/WElcome_123#@dbhost:1521/pdb1
   
   SQL*Plus: Release 19.0.0.0.0 - Production on Tue Mar 17 12:34:04 2020
   Version 19.3.0.0.0
   
   Copyright (c) 1982, 2019, Oracle.  All rights reserved.
   
   
   Connected to:
   Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
   Version 19.3.0.0.0
   
   SQL> exit
   Disconnected from Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
   Version 19.3.0.0.0
   [oracle@dbhost ~]$ exit
   exit
   [opc@oke-bastion ~]$ 
   ```

   There is no problem in this container.

