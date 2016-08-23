# chapter 7 Database Access

## 1. 개요

해당 chapter 에서는 
clojure.java.jdbc library 를 사용하여 database 를 사용하는 방법을 이야기한다. 
또한 database 에 insert 된 데이터를 기반으로 PDF 를 생성하는 간단한 application 을 만들어본다.


## 2. Work with Relational Databases

클로저는 자바의 JDBC 로 access 가 가능한 database 에서 작업이 가능하다. 
> **ex:** MySQL, SQL Server, PostgreSQL, and Oracle

클로저에서 RDBMS 을 다루는 핵심 라이브러리는 clojure.data.jdbc 이다. 
> https://clojure.github.io/java.jdbc/

그리고 클로저에서 database libraries 대부분은 clojure.data.jdbc 를 기반으로 한다. 

- HugSQL : clojure source 와 SQL 을 분리하여 개발이 가능하도록 해준다.
- HoneySQL : (list, map, vector, set 과 같은) clojure data structure 를 사용하여 SQL 을 구현하게 한다.
이런 방식의 장점은 clojure 안에서 직접 쿼리를 생성하고 조합하는 것을 가능하게 한다. 
- SQL Korma : clojure DSL 을 사용하여 쿼리 작성을 가능하게 하고 back-end 에 특화된 SQL 을 생성해준다. 
(hibernate 의 dialect 개념과 QueryDSL 같은 개념을 말하는 듯) http://sqlkorma.com/

우선 처음에는 (프레임웍 없이 날 것의) 자바의 jdbc 를 기반으로 db 연동 sql 코드를 작성하듯이
clojure.data.jdbc 를 그대로 사용하여 clojure 에서 db 연동 sql 작성하는 법을 알아본다.

### 2.1. 사전 작업 

1. 로컬에 PostgresSQL 설치 - 생략

2. 다음의 sql 실행
``` 
- CREATE USER admin WITH PASSWORD 'admin';
- CREATE DATABASE REPORTING OWNER admin;
```

### 2.2. Access the Database

(당연하겠지만) database 에 access 하기 위해서는 project.clj 에 org.clojure/java.jdbc library 가 추가되어야 한다.
```
(defproject db-examples "0.1.0-SNAPSHOT" 
	:description "FIXME: write description" 
	:url "http://example.com/FIXME"
	:license {:name "Eclipse Public License"	
			  :url "http://www.eclipse.org/legal/epl-v10.html"} 
	:dependencies [[org.clojure/clojure "1.8.0"]
				   [com.layerware/hugsql "0.4.1"] 
				   [org.clojure/java.jdbc "0.4.2"] 
				   [org.postgresql/postgresql "9.4-1201-jdbc41"]])
```

또한 (실습에서 사용할) 네임스페이스 선언과 reference 를 선언한다. 
```
(ns db-examples.core
  (:require [clojure.java.jdbc :as sql]))
```

이제 (로컬에 설치한 postgresql) db 에 access 할 connection 을 정의한다. 
connection 정보를 정의하는 방법은 여러 가지가 있다.

1. Parameter Map : 가장 간단한 방법. 개발/스테이징/운영 에 맞는 맵을 선언 후 서버 deploy 시에 환경에 맞게 db connection 정보 맵을 선택한다.
2. Specifying the Driver Directly : JDBC DataSource 활용. 예시에서는 PGPoolingDataSource 제공
3. Defining a JNDI String : 실제 connection 정보는 application server 에 존재하고 이를 JNDI 방식으로 연결하여 사용하는 것

예시에서는 간단한 Parameter Map 방식을 보여준다.

```
(def db {:subprotocol "postgresql" 
		 :subname "//localhost/reporting"
		 :user "admin" 
		 :password "admin"})
```

#### 2.2.1. clojure.java.jdbc 활용

이제 실습이다. 
책에서는 clojure 를 활용하여 programmatically 하게 database 작업을 한다고 하면서 
기본적인 DDL, DML 함수를 선언하는 것을 보여주나 실제로 어떻게 동작을 시키는지는 제시하고 있지 않다.

https://pragprog.com/titles/dswdcloj2/source_code 에서 소스 코드를 다운 받은 후 
해당 프로젝트 내에서 repl 실행 후 load-file 을 기반으로한 다음과 같은 스크립트를 실행하였다. 

```
cd ~/clojure/db-examples/src
lein repl

user=>  (load-file "src/db_examples/core.clj")
user=>  (ns db-examples.core)
user=>  (drop-table! "users")
user=>  (create-users-table!)
user=>  (add-user! {:id "foo" :pass "bar"})
user=>  (add-users! {:id "foo1" :pass "bar"} {:id "foo2" :pass "bar"} {:id "foo3" :pass "bar"})
user=>  (get-user "foo")
user=>  (set-pass! "foo" "bar")
user=>  (remove-user! "foo")
```

#### Creating Tables
```
(defn create-users-table! [] 
	(sql/db-do-commands db
		(sql/create-table-ddl
      		:users
			[:id "varchar(32) PRIMARY KEY"] 
			[:pass "varchar(100)"])))
```
DDL 문장은 db 접속정보 맵을 가져야 하는 db-do-commands function 으로 wrapped 되었다. 
DB table 생성 시 column 은 dash 를 가질 수 없다. SQL syntax error 

#### Drop Tables 
```
(defn drop-table! [name]
  (sql/db-do-commands db
    (sql/drop-table-ddl name)))
```

#### Selecting Records
```
(defn get-user [id]
	(first (sql/query db ["select * from users where id = ?" id])))
	
ex) (get-user "foo")
```

#### Inserting Records
single insert
```
(defn add-user! [user] 
	(sql/insert! db :users user))

ex) (add-user! {:id "foo" :pass "bar"})
```
multiple insert
```
(defn add-users! [& users]
	(apply sql/insert! db :users users))

ex) (add-users!
	   {:id "foo1" :pass "bar"} 
	   {:id "foo2" :pass "bar"} 
	   {:id "foo3" :pass "bar"})
```

#### Updating Existing Records
```
(defn set-pass! [id pass] 
(sql/update! db :users
  {:pass pass} 
  ["id=?" id]))

ex) (set-pass! "foo" "bar")
```
vector = where,  map = updated row 를 표현한다.

#### Deleting Records 
```
(defn remove-user! [id]
	(sql/delete! db :users ["id=?" id]))

ex) (remove-user! "foo")
```
#### Transactions
트랜잭션을 지원한다. sql/with-db-transaction [t-conn db접속정보] 로 선언한 후, Tx 으로 묶을 각각의 쿼리에 t-conn 을 선언한다.
```
(defn transaction-example! []
  (sql/with-db-transaction [t-conn db]
    (sql/update!
        t-conn
        :users
        {:pass "bar1"} ["id=?" "foo1"])
    (sql/update!
        t-conn
        :users
        {:pass "baz"} ["id=?" "foo2"]))

ex) (transaction-example!) tx 으로 묶인 쿼리 전체가 성공해야만 commit, 아니면 rollback
```

#### 2.2.2. Use HugSQL

HugSQL 의 장점은 SQL query 와 clojure 코드를 분리하여 구현할 수 있다는 점이다. 
앞의 2.1. 의 예제에서는 쿼리 문장 전체, 또는 일부가 clojure 코드 안에 존재했다. 

def-db-fns macro 를 활용하여 따로 선언한 sql 파일을 읽어올 수 있다.
http://www.hugsql.org/#using-def-db-fns

```
(ns db-examples.hugsql
  (:require [db-examples.core :refer [db]]
            [clojure.java.jdbc :as sql]
            [hugsql.core :as hugsql]))

(hugsql/def-db-fns "users.sql")
```

HugSQL 에서 사용하는 sql 파일에서 comments 는 정의된 함수를 위한 metadata 역할을 한다. 
(특히) sql comments 의 name 항목은 해당 쿼리를 실행시키는 함수명을 나타낸다. 

ex) 프로젝트 root 의 resources 밑에 위치한 sql 파일 일부
db-examples/resources/users.sql 에서 정의된 함수를 실행한다.

```
cd ~/clojure/db-examples/src
lein repl

user=>  (load-file "src/db_examples/hugsql.clj")
user=>  (ns db-examples.hugsql)
user=>  (add-user! db {:id "foo" :pass "bar"})
user=>  (add-user-returning! db {:id "baz" :pass "bar"})
user=>  (add-users! db {:users [["bob" "Bob"] ["alice" "Alice"]]})
user=>  (find-user db {:id "bob"})
user=>  (find-users db {:ids ["foo" "bar" "baz"]})
user=>  (add-user-transaction {:id "foobar" :pass "I'm transactional"})
user=>  (add-user-transaction {:id "foobar" :pass "I'm transactional"})
```

#### Insert 
```
-- :name add-user! :! :n 
-- :doc adds a new user 
INSERT INTO users
(id, pass)
VALUES (:id, :pass)

ex) (add-user! db {:id "foo" :pass "bar"})) 
```

여기서 :! flag 는 해당 function 이 data 를 modifies 하는 것을 가리키고,
:n 는 해당 함수 실행으로 영향 받는 rows 수를 나타낸다. 
126 p 에 각각의 flag 에 대한 설명이 나온다. 

#### Insert And ID Returning
```
-- :name add-user-returning! :i :1
-- :doc adds a new user returning the id INSERT INTO users
(id, pass)
VALUES (:id, :pass)
returning id

ex) (add-user-returning! db {:id "baz" :pass "bar"}) 
```
retuning 은 database driver 에 dependent 하다. 안될 수도 있다는 뜻

#### Multiple Insert
```
-- :name add-users! :! :n 
-- :doc add multiple users 
INSERT INTO users
(id, pass)
VALUES :t*:users

ex) add-users! db 
	{:users
		[["bob" "Bob"]
		["alice" "Alice"]]})
```
:t* flag 는 multi insert 할 records vector 의 key 를 가리킨다. 

#### Select 
```
-- :name find-user :? :1
-- find the user with a matching ID SELECT *
FROM users
WHERE id = :id

ex) (find-user db {:id "bob"}) 
```

#### Multi Select 
```
-- :name find-users :? :*
-- find users with a matching ID 
SELECT *
FROM users
WHERE id IN (:v*:ids)

ex) (find-users db {:ids ["foo" "bar" "baz"]})
```

#### mix with clojure.java.jdbc - Transaction
```
(defn add-user-transaction [user] (sql/with-db-transaction [t-conn db]
    (if-not (find-user t-conn {:id (:id user)})
            (add-user! t-conn user))))
```

## 3. Generate Reports 

### 3.1. 개요
clj-pdf library 를 활용하여 database 의 data 기반으로 pdf report 를 생성한다. 
(책에서 제공하는) 예시 프로젝트 구현을 위해 다음의 단계를 거친다.

```
> lein new luminus reporting-example +postgres
> lein run migrate
```

> **비고**
> - 2016.08 기준으로 lein new luminus reporting-example +postgres 실행 시 책의 예제와 일부 다른 코드가 생성된다. 
책에서 제공하는 예시 코드를 다운로드 받아 진행한다. 

luminus 기반의 웹 프로젝트를 생성한다. 
luminus 는 database access 와 관련하여 conman lib 를 사용한다. 
conman 은 connection pooling 과 connect!, disconnect! funtion 을 제공한다.
또한 connection 의 lifecycle 을 관리한다. 

예전 발표의 예제에서는 embedded H2 database 를 사용했었지만 지금의 예제에서는 PostgresSql 을 사용한다.
따라서 외부 database 의 connection 정보를 담고 있는 datasource 생성을 위해 접속 정보가 담긴 map 등이 필요하다. 

database access 정보를 담은 map 의 선언과 설정이 끝나면 migration 을 실행한다. 
(reporting-example/profile.clj 참고 )
실행 시 프로젝트 루트의 resources/migrations 밑의 sql 이 동작한다. 

### 3.2. Serializing and Deserializing Data Based on Its Type

사실 문법을 세세하기 이해하진 못했다.
https://clojure.github.io/java.jdbc/#proto-section 와 책을 기반으로 이해해보면

### a. extend-protocol jdbc/IResultSetReadColumn
- database 의 조회 결과를 객체로 변환 (deserialize)
- clojure.java.jdbc 를 확인해보면...
```
> Protocol for reading objects from the java.sql.ResultSet 
> result-set-read-column: Function for transforming values after reading them from the database
```
- 추측하기로는, 클로저 내부적으로는 jdbc 를 통해 데이터를 조회한 후 리턴되는 java.sql.ResultSet 을 클로저에 맞게 변환하는 듯 

### b. jdbc/ISQLParameter set-parameter
### c. extend-protocol jdbc/ISQLValue
- database 에서 사용할 파라미터를 변환 (serialize)


### 3.3. 실습 - 준비
우선 repl 에서 다음의 스크립트 실행

```
user=> mount.core/start #'reporting-example.db.core/*db*)
user=> (in-ns 'reporting-example.db.core)
user=> (jdbc/insert! *db*
:employee
[:name :occupation :place :country]
["Albert Einstein", "Engineer", "Ulm", "Germany"]
["Alfred Hitchcock", "Movie Director", "London", "UK"] 
["Wernher Von Braun", "Rocket Scientist", "Wyrzysk", "Poland"] 
["Sigmund Freud", "Neurologist", "Pribor", "Czech Republic"] 
["Mahatma Gandhi", "Lawyer", "Gujarat", "India"]
["Sachin Tendulkar", "Cricket Player", "Mumbai", "India"] 
["Michael Schumacher", "F1 Racer", "Cologne", "Germany"])
``` 

mount.core 는 잘 모르겠음. 
> https://github.com/tolitius/mount
> http://www.luminusweb.net/docs/components.md 

```
user=> (conman/bind-connection *db* "sql/queries.sql")
user=> (read-employees)
user=> 결과 조회됨... 
```

### 3.4. 실습 - reporting generate web 띄우기

지난 발표에서 소개된 구조(luminus)를 따른다. handler, layout, middleware, routes 등
pdf 구현 중심으로만 간략히 설명한다. 

clj-pdf.core 의 pdf function 은 2개의 argument 를 가진다. 
하나는 document 를 표현하는 vector, 또는 input stream 이고, 다른 하나는 output file name, 또는 output stream 이다. 

다음을 확인해보자. 
```
(ns reporting-example.reports
  (:require [reporting-example.db.core :as db]
            [clj-pdf.core :refer [pdf template]]))

(pdf
[{:header "Employee List"}
  (into [:table
         {:border false
          :cell-border false
		  :header [{:backdrop-color [0 150 150]} "Name" "Occupation" "Place" "Country"]}
		  ]
	(employee-template (db/read-employees)))]
"report.pdf")

(def employee-template
  (template [$name $occupation $place $country]))
```

employee-template 에 선언된 template 는 input data 를 format 처리하는 macro 이다. 
(db/read-employees) 는 네임스페이스 reporting-example.db.core 의 read-employees funtion 을 말하고 
read-employees 는 resources/sql 밑의 select * from employees 쿼리를 나타낸다. 

db/core.clj 의 선언 
```
(conman/bind-connection *db* "sql/queries.sql")
```
http://www.luminusweb.net/docs/database.md

앞의 HugSQL 과 같은 방식으로 query 의 comment 에 function name 이 선언된다. 

reports.clj 를 정리하면 
pdf function 의 형식에 맞게 헤더 제목, 내용, output file 등이 정의 되고
특히 내용 부분에서는 core.clj 에서 db 와 connection 하여 생성된 function 을 활용하여 db data 를 조회하여 리턴한다.

나머지 layout 이나 웹 UI 구현 등은 예제 보면서 패스한다.



