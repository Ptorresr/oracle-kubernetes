# Build Oracle Database Operator image

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
   $ git clone https://github.com/minqiaowang/oracle-db-operator
   ```

4. Make sure you have install the JRE before. otherwise download the *server-jre-8u241-linux-x64.tar.gz* from https://www.oracle.com/java/technologies/javase-server-jre8-downloads.html. In order to compile the operator you need to execute the following command:

   ```
   $ cd oracle-db-operator
   $ make build
   ```

5. once compiled you can build the docker file and push it to your registry, for example with:

   ```
   docker build -t minqiao/oracle-db-operator:1.0.4 -f Dockerfile .
   ```

6. List the docker image that you just build.

   ```
   [opc@oke-bastion 19.3.0]$ docker image ls
   REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
   minqiao/oracle-db-operator   1.0.4               12de2f08a927        6 days ago          304MB
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
    [opc@oke-bastion oracle-db-operator]$ docker push minqiao/oracle-db-operator:1.0.4
    The push refers to repository [docker.io/minqiao/oracle-db-operator]
    0b52df927548: Pushed 
    64bbf707ce70: Mounted from library/openjdk 
    6e7c36ed7861: Mounted from library/openjdk 
    33328dafa5c7: Mounted from library/openjdk 
    f2cb0ecef392: Mounted from library/openjdk 
    1.0.4: digest: sha256:d02dd719cb619d3c87d70ea0c6b20ed7ec734d3ab9f79808235450a5b8b12d4c size: 1372
    [opc@oke-bastion oracle-db-operator]$ 
    ```

Now, the Oracle Database Operator image is ready.
