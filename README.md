# HBase
## Overview
* HBase는 NoSQL의 한 종류이다. 
* HBase는 분산 Database 이다.
* HBase는 사실 "Data Base" 보다는 "Data Store"이다. (RDBMS의 Feature인 culumns, secondary indexies, triggers, and advanced query languages 등 많은 부분이 부족하기 떄문)
* HBase는 선형적이고 모듈화된 scaling 위해 많은 기능을 제공한다.
* Strongly consistent reads/writes: HBase는 "eventually consistent" DataStore가 아니다. 이는 매우 빠른 counter aggregation이 가능하게 한다. 
* Automatic sharding: HBase tables는 Region을 통해 클러스터에 분산되어 있고 Region은 자동적으로 나누어지고 Data의 양이 늘어남에 따라 재 분산된다.
* Automatic RegionServer Failover
* Hadoop/HDFS Integration: HBase는 HDFS를 지원한다.
* MapReduce: HBase는 HBase를 활용하는 Source와 Sink로써 MapReduce를 통해 massively parallelized processing를 지원한다. 
* Java Client API: HBase는 프로그램적인 접근을 쉽게 Java API를 이용도록 지원한다.
* Thrift/REST


구글 Bigtable을 모델로 하여 개발된 분산 컬럼 기반 데이터베이스이다. Hadoop HDFS를 기반으로 구현되어 가용성과 확장성을 좋다. 컬럼 단위의 데이터 저장, 압축, 메모리 작업 bloom 필터 등을 제공한다. 

## HBase Keyword
### Column Family
* Colum들의 그룹으로 모든 Column Family의 Member는 같은 접두사를 사용
* 모든 컬럼패밀리 멤버는 물리적으로 파일시스템에 함께 저장
* 새로운 컬러페밀리 멤버는 동적으로 추가 가능
### Row key
* 임의의 바이트열로 사전순으로 내림차순 정렬
* 빈 바이트문자열은 테이블의 시작과 끝을 의미
* 문자열, 정수 바이너리, 직렬화된 데이터 구조까지 어떤 것도 로우키가 될 수 있음
### Cell
* 로우키 컬럼 버전이 명시된 튜플
* 값은 임의의 바이트열이며 타임스템프
* 테이블 셀은 버전관리 됨
### Table
Row들의 집합 (Row key가 있으며 다수의 column family로 구성)
schema정의서 컬럼 패밀리만 정의

## Sclability

## Architecture
![IMAGE](quiver-image-url/588FEB1390114EB860B93D8980903B28.jpg =550x361)

## 장단점
### 장점
성능이슈가 발생할 경우 계속 리전서버만 추가하면 성능유지
fail over가 쉽과 관리 용이
MapReduce의 input으로 사용하기 편함
### 단점
MapReduce의 inpu으로 사용하기 편하나 동시에 파일 Input과 같이 사용시 CPU사용률이 높아 Region서버가 쉽게 다운됨
특정 Region 서버에 특정 table의 Region이 집중됙 ㅣ쉬우며 이는 성능 저하로 이어짐. 구성시 HBase의 적절한 설계사 필요

## Operation
### Points
* HBase에 접근하는 주요 클라이언트 인터페이스는 HTable class이다
* 데이터를 변경하는 Row 단위 작업은 원자성이 보장된다.
* 일반적으로 HTable Interface는 Application을 시작할 때 단 한번만 생성한다.
### Put Method
#### Single Puts
* put instance를 생성하기 위해서는 row key를 제공해야 한다.
* HBase의 Row는 고유한 Row key로 식별하며, Rowkey 타입은 java 데이터타입인 byte array로 저장된다.
* Cloumn을 추가하려면 add()를 사용한다.
* 특정 cell이 존재하는지 알아보려면 has()를 사용한다.
* 클라이언트 코드에서 설정 파일에 접근하려면 HBaseConfiguration class를 사용한다.
#### KeyValue calss
* Row 단위가 아닌 특정 Cell 단위의 모든 정보를 반환한다.
* 좌표계처럼 Row key, Column qualifier, timestamp가 3차원 공간의 한 지점을 가리키는 모습이다.
* 주로 Key 데이터간의 비교/검증/복제 등에 사용한다.
* 저장공간을 최소하하여 효율적으로 데이터 저장하고, 빠른 데이터 연산을 제공하기 위해 Byte Array 타입을 사용한다.
#### List of puts
* 연산을 일괄로 한데 묶어서 처리한다.
* 리스트 기반의 입력은 서버 측에서 일력 연산이 적용되는 순서를 제어 할 수 없다.
* 데이터 입력 순서를 보장해야 하는 경우에는 작게 나눈 후 White cache를 명시적으로 Flush해야 한다.
### Get Method
#### Single gets
* 특정 Row 하나를 대상으로 수행되지만, row 내에서는 Column/Cell 제한이 없다.
* 버전 개수는 1로 최근 값만 리턴 받고, setMaxVersions()를 통하여 지정 가능하다.
#### Result class
* get()을 통하여 데이터를 읽어 들일 때 get 조건에 만족하는 모든 Cell을 담고 있는 Result class의 Interface를 반환한다.
* 서버에서 반환한 모든 Column family, Column qualifier, Timestamp 접근 수단을 제공한다.
#### List of Gets
* 하나의 요청으로 여러개의 Row를 요구할 때 사용된다.
* get instance와 동일한 배열을 반환, 예외발생 중 하나로만 동작한다.
### Delete Method
#### Single Deletes
* Delete class를 생성하려면 삭제 대상 Row Key를 입력해야 한다.
* Rowlock 파라미터를 추가 선택하여 동일한 Row를 두 번 이상 변경하고자 할 때 사용자 자신의 락을 설정한다.
* 전체 Family 및 그에 속하는 Column 삭제, Timestamp 지정이 가능하다.
#### List of Deletes
* put list 와 유사하게 동작한다.
* Remote 서버에서는 데이터가 삭제되는 순서를 보장할 수 없다는 것을 주의한다.
* Table.delete(deletes) 수행 시 실패한 작업은 deletes에 남게 되며, Exception 처리는 try/catch 구문을 이용한다.

##참고사항
* https://www.joinc.co.kr/w/man/12/hadoop/hbase/about
* http://bitnine.tistory.com/264
* http://jdm.kr/blog/189
* https://hbase.apache.org/book.html
* https://www.slideshare.net/madvirus/hbase-29278429
* http://blog.embian.com/10
