---
title: "R에서 MySQL데이터의 접근"
author: Jinseog Kim 
---

## ODBC(Open DataBase Connectivity)

1. ODBC는 마이크로소프트가 만든 데이터베이스에 접근하기 위한 소프트웨어의 표준 규격
1. 사용자는 소스데이터의 DBMS에 상관없이 ODBC에 정해진 순서에 따라 데이터에 접근
1. 윈도우즈에는 기본적으로 OS를 설치할 때 함께 ODBC가 기본 설치
1. RODBC는 R에서 ODBC를 이용하여 데이터베이스에 접근할 수 있도록 Michael Lapsley와 Brian Ripley가
만든 R패키지
1. (R)ODBC는 데이터베이스 뿐만 아니라 엑셀파일의 접근에도 이용 



## ODBC connector설치 및 MySQL DSN설정
* 환경
    1. Ubuntu 16.04
    1. R version 3.4.1 (2017-06-30)
    1. MySQL server 5.7.19-0ubuntu0.16.04.1
    
* 필요 프로그램 설치 
    1. https://downloads.mysql.com/archives/c-odbc/
    1. 다운로드 및 압축해제 
    1. `lib`에 있는 라이브러리 파일들(`libmyodbc5w.so`)을 `/usr/lib/x86_64-linux-gnu/odbc/`에 복사 
    1. 기타 설치에 필요한 프로그램들 찾아서 설치 

## ODBC MySQL 드라이버 설정

```{r, engine='bash', eval=FALSE}
sudo vi /etc/odbcinst.ini
```

```
[MySQL]
Description=ODBC for MySQL
Driver=/usr/lib/x86_64-linux-gnu/odbc/libmyodbc5w.so
Setup=/usr/lib/x86_64-linux-gnu/odbc/libodbcmyS.so
FileUsage=1
UsageCount=4
```

## 참고 - MySQL 버전 체크
```{r, engine='bash', eval=FALSE}
$ mysqladmin -u root -p version
```

```
Enter password:
Server version          5.7.19-0ubuntu0.16.04.1
Protocol version        10
Connection              Localhost via UNIX socket
UNIX socket             /var/run/mysqld/mysqld.sock
Uptime:                 5 days 20 hours 37 min 58 sec
...
```

## MySQL DSN설정
* `odbc.ini`편집 
```{r, engine='bash', eval=FALSE}
sudo vi /etc/odbc.ini
```
```
[myodbc_mysql_dsn]  <---- DSN이름 
Description=MySQL DSN by Jskim
Driver=MySQL        <---- /etc/odbcinst.ini에서 지정한 이름 
Server=localhost
Port=3306
Socket=/var/run/mysqld/mysqld.sock
Database=
Option=3
ReadOnly=No
```
## ODBC 드라이버 및 MySQL DSN 적용
* ODBC driver 적용:
```{r, engine='bash', eval=FALSE}
sudo odbcinst -i -d -f /etc/odbcinst.ini
```
* DSN 적용:
```{r, engine='bash', eval=FALSE}
sudo odbcinst -i -s -l -f /etc/odbc.ini
```
* Test
```{r, engine='bash', eval=FALSE}
odbcinst -s -q
```



## RODBC를 이용한 MySQL 서버 접속 
```{r, eval=FALSE}
# install.packages("RODBC")
library("RODBC")
# MySQL 서버 접속 
channel <- odbcConnect('myodbc_mysql_dsn', uid='****', pwd='*****')
# 접속 종료
# close(channel) # ODBC connection 종료
```

## 데이터베이스 검색 
```{r}
sqlQuery(channel, "show databases")
#             Database
#1  information_schema
#2           dydrudwls
#3            gwak1494
#4               hkkim
```
## 테이블 검색 
```{r}
sqlQuery(channel, "show tables from jskim")
#  Tables_in_jskim
#1        customer
#2            iris
#3             pet
```

## 테이블 데이터의 조회

```{r, eval=FALSE}
sqlQuery(channel, 'select * from jskim.pet limit 5')
#    name   owner species sex      birth      death
#1 Bowser  Dianne     dog  NA 1998-08-31 1995-07-29
#2  Buffy  Harold     dog  NA 1989-05-13       <NA>
#3 Chirpy    Gwen    bird  NA 1998-09-11       <NA>
#4  Claws    Gwen     cat  NA 1994-03-17       <NA>
#5   Fang   Benny     dog  NA 1990-08-27       <NA>
```
## R데이터를 MySQL 테이블로 저장 
```{r}
sqlSave(channel, USArrests, tablename="jskim.USArrests", rownames = "state", addPK=TRUE)
sqlQuery(channel, "show tables from jskim")
#  Tables_in_jskim
#1       USArrests
#2        customer
#3            iris
#4             pet
close(channel) # ODBC connection 종료
```


## RODBC 함수
* Open connections to ODBC databases
```{r}
odbcConnect(dsn=, uid=, pwd=, ...)
```
* input
    1. dsn: data source name, 문자열
    2. uid: MySQL server접속 유저name, 문자열
    3. pwd: MySQL server접속 유저패스워드, 문자열
    4. readOnly: 읽기모드, logical
* output
    1. 연결 성공: connection handler
    2. 실패: -1 
* 예제
```{r}
db <- odbcConnect("DSN_NAME", uid="root", pwd="my_pass")
```

## RODBC 함수
* Close connections to ODBC databases
```{r}
odbcClose(channel) 
odbcCloseAll() 
```
* input
    1. channel: `odbcConnect`함수에 의해 반환된 RODBC 연결객체
* output :  성공/실패에 따라 TRUE/FALSE

```{r}
db <- odbcConnect("DSN_NAME", uid=, pwd=)
odbcClose(db)
```
## RODBC 함수
* ODBC databases에 다시 연결
```{r}
odbcReConnect(channel=) 
```
* input   
    1. `channel`: `odbcConnect`함수로 생성된 RODBC connection object 
* output 
    1. 재연결 성공: connection handler
    2. 실패: -1 
* 예제
```{r}
db <- odbcConnect("mydsn", uid="root", pwd="pass")
odbcClose(db)
channel <- odbcReConnect(db) 
```

## RODBC 함수
1. ODBC databases에서 데이터를 가져옴
```
sqlFetch(channel, sqtable, ..., colnames = FALSE, 
    rownames = TRUE)
```

2. SQL Query
```{r}
odbcQuery(channel, query, rows_at_time)
sqlQuery(channel, query, errors=TRUE, rows_at_time)
sqlQuery(channel=db, query="select * from stat.test")
# Delete or Drop Table in ODBC databases
sqlClear(channel, sqtable, errors = TRUE)
sqlDrop(channel, sqtable, errors = TRUE)
```
## RODBC 함수
3. SQL Query
```{r}
#List Tables on an ODBC Connection
sqlTables(channel)	
# Request Information about Data Types in an ODBC Database
sqlTypeInfo(channel, type = "all", errors = TRUE, as.is = TRUE)
# Write or update a table in an ODBC database.
sqlSave(channel, dat, tablename = NULL, append = FALSE,...)
sqlUpdate(channel, dat, tablename = NULL, index = NULL,...)
```


