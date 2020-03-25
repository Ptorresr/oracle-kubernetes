# Create the Oracle Connection Manager image

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
   $ git clone https://github.com/malagoli/docker-images
   ```

4. Download Oracle Rest Data Services *ords-19.2.0.199.1647.zip* from http://www.oracle.com/technetwork/developer-tools/rest-data-services/downloads/index.html

5. Copy the file *ords-19.2.0.199.1647.zip* to the Dockerfile directory

   ```
   $ cp ords-19.2.0.199.1647.zip docker-images/OracleRestDataServices/dockerfiles
   ```

6. Edit the image build scripts. Found the ```IMAGE_NAME="oracle/restdataservices:$VERSION"```, modify to  your own account,  *your-dockerhub-account/image-name:tag*, for example: ```IMAGE_NAME="minqiao/restdataservices:$VERSION"```, 

   Set the version ```VERSION=""``` to ```VERSION="19.2.0"```. 

   Save the file.

   ```
   $ cd docker-images/OracleRestDataServices/dockerfiles
   $ vi buildDockerImage.sh
   ```

7. Edit Dockerfile, modify ```FROM container-registry.oracle.com/java/serverjre:8``` to ```FROM minqiao/serverjre:8```

8. Run the following command to build the docker image.

   ```
   $ ./buildDockerImage.sh
   ```

9. List the docker image that you just build.

   ```
   [opc@oke-bastion dockerfiles]$ docker image ls
   REPOSITORY                 TAG                 IMAGE ID            CREATED              SIZE
   minqiao/restdataservices   19.2.0              3a970618cb82        About a minute ago   399MB
   minqiao/serverjre          8                   7c6632fb2dde        20 minutes ago       270MB
   oraclelinux                7-slim              fd84774952b5        9 days ago           118MB
   [opc@oke-bastion dockerfiles]$ 
   ```

10. Log into dock hub with your own account.

   ```
   docker login
   ```

11. Push this image to the remote repository with your docker hub account *your-dockerhub-account/image-name:tag*

    ```
    [opc@oke-bastion 19.3.0]$ docker push minqiao/restdataservices:19.2.0
    ```

Now, the Oracle Rest Data Services image is ready.

##Test the docker image

1. Before creating container, create the bridge. If you are using same bridge with same network then you can use same IPs mentioned in **Create Containers** section.

   ```
   docker network create --driver=bridge --subnet=172.16.1.0/24 db_pub1_nw
   ```
   
2. Execute following command  to create ords container with your own account: *your-dockerhub-account/image-name:tag*. Make sure a database has already run, and you have the information like host port service and the administrator's password.

   ```
   docker run -d --name ords \
   --network=db_pub1_nw \
   -p 8888:8888 \
   -e ORACLE_HOST=172.16.1.2 \
   -e ORACLE_PORT=1521 \
   -e ORACLE_SERVICE=ORCL \
   -e ORACLE_PWD=WElcome_123# \
   -e ORDS_PWD=WElcome_123# \
   -v /home/opc/ords:/opt/oracle/ords/config/ords \
   minqiao/restdataservices:19.2.0
   ```

3. To check the Cman container/services creation logs , please tail docker logs. It will take 5 minutes to create the Cman container service.

   ```
   docker logs --follow ords
   ```

   You should see following when cman container setup is done:  Press **control-c** to continue.

   ```
   Mar 19, 2020 6:56:10 AM  
   INFO: Oracle REST Data Services initialized
   Oracle REST Data Services version : 19.2.0.r1991647
   Oracle REST Data Services server info: jetty/9.4.z-SNAPSHOT
   
   2020-03-19 06:56:11.534:INFO:oejsh.ContextHandler:main: Started o.e.j.s.ServletContextHandler@1d057a39{/ords,null,AVAILABLE}
   2020-03-19 06:56:11.536:INFO:oejsh.ContextHandler:main: Started o.e.j.s.h.ContextHandler@464bee09{/,null,AVAILABLE}
   2020-03-19 06:56:11.537:INFO:oejsh.ContextHandler:main: Started o.e.j.s.h.ContextHandler@f6c48ac{/i,null,AVAILABLE}
   2020-03-19 06:56:11.541:INFO:oejs.RequestLogWriter:main: Opened /tmp/ords_log/ords_2020_03_19.log
   2020-03-19 06:56:11.563:INFO:oejs.AbstractConnector:main: Started ServerConnector@1ce24091{HTTP/1.1,[http/1.1, h2c]}{0.0.0.0:8888}
   2020-03-19 06:56:11.565:INFO:oejs.Server:main: Started @8209ms
   ```

4. Log into the ords container:

   ```
   [opc@oke-bastion oracle-db-operator]$ docker exec -it ords bash
   [oracle@0868f7e6b836 ~]$
   ```

5. Generate the password for the credentials file, use the password: *WElcome_123#*.

   ```
   [oracle@0868f7e6b836 ~]$ cd /opt/oracle/ords/
   [oracle@0868f7e6b836 ords]$ java -jar ords.war user admin "SQL Administrator,System Administrator"
   Enter a password for user admin: 
   Confirm password for user admin: 
   Mar 20, 2020 6:01:03 AM oracle.dbtools.standalone.ModifyUser execute
   INFO: Created user: admin in file: /opt/oracle/ords/config/ords/credentials
   [oracle@0868f7e6b836 ords]$ cat /opt/oracle/ords/config/ords/credentials
   admin;{SSHA-512}I9yp9gOvNtML/DfGkz2XHqb/lXbmeryleCLRGPdoDG1r67dKDPxPosgeG57Y08iDbT7JMnmxXNxz3QTAZzgu95fKBqx2P/wx;SQL Administrator,System Administrator
   [oracle@0868f7e6b836 ords]$ 
   ```

6. Copy the content in the /opt/oracle/ords/config/ords/credentials, this will used in Lab4

   ```
   admin;{SSHA-512}I9yp9gOvNtML/DfGkz2XHqb/lXbmeryleCLRGPdoDG1r67dKDPxPosgeG57Y08iDbT7JMnmxXNxz3QTAZzgu95fKBqx2P/wx;SQL Administrator,System Administrator
   ```

   