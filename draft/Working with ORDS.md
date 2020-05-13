# Working with ORDS

Oracle REST Data Services is a Java EE-based alternative for Oracle HTTP Server and `mod_plsql`. The Java EE implementation offers increased functionality including a command line based configuration, enhanced security, file caching, and RESTful web services. Oracle REST Data Services also provides increased flexibility by supporting deployments using Oracle WebLogic Server, Apache Tomcat, and a standalone mode.

## Required Artifacts

- You already have a functioning installation of ORDS 17.4 or higher.

- You have an Oracle database available. In this lab we will be using a 19c database

- To enable the REST Enabled SQL functionality we must amend the "defaults.xml" file. This entry is the on/off switch for this functionality. By default the functionality is only available over HTTPS, which is very sensible since the payload contains credentials, but we can allow HTTP access by using the following setting because in this lab the database and ORDS are in the private subnet.

  ```
  <entry key="restEnabledSql.active">true</entry>
  <entry key="security.verifySSL">false</entry>
  ```

- Create a user in Oracle REST Data Services with the **SQL Developer** role. This Oracle REST Data Services user will be able to run SQL for any Oracle database schema that is REST-enabled. For example, we create a user named **test**, and the password set to **WElcome_123#**. 

  ```
  java -jar ords.war user test "SQL Developer"
  ```

  

## Create a Test Database User

1. We need a new database user for testing using **sqlplus**:

   ```
   CONN / AS SYSDBA
   ALTER SESSION SET CONTAINER=orclpdb;
   
   DROP USER testuser1 CASCADE;
   CREATE USER testuser IDENTIFIED BY testuser
     DEFAULT TABLESPACE users QUOTA UNLIMITED ON users;
     
   GRANT CREATE SESSION, CREATE TABLE, CREATE PROCEDURE TO testuser;
   ```

   

2. Create a test table:

   ```
   connect testuser/testuser@orclpdb
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
   ```

   

3. Insert some test records:

   ```
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

   

## Enable ORDS

Enable REST web services for the test schema itself. We could use any unique and legal URL mapping pattern for the schema, so it is not necessary to expose the schema name. In this example we've used a schema alias of **hr**.

```
CONN testuser/testuser@orclpdb

BEGIN
  ORDS.enable_schema(
    p_enabled             => TRUE,
    p_schema              => 'TESTUSER',
    p_url_mapping_type    => 'BASE_PATH',
    p_url_mapping_pattern => 'hr',
    p_auto_rest_auth      => FALSE
  );
    
  COMMIT;
END;
/
```

Enable AutoREST for the EMP table.

```
BEGIN

    ORDS.ENABLE_OBJECT(p_enabled => TRUE,
                       p_schema => 'TESTUSER',
                       p_object => 'EMP',
                       p_object_type => 'TABLE',
                       p_object_alias => 'emp',
                       p_auto_rest_auth => FALSE);

    commit;

END;
/
```

2. Now we can query the table using the AutoREST, GET Web Services (READ)

   ```
   curl http://140.238.44.83:8888/ords/orclpdb/hr/emp/
   ```

3. POST Web Services (INSERT)

   ```
   curl -i -X POST --data-binary @insert-payload.json -H "Content-Type: application/json" http://140.238.44.83:8888/ords/orclpdb/hr/emp/
   ```

   The insert-payload.json file like:

   ```
   {"empno":9999,"ename":"HALL","job":"ANALYST","mgr":7782,"hiredate":"2016-01-01T00:00:00Z","sal":1000,"comm":null,"deptno":10}
   ```

   The output like this:

   ```
   HTTP/1.1 201 Created
   Content-Type: application/json
   Content-Location: http://140.238.44.83:8888/ords/orclpdb/hr/emp/9999
   ETag: "QSBGND2ynPrieQMNnB21vd0FOoZlR2p5+rQ6JYpANln6gTZ6Gxcs4U+YBBZINUaPWGU4u+9Cc+95Y7YY1wyNrg=="
   Location: http://140.238.44.83:8888/ords/orclpdb/hr/emp/9999
   Transfer-Encoding: chunked
   
   {"empno":9999,"ename":"HALL","job":"ANALYST","mgr":7782,"hiredate":"2016-01-01T00:00:00Z","sal":1000,"comm":null,"deptno":10,"col1":null,"links":[{"rel":"self","href":"http://140.238.44.83:8888/ords/orclpdb/hr/emp/9999"},{"rel":"edit","href":"http://140.238.44.83:8888/ords/orclpdb/hr/emp/9999"},{"rel":"describedby","href":"http://140.238.44.83:8888/ords/orclpdb/hr/metadata-catalog/emp/item"},{"rel":"collection","href":"http://140.238.44.83:8888/ords/orclpdb/hr/emp/"}]}
   ```

   

4. PUT Web Services (UPDATE)

   ```
   curl -i -X PUT --data-binary @update-payload.json -H "Content-Type: application/json" http://140.238.44.83:8888/ords/orclpdb/hr/emp/9999
   ```

   The update-payload.json file like this:

   ```
   {"empno":9999,"ename":"WOOD","job":null,"mgr":null,"hiredate":null,"sal":null,"comm":null,"deptno":20}
   ```

   The output like this:

   ```
   HTTP/1.1 200 OK
   Content-Type: application/json
   Content-Location: http://140.238.44.83:8888/ords/orclpdb/hr/emp/9999
   ETag: "uC1GbexgsbwEUgSNgRIm6htH4JYXd5lXnoo12sFOGr+uRYpFzA6EOU4e7qdRGbHGET6XglHeHXB+/+dZKWx1fw=="
   Transfer-Encoding: chunked
   
   {"empno":9999,"ename":"WOOD","job":null,"mgr":null,"hiredate":null,"sal":null,"comm":null,"deptno":20,"col1":null,"links":[{"rel":"self","href":"http://140.238.44.83:8888/ords/orclpdb/hr/emp/9999"},{"rel":"edit","href":"http://140.238.44.83:8888/ords/orclpdb/hr/emp/9999"},{"rel":"describedby","href":"http://140.238.44.83:8888/ords/orclpdb/hr/metadata-catalog/emp/item"},{"rel":"collection","href":"http://140.238.44.83:8888/ords/orclpdb/hr/emp/"}]}
   ```

   

5. DELETE Web Services (DELETE), The URL is an encoded version of q={"empno":9999}:

   ```
   curl -i -X DELETE  http://140.238.44.83:8888/ords/orclpdb/hr/emp/?q=%7B%22empno%22%3A9999%7D
   ```

   The output like this:

   ```
   HTTP/1.1 200 OK
   Date: Thu, 16 Apr 2020 00:49:26 GMT
   Content-Type: application/json
   Transfer-Encoding: chunked
   
   {"rowsDeleted":1}
   ```
   
   

##REST-Enabled SQL Service

The REST Enabled SQL functionality introduced in Oracle REST Data Services (ORDS) 17.4 allows REST calls to send DML, DDL and scripts to any REST enabled schema.

1. Run a Query:

```
curl -s -k -X "POST" "http://140.238.44.83:8888/ords/orclpdb/hr/_/sql" \
       -H "Content-Type: application/sql" \
       -u test:WElcome_123# \
       -d $'SELECT * from emp;'
```

The result like this:

```
{"env":{"defaultTimeZone":"GMT"},"items":[{"statementId":1,"statementType":"query","statementPos":{"startLine":1,"endLine":1},"statementText":"SELECT * from emp","resultSet":{"metadata":[{"columnName":"EMPNO","jsonColumnName":"empno","columnTypeName":"NUMBER","precision":4,"scale":0,"isNullable":0},{"columnName":"ENAME","jsonColumnName":"ename","columnTypeName":"VARCHAR2","precision":10,"scale":0,"isNullable":1},{"columnName":"JOB","jsonColumnName":"job","columnTypeName":"VARCHAR2","precision":9,"scale":0,"isNullable":1},{"columnName":"MGR","jsonColumnName":"mgr","columnTypeName":"NUMBER","precision":4,"scale":0,"isNullable":1},{"columnName":"HIREDATE","jsonColumnName":"hiredate","columnTypeName":"DATE","precision":0,"scale":0,"isNullable":1},{"columnName":"SAL","jsonColumnName":"sal","columnTypeName":"NUMBER","precision":7,"scale":2,"isNullable":1},{"columnName":"COMM","jsonColumnName":"comm","columnTypeName":"NUMBER","precision":7,"scale":2,"isNullable":1},{"columnName":"DEPTNO","jsonColumnName":"deptno","columnTypeName":"NUMBER","precision":2,"scale":0,"isNullable":1}],"items":[{"empno":7369,"ename":"SMITH","job":"CLERK","mgr":7902,"hiredate":"1980-12-17T00:00:00Z","sal":800,"comm":null,"deptno":20},{"empno":7499,"ename":"ALLEN","job":"SALESMAN","mgr":7698,"hiredate":"1981-02-20T00:00:00Z","sal":1600,"comm":300,"deptno":30},{"empno":7521,"ename":"WARD","job":"SALESMAN","mgr":7698,"hiredate":"1981-02-22T00:00:00Z","sal":1250,"comm":500,"deptno":30},{"empno":7566,"ename":"JONES","job":"MANAGER","mgr":7839,"hiredate":"1981-04-02T00:00:00Z","sal":2975,"comm":null,"deptno":20},{"empno":7654,"ename":"MARTIN","job":"SALESMAN","mgr":7698,"hiredate":"1981-09-28T00:00:00Z","sal":1250,"comm":1400,"deptno":30},{"empno":7698,"ename":"BLAKE","job":"MANAGER","mgr":7839,"hiredate":"1981-05-01T00:00:00Z","sal":2850,"comm":null,"deptno":30},{"empno":7782,"ename":"CLARK","job":"MANAGER","mgr":7839,"hiredate":"1981-06-09T00:00:00Z","sal":2450,"comm":null,"deptno":10},{"empno":7788,"ename":"SCOTT","job":"ANALYST","mgr":7566,"hiredate":"1987-04-19T00:00:00Z","sal":3000,"comm":null,"deptno":20},{"empno":7839,"ename":"KING","job":"PRESIDENT","mgr":null,"hiredate":"1981-11-17T00:00:00Z","sal":5000,"comm":null,"deptno":10},{"empno":7844,"ename":"TURNER","job":"SALESMAN","mgr":7698,"hiredate":"1981-09-08T00:00:00Z","sal":1500,"comm":0,"deptno":30},{"empno":7876,"ename":"ADAMS","job":"CLERK","mgr":7788,"hiredate":"1987-05-23T00:00:00Z","sal":1100,"comm":null,"deptno":20},{"empno":7900,"ename":"JAMES","job":"CLERK","mgr":7698,"hiredate":"1981-12-03T00:00:00Z","sal":950,"comm":null,"deptno":30},{"empno":7902,"ename":"FORD","job":"ANALYST","mgr":7566,"hiredate":"1981-12-03T00:00:00Z","sal":3000,"comm":null,"deptno":20},{"empno":7934,"ename":"MILLER","job":"CLERK","mgr":7782,"hiredate":"1982-01-23T00:00:00Z","sal":1300,"comm":null,"deptno":10}],"hasMore":false,"limit":10000,"offset":0,"count":14},"response":[],"result":0}]}
```

2. Run DML
   We can string together one or more DML statements, each ending with a ";". In the script example we did this using a file, but we can include the commands inline, provided we escape any necessary characters.

   ```
   curl -s -k -X "POST" "http://140.238.44.83:8888/ords/orclpdb/hr/_/sql" \
          -H "Content-Type: application/sql" \
          -u test:WElcome_123# \
          -d $'UPDATE emp SET sal = sal + 1 WHERE  empno > 1000;'
   ```
   
   The output like this:

   ```
{"env":{"defaultTimeZone":"GMT"},"items":[{"statementId":1,"statementType":"dml","statementPos":{"startLine":1,"endLine":3},"statementText":"UPDATE emp SET sal = sal + 1 WHERE  empno > 1000","response":["\n15 rows updated.\n\n"],"result":15}]}
   ```
   
3. Run DDL

   ```
   curl -s -k -X "POST" "http://140.238.44.83:8888/ords/orclpdb/hr/_/sql" \
          -H "Content-Type: application/sql" \
          -u test:WElcome_123# \
          -d $'Alter TABLE emp add col1 NUMBER;'
   ```
   

The output like this:

```
   {"env":{"defaultTimeZone":"GMT"},"items":[{"statementId":1,"statementType":"ddl","statementPos":{"startLine":1,"endLine":2},"statementText":"Alter TABLE emp add col1 NUMBER","response":["\nTable EMP altered.\n\n"],"result":0}]}
```



4. JSON Documents
   Rather than a plain SQL statement or script, a JSON document can be sent as a payload to REST Enabled SQL. Create a procedure with some parameters.

   - Create a json file named test.json, the content like following:

   ```
   { 
     "statementText": "SELECT ename FROM emp WHERE deptno = :p_deptno ORDER BY ename;",
     "offset": 2, 
     "limit": 2,
     "binds":[
       {"name":"p_deptno","data_type":"NUMBER","value":20}
     ] 
   }
   ```

   - Run the following command:

   ```
   curl -s -k -X "POST" "http://140.238.44.83:8888/ords/orclpdb/hr/_/sql" \
          -H "Content-Type: application/json" \
          -u test:WElcome_123# \
          -d @test.json
   ```

   - The result like the following:

   ```
   {"env":{"defaultTimeZone":"GMT"},"items":[{"statementId":1,"statementType":"query","statementPos":{"startLine":1,"endLine":1},"statementText":"SELECT ename FROM emp WHERE deptno = :p_deptno ORDER BY ename","binds":[{"name":"p_deptno","data_type":"NUMBER","value":20}],"resultSet":{"metadata":[{"columnName":"ENAME","jsonColumnName":"ename","columnTypeName":"VARCHAR2","precision":10,"scale":0,"isNullable":1}],"items":[{"ename":"JONES"},{"ename":"SCOTT"}],"hasMore":true,"limit":2,"offset":2,"count":2},"response":[],"result":0}]}
   ```



## Enable Database API

ORDS database API is a database management and monitoring REST API embedded into Oracle REST Data Services. Depending on the database version and configuration, ORDS database API provides services such as manage pluggable databases, export data and review database performance. We already create a middle tier user in Lab5 with the user named **admin**, password is **WElcome_123#** with **SQL Administrator** role.

For example, execute the following command to use `TESTUSER` schema in the `ORCLPDB` database for the ORDS database API services:

```
GRANT DBA to TESTUSER;
GRANT PDB_DBA TO TESTUSER;
ORDS_ADMIN.ENABLE_SCHEMA(p_schema => 'TESTUSER');
```

The ORDS database API services are now ready for use. To list the tablespaces, send a `GET` request to

```
curl -u admin:WElcome_123# http://140.238.44.83:8888/ords/orclpdb/hr/_/db-api/stable/database/storage/tablespaces/
```

Because the when enable the ORDS the URL mapping to **hr** in the previous step.

An OpenAPI V3 document that describes the available ORDS database API services can be accessed at `http://140.238.44.83:8888/ords/orclpdb/hr/_/db-api/stable/metadata-catalog/openapi.json`.  All the ORDS database API services are made available.

## Clone the pdb

To enable the Pluggable Database (PDB) lifecycle management operations. Pluggable Database management is performed in the Container Database (CDB) and includes create, clone, plug, unplug and delete operations, the default CDB administrator credentials, `db.cdb.adminUser` and `db.cdb.adminUser.password` must be defined in the connection pool. Create the CDB administrator user and grant the SYSDBA privilege, set admin properties for the connection pool have already done in the Lab5.

1. Clone the pdb using the ORDS database API,

   - create a json file named request_body.json

     ```
     {
       "method": "CLONE",
       "clonePDBName": "pdbnew",
       "fileNameConversions": "NONE",
       "totalSize": "1000000000",
       "tempSize": "1000000000"
     }
     ```

     

   - Run the following command to clone the database

     ```
     curl -i -X POST -u admin:WElcome_123# -d @request_body.json -H "Content-Type:application/json" http://140.238.44.83:8888/ords/_/db-api/stable/database/pdbs/orclpdb/
     ```

   - The output result:

     ```
     {"env":{"defaultTimeZone":"GMT"},"items":[{"statementId":1,"response":["\nPL\/SQL procedure successfully completed.\n\n"],"result":0},{"statementId":2,"response":["\nPL\/SQL procedure successfully completed.\n\n"],"result":0},{"statementId":3,"response":["\nPL\/SQL procedure successfully completed.\n\n"],"result":0},{"statementId":4,"response":["\nPluggable database \"pdbnew\" altered.\n\n"],"result":0},{"statementId":5,"response":[],"result":0}]}
     ```

   2. Run the query from the new pdb:

      ```
      curl -s -k -X "POST" "http://140.238.44.83:8888/ords/pdbnew/hr/_/sql" \
             -H "Content-Type: application/sql" \
             -u test:WElcome_123# \
             -d $'SELECT * from emp;'
      ```

      You can see the new pdb is already ready for the REST API

      ```
      {"env":{"defaultTimeZone":"GMT"},"items":[{"statementId":1,"statementType":"query","statementPos":{"startLine":1,"endLine":1},"statementText":"SELECT * from emp","resultSet":{"metadata":[{"columnName":"EMPNO","jsonColumnName":"empno","columnTypeName":"NUMBER","precision":4,"scale":0,"isNullable":0},{"columnName":"ENAME","jsonColumnName":"ename","columnTypeName":"VARCHAR2","precision":10,"scale":0,"isNullable":1},{"columnName":"JOB","jsonColumnName":"job","columnTypeName":"VARCHAR2","precision":9,"scale":0,"isNullable":1},{"columnName":"MGR","jsonColumnName":"mgr","columnTypeName":"NUMBER","precision":4,"scale":0,"isNullable":1},{"columnName":"HIREDATE","jsonColumnName":"hiredate","columnTypeName":"DATE","precision":0,"scale":0,"isNullable":1},{"columnName":"SAL","jsonColumnName":"sal","columnTypeName":"NUMBER","precision":7,"scale":2,"isNullable":1},{"columnName":"COMM","jsonColumnName":"comm","columnTypeName":"NUMBER","precision":7,"scale":2,"isNullable":1},{"columnName":"DEPTNO","jsonColumnName":"deptno","columnTypeName":"NUMBER","precision":2,"scale":0,"isNullable":1},{"columnName":"COL1","jsonColumnName":"col1","columnTypeName":"NUMBER","precision":0,"scale":-127,"isNullable":1}],"items":[{"empno":7369,"ename":"SMITH","job":"CLERK","mgr":7902,"hiredate":"1980-12-17T00:00:00Z","sal":801,"comm":null,"deptno":20,"col1":null},{"empno":7499,"ename":"ALLEN","job":"SALESMAN","mgr":7698,"hiredate":"1981-02-20T00:00:00Z","sal":1601,"comm":300,"deptno":30,"col1":null},{"empno":7521,"ename":"WARD","job":"SALESMAN","mgr":7698,"hiredate":"1981-02-22T00:00:00Z","sal":1251,"comm":500,"deptno":30,"col1":null},{"empno":7566,"ename":"JONES","job":"MANAGER","mgr":7839,"hiredate":"1981-04-02T00:00:00Z","sal":2976,"comm":null,"deptno":20,"col1":null},{"empno":7654,"ename":"MARTIN","job":"SALESMAN","mgr":7698,"hiredate":"1981-09-28T00:00:00Z","sal":1251,"comm":1400,"deptno":30,"col1":null},{"empno":7698,"ename":"BLAKE","job":"MANAGER","mgr":7839,"hiredate":"1981-05-01T00:00:00Z","sal":2851,"comm":null,"deptno":30,"col1":null},{"empno":7782,"ename":"CLARK","job":"MANAGER","mgr":7839,"hiredate":"1981-06-09T00:00:00Z","sal":2451,"comm":null,"deptno":10,"col1":null},{"empno":7788,"ename":"SCOTT","job":"ANALYST","mgr":7566,"hiredate":"1987-04-19T00:00:00Z","sal":3001,"comm":null,"deptno":20,"col1":null},{"empno":7839,"ename":"KING","job":"PRESIDENT","mgr":null,"hiredate":"1981-11-17T00:00:00Z","sal":5001,"comm":null,"deptno":10,"col1":null},{"empno":7844,"ename":"TURNER","job":"SALESMAN","mgr":7698,"hiredate":"1981-09-08T00:00:00Z","sal":1501,"comm":0,"deptno":30,"col1":null},{"empno":7876,"ename":"ADAMS","job":"CLERK","mgr":7788,"hiredate":"1987-05-23T00:00:00Z","sal":1101,"comm":null,"deptno":20,"col1":null},{"empno":7900,"ename":"JAMES","job":"CLERK","mgr":7698,"hiredate":"1981-12-03T00:00:00Z","sal":951,"comm":null,"deptno":30,"col1":null},{"empno":7902,"ename":"FORD","job":"ANALYST","mgr":7566,"hiredate":"1981-12-03T00:00:00Z","sal":3001,"comm":null,"deptno":20,"col1":null},{"empno":7934,"ename":"MILLER","job":"CLERK","mgr":7782,"hiredate":"1982-01-23T00:00:00Z","sal":1301,"comm":null,"deptno":10,"col1":null}],"hasMore":false,"limit":10000,"offset":0,"count":15},"response":[],"result":0}]}
      ```

      