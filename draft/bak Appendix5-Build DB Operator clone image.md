1. cp -R oracle-db-operator cl-oracle-db-operator

2. modify ./src/main/java/com/oracle/OrdsClient/OrdsClient.java

     

3. ./src/main/java/com/oracle/OrdsClient/PdbCreationRequestModel.java

   public String getMethod() { return "CREATE"; } to public String getMethod() { return "CLONE"; }

     public String getClonePDBName() {

   ​    return clonePDBName;

     }

   

     public void setClonePDBName(String clonePDBName) {

   ​    this.clonePDBName = clonePDBName;

     }

   

4. ./src/main/java/com/oracle/DbOperator.java

   getClonePDBName()

5. ./src/main/java/com/oracle/OracleService.java

     public String getClonePDBName() {

   ​    return clonePDBName;

     }

   

     public void setClonePDBName(String clonePDBName) {

   ​    this.clonePDBName = clonePDBName;

     }

6. ./src/main/resources/schema/oracleService.json

     "clonePDBName": {

      "type": "string",

     },

7. ./target/classes/schema/oracleService.json

     "clonePDBName": {

      "type": "string",

     },

8. dsf

   

9. make build

10. docker build -t minqiao/oracle-db-operator:clone -f Dockerfile .

11. docker login

12. docker push minqiao/oracle-db-operator:clone





1. cp ./manifest/operator-k8s.yaml ./manifest/cloneoperator-k8s.yaml

2. vi ./manifest/cloneoperator-k8s.yaml

   -- oracle-db-operator to oracle-db-operator-clone

   image: minqiao/oracle-db-operator:1.0.4 to image: minqiao/oracle-db-operator:clone

3. kubectl apply -f manifest/cloneoperator-k8s.yaml

4. get pods

5. logs -f

6. cp ./examples/cr.yaml ./examples/clone.yaml

7. vi ./examples/clone.yaml

8. kubectl apply -f examples/clone.yaml

9. sdaf

10. sadf

