---
title: 👨‍🔧 DMS cdc 실패 원인과 해결 과정을 기록하자(feat. binary log)
author: mingo
date: 2023-01-08 23:21:09 +0900
categories: [troubleshooting, etl]
tags: [dms, etl]
---

----

## 문제 발생
사 내 AWS DMS(AWS Database Migration Service)를 통한 DW(Data Warehouse) 구축 중에 발생하였던 이슈를 기록해보자.
DMS를 통해 연결된 데이터의 출발지인 source-target 즉, RDBMS를 접근하여 지정된 테이블을 가져와야하는데 특정 RDBMS에서는 변경 데이터 캡쳐(Replicate data changes only, cdc)가 정상적으로 동작하지 않고
실행과 동시에 뻗어버리는 현상이 발생했다. 전체 테이블 복사(Migrate existing data, fill-load)를 통해 이전 데이터를 긁어올 땐 이슈가 없었는데 실시간 성 데이터를 수집하려고 하니 발생한 문제였다. 

## 문제 원인
바로 CloudWatch를 통해 DMS 내부 로그를 확인해보니, Binary Log를 설정을 확인하라는 메시지를 확인하였다. 
MySQL의 바이너리 로그는 MySQL 3.22 버전부터 도입되었으며 DDL과 DML을 통해 데이터의 변화가 발생할 경우 해당 이벤트를 기록하는 로그 집합이다.
바이너리 로그는 `STATEMENT`, `ROW`, `MIXED` 세 가지 종류가 있고 각각의 특징은 다음과 같다.
 - `STATEMENT` : 질의문 기반 로깅 방식이고 질의문 상태로 로그에 저장되어 저장공간을 효율적으로 사용한다. (MySQL 5.7.6까지 default 설정)
 - `ROW` : 행 기반 로깅 방식이며 변경된 데이터가 바로 적용되기 때문에 더 적은 Lock을 점유하지만 쿼리결과에 의한 컬럼의 데이터를 저장하기 때문에 바이너리 로그 파일의 크기를 신경써주어야 한다.
 - `MIXED` : STATEMENT와 ROW 방식을 혼합한 로깅 방식(기본적으로 STATEMENT, 실행된 쿼리와 스토리지 엔진의 종류에 따라 ROW 포맷으로 전환됨)

변경 데이터 캡쳐를 수행할 때, DMS는 이러한 바이너리 로그를 바탕으로 데이터를 가져오기 때문에 해당 기능이 활성화 되어 있어야 하고 데이터 그 자체로 바라보니 ROW 설정이 되어 있어야 한다. 물론 전체 테이블 복사(full-load)의 경우
트랙잭션 로그를 참조하는 것이 아닌 테이블 자체를 보고 데이터를 가져오니 바이너리 로그가 활성화 되지 않아도 기능이 정상적이르 수행되는 것이였다.

>
- 참조
  : <https://dev.mysql.com/doc/refman/8.0/en/replication-options-binary-log.html>
  : <https://repost.aws/ko/knowledge-center/dms-binary-logging-aurora-mysql>
  : <https://docs.aws.amazon.com/ko_kr/dms/latest/userguide/CHAP_Source.MySQL.html#CHAP_Source.MySQL.Prerequisites>
  {: .prompt-tip }

## Oracle 설정
오라클도 Supplemental Log 설정을 통해 작업이 되어야 cdc가 정상적을 동작한다.

```bigquery
ALTER TABLE <table_name> ADD SUPPLEMENTAL LOG DATA (ALL) columns;
```
- 참조
 : <https://docs.aws.amazon.com/ko_kr/dms/latest/userguide/CHAP_Source.Oracle.html>
  {: .prompt-tip } 

## MySQL 설정
MySQL의 경우 서버의 `my.cnf` 설정 파일을 수정하는 방법이다. 해당 설정파일에 내용을 추가한다.

```text
// my.config
log-bin=/var/log/mysql_bin
binlog_format=ROW
```
 - log-bin : 바이너리 로그를 활성화하고 로그 파일 저장경로를 선택
 - binlog_format : 로그파일 형식

MySQL의 경우, 변경된 서버설정을 적용하려면 재기동이 필요하다.

![Desktop View](/assets/img/post/20230108/1.png){: width="400" height="400" }
_dba님 감사합니다_
![Desktop View](/assets/img/post/20230108/2.png){: width="400" height="400" }
_dba님 감사합니다_

## SQL Server 설정
SQL Server는 DMS cdc 기능을 사용하기 위해 추가적으로 작업할 것이 없다. 
SQL Server의 트랜잭션 로그(transaction log)는 데이터 변경 사항을 기록하고 있고 이것을 dms가 참조하기 때문이다.

## 마무리
설정 적용을 위해서는 DB 재기동이 필요하니 DBA님에게 커피 하나 사들고 가서 부탁드리자.
