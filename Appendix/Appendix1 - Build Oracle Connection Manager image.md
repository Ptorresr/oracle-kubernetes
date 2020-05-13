# Build Oracle Connection Manager image

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

4. Download DB 19.3 Client binary from https://www.oracle.com/technetwork/database/enterprise-edition/downloads

5. Copy the file LINUX.X64_193000_client.zip to the Dockerfile directory

   ```
   $ cp LINUX.X64_193000_client.zip docker-images/OracleDatabase/RAC/OracleConnectionManager/dockerfiles/19.3.0/
   ```

6. Edit the image build scripts. Found the ```IMAGE_NAME="oracle/client-cman:$VERSION"```, modify to  your own account,  *your-dockerhub-account/image-name:tag*, for example: ```IMAGE_NAME="minqiao/cman:$VERSION"```, save the file.

   ```
   $ cd docker-images/OracleDatabase/RAC/OracleConnectionManager/dockerfiles
   $ vi buildDockerImage.sh
   ```

7. Run the following command to build the docker image.

   ```
   $ ./buildDockerImage.sh -v 19.3.0
   ```

8. List the docker image that you just build.

   ```
   [opc@oke-bastion 19.3.0]$ docker image ls
   REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
   minqiao/cman        19.3.0              8f750ff8b71e        About a minute ago   3.53GB
   oraclelinux         7-slim              fd84774952b5        7 days ago           118MB
   [opc@oke-bastion 19.3.0]$ 
   ```

9. Log into dock hub with your own account.

   ```
   docker login
   ```

10. Push this image to the remote repository with your docker hub account *your-dockerhub-account/image-name:tag*

    ```
    [opc@oke-bastion 19.3.0]$ docker push minqiao/cman:19.3.0
    The push refers to repository [docker.io/minqiao/cman]
    b7598f7cc2e4: Pushed 
    26083f69a8ef: Pushed 
    a5902fa492e5: Pushed 
    12a9cd7d069e: Layer already exists 
    19.3.0: digest: sha256:4de2e8d67d914de62b1aa4fa62db1eafd2a9cba28746e03e24ea6d709e272839 size: 1164
    [opc@oke-bastion 19.3.0]$
    ```

Now, the Oracle Connection Manager image is ready.

##Test the docker image

1. Before creating container, create the bridge. If you are using same bridge with same network then you can use same IPs mentioned in **Create Containers** section.

   ```
   docker network create --driver=bridge --subnet=172.16.1.0/24 db_pub1_nw
   ```
   
2. Execute following command  to create connection manager container using the remote repositroy with your own account: *your-dockerhub-account/image-name:tag*

   ```
   docker run -d --hostname cman1 --dns-search=example.com \
   --network=db_pub1_nw --ip=172.16.1.15 \
   -e DOMAIN=example.com -e PUBLIC_IP=172.16.1.15 \
   -e PUBLIC_HOSTNAME=cman1 \
   -e SCAN_IP=172.16.1.2 -e SCAN_NAME=dbhost
   --privileged=false \
   -p 1521:1521 --name cman minqiao/cman:19.3.0
   ```

4. To check the Cman container/services creation logs , please tail docker logs. It will take 5 minutes to create the Cman container service.

   ```
   docker logs --follow cman
   ```

   You should see following when cman container setup is done:  Press **control-c** to continue.

   ```
   03-18-2020 01:13:06 UTC :  : ################################################
   03-18-2020 01:13:06 UTC :  :  CONNECTION MANAGER IS READY TO USE!            
   03-18-2020 01:13:06 UTC :  : ################################################
   03-18-2020 01:13:06 UTC :  : cman started sucessfully
   ```

5. Now the container is running.

   ```
   [opc@oke-bastion 19.3.0]$ docker container ls
   CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                              NAMES
   4b308111d855        minqiao/cman:19.3.0   "/bin/sh -c 'exec $S…"   8 minutes ago       Up 8 minutes        0.0.0.0:1521->1521/tcp, 5500/tcp   cman
   [opc@oke-bastion 19.3.0]$
   ```

6. Login to the container:

   ```
   [opc@oke-bastion ~]$ docker exec -it cman bash
   [oracle@cman1 ~]$
   ```

7. Check the CMAN status and exit.

   ```
   [oracle@cman1 ~]$ cmctl show status -c CMAN_cman1.example.com
   
   CMCTL for Linux: Version 19.0.0.0.0 - Production on 18-MAR-2020 01:16:51
   
   Copyright (c) 1996, 2019, Oracle.  All rights reserved.
   
   Current instance CMAN_cman1.example.com is already started
   Connecting to (DESCRIPTION=(address=(protocol=tcp)(host=cman1.example.com)(port=1521)))
   Status of the Instance
   ----------------------
   Instance name             cman_cman1.example.com
   Version                   CMAN for Linux: Version 19.0.0.0.0 - Production
   Start date                18-MAR-2020 01:08:48
   Uptime                    0 days 0 hr. 8 min. 22 sec
   Num of gateways started   2
   Average Load level        0
   Log Level                 USER
   Trace Level               USER
   Instance Config file      /u01/app/oracle/product/19.3.0/client_1/network/admin/cman.ora
   Instance Log directory    /u01/app/oracle/diag/netcman/db-cman1/cman_cman1.example.com/alert
   Instance Trace directory  /u01/app/oracle/diag/netcman/db-cman1/cman_cman1.example.com/trace
   The command completed successfully. 
   [oracle@db-cman1 admin]$ exit
   exit
   [opc@oke-bastion ~]$ 
   ```

   Now, The Oracle Connection Manager docker image is working.