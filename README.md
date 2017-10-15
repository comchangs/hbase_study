# HBase

## Architecture 

### Overview
* HBase는 NoSQL의 한 종류이다. 
* HBase는 분산 Database 이다.
* HBase는 사실 "Data Base" 보다는 "Data Store"이다. (RDBMS의 Feature인 culumns, secondary indexies, triggers, and advanced query languages 등 많은 부분이 부족하기 때문)
* HBase는 선형적이고 모듈화된 scaling 위해 많은 기능을 제공한다.
* **Strongly consistent reads/writes**: HBase는 "eventually consistent" DataStore가 아니다. 이는 매우 빠른 counter aggregation이 가능하게 한다. 
* **Automatic sharding**: HBase tables는 Region을 통해 클러스터에 분산되어 있고 Region은 자동적으로 나누어지고 Data의 양이 늘어남에 따라 재 분산된다.
* **Automatic RegionServer Failover**
* **Hadoop/HDFS Integration**: HBase는 HDFS를 지원한다.
* **MapReduce**: HBase를 활용하는 Source와 Sink로써 MapReduce를 통해 massively parallelized processing를 지원한다. 
* **Java Client API**: HBase는 프로그램적인 접근을 쉽게 Java API를 이용도록 지원한다.
* **Thrift/REST API**: Java가 이닌 Front-end를 위해 Thrift와 REST를 지원한다.
* **Block Cache and Bloom Filters**: 고용량 쿼리 최적화를 위해 Block Cache와 Bloom Filter를 지원한다.
* **Operational Management**: JMX Metric에 더하여 추가적인 insight를 위해 Build-in web-page를 제공한다.

### Catalog Tables
Catalog table `hbase:meta`는 하나의 HBase table이고 HBase shell의 `list` 명령에서 제외 되어있다. 그러나 사실 다른 것들과 마찬가지로 단지 테이블이다.

#### hbase:meta
`hbase:meta` table은 Region들의 목록을 가지고 있고 `hbase:meta`의 위치는 Zookeeper에 저장되어 있다.

##### Key
* Reagion key of the format (`[table],[region start key],[region id]`)

##### Value
* `info:regioninfo` 해당 Region의 [HRegionInfo](http://hbase.apache.org/apidocs/org/apache/hadoop/hbase/HRegionInfo.htm) instance 가 나열됨
* `info:server` 해당 Region을 담고 있는 RegionServer의 port
* `info:serverstartcode` 해당 Region을 담고 있는 RegionServer process의 시작 시간

## Data Model
Data는 rows와 columns를 갖는 tables로 저장된다. Terminology가 RDBMS와 overlap되지만 완전히 같은 유사점이 아니다. 대신, HBase table은 multi-dementional map이라고 생각하는 것이 더욱 효과적으로 이해할 수 있다.

### Table
Table은 여러 Row들로 구성되어 있다.

### Column
Column은 콜론(:)나누어지는 Column family와 Column qualifier로 구성되어 있다.

### Column Family
Column family는 성능 관련 이유로 흔히 column과 그 값이 물리적으로 같이 존재한다.
각 Column family는 값들이 메모리에 캐싱되어야 하는지 data가 압축되거나 row key가 encoding되는지 등의 storage properties의 집합을 가지고 있다. 

### Column Qualifier
Column qualifier는 단편의 Data를 제공하기 위해 column family에 추가되어 있다. Column family "content"가 있다면 Column qualifier는 "content:html"나 "content:pdf"일 것이다. 비록 Column family는 table 생성 시 고정되지만, Column qualifier는 변할 수 있고, row들 사이에서도 매우 다들 수 있다.

### Cell
Cell은 값의 버전을 나타내는 row와 column family, column qualifier, 값과 Timestamp를 포함하고 있는 결합이다.
* Row key -> Coulmn Family -> Coulmn -> Version: Value
```
{row, column, version}
```

### Timestamp
각 값과 나란히 쓰여진 timestamp로 식별자로써 값의 주어진 버전이기도 하다. 기본적으로 timestamp는 RegionServer에서 data를 쓸 때 나타나는 시간이지만 data를 cell에 넣을 때 특정 timestamp 값을 지정할 수 있다.

### Row Key
Row Key는 Human readable하지 않은 바이트로 구성된 키이다.

### Row
단일 row에 대한 읽기/쓰기는 원자성이 보장한다.
테이블에서 가장 낮은 순서로 첫 번째로 표시되는 사전순 정렬된다.
빈 바이트 배열은 테이블의 네임 스페이스의 시작과 끝을 나타내는데 사용된다.
table은 동적으로 row key의 범위를 잘라서 파티셔닝(tablet)한다.

### Conceptual View
|Row Key|Time Stamp|ColumnFamily contents|ColumnFamily anchor|ColumnFamily people|
|-------|----------|---------------------|-------------------|-------------------------|
|"com.cnn.www"|t9||anchor:cnnsi.com = "CNN"||
|"com.cnn.www"|t8||anchor:my.look.ca = "CNN.com"||
|"com.cnn.www"|t6|contents:html = "<html>…​"|||
|"com.cnn.www"|t5|contents:html = "<html>…​"|||
|"com.cnn.www"|t3|contents:html = "<html>…​"|||
|"com.example.www"|t5|contents:html = "<html>…​"||people:author = "John Doe"|

### Physical View
|Row Key|Time Stamp|ColumnFamily anchor|
|-------|----------|-------------------|
|"com.cnn.www"|t9|anchor:cnnsi.com = "CNN"|
|"com.cnn.www"|t8|anchor:my.look.ca = "CNN.com"|

|Row Key|Time Stamp|ColumnFamily contents|
|-------|----------|---------------------|
|"com.cnn.www"|t6|contents:html = "<html>…​"|
|"com.cnn.www"|t5|contents:html = "<html>…​"|
|"com.cnn.www"|t3|contents:html = "<html>…​"|

### Namespace
RDB와 유사한 논리적인 그룹이다.
이 추상적 개념은 아래의 multi-tenancy 관련 기능의 기초가 된다.
* Quota Management: Namespace가 소비하는 자원의 양을 제한하다.
* Namespace Security Administration: tenants를 위한 보안관리의 다른 레벨을 제공한다.
* Region server groups: Namespace/table은 RegionServer의 하위 집합이라고 할 수 있어 거친 수준의 격리가 보장된다.

### Data Model Operation
* 기본적인 네가지의 model operation은 Get과 Put, Scan, Delete이다. Operration은 [Table](http://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Table.html) Instance를 통해 적용된다.
* 데이터를 변경하는 Row 단위 작업은 원자성이 보장
* 일반적으로 Table은 Application을 시작할 때 단 한번만 생성

#### Put Method

##### Single Puts
* put instance를 생성하기 위해서는 row key를 제공해야 한다.
* HBase의 Row는 고유한 Row key로 식별하며, Rowkey 타입은 java 데이터타입인 byte array로 저장된다.
* Cloumn을 추가하려면 add()를 사용한다.
* 특정 cell이 존재하는지 알아보려면 has()를 사용한다.
* 클라이언트 코드에서 설정 파일에 접근하려면 HBaseConfiguration class를 사용한다.

##### KeyValue class
* Row 단위가 아닌 특정 Cell 단위의 모든 정보를 반환한다.
* 좌표계처럼 Row key, Column qualifier, timestamp가 3차원 공간의 한 지점을 가리키는 모습이다.
* 주로 Key 데이터간의 비교/검증/복제 등에 사용한다.
* 저장공간을 최소하하여 효율적으로 데이터 저장하고, 빠른 데이터 연산을 제공하기 위해 Byte Array 타입을 사용한다.

##### List of puts
* 연산을 일괄로 한데 묶어서 처리한다.
* 리스트 기반의 입력은 서버 측에서 일력 연산이 적용되는 순서를 제어 할 수 없다.
* 데이터 입력 순서를 보장해야 하는 경우에는 작게 나눈 후 White cache를 명시적으로 Flush해야 한다.

#### Get Method

##### Single gets
* 특정 Row 하나를 대상으로 수행되지만, row 내에서는 Column/Cell 제한이 없다.
* 버전 개수는 1로 최근 값만 리턴 받고, setMaxVersions()를 통하여 지정 가능하다.

##### Result class
* get()을 통하여 데이터를 읽어 들일 때 get 조건에 만족하는 모든 Cell을 담고 있는 Result class의 Interface를 반환한다.
* 서버에서 반환한 모든 Column family, Column qualifier, Timestamp 접근 수단을 제공한다.

##### List of Gets
* 하나의 요청으로 여러개의 Row를 요구할 때 사용된다.
* get instance와 동일한 배열을 반환, 예외발생 중 하나로만 동작한다.

#### Delete Method

##### Single Deletes
* Delete class를 생성하려면 삭제 대상 Row Key를 입력해야 한다.
* Rowlock 파라미터를 추가 선택하여 동일한 Row를 두 번 이상 변경하고자 할 때 사용자 자신의 락을 설정한다.
* 전체 Family 및 그에 속하는 Column 삭제, Timestamp 지정이 가능하다.

##### List of Deletes
* put list 와 유사하게 동작한다.
* Remote 서버에서는 데이터가 삭제되는 순서를 보장할 수 없다는 것을 주의한다.
* Table.delete(deletes) 수행 시 실패한 작업은 deletes에 남게 되며, Exception 처리는 try/catch 구문을 이용한다.

## HBase Schema Design
* Table의 row key만 인덱스로 갖는다.
* Row들은 row key에 의해 사전식 정렬로 되어 있다.
* Row 수준의 모든 operation은 원자성(Atomic)을 갖는다.
* 읽기와 쓰기는 고르게 분산되어 있어야 한다.
* 일반적으로 단일 row에는 entity의 모든 정보를 갖는다.
* 관계가 있는 entity는 인접한 row에 저장한다.
* 빈 column은 공간을 소고 하지 않아 매우 많은 수의 column이 대부분의 row에 비어 있어도 괜찮다.

### Rowkey Design

#### Hotspotting

* Salting: 앞에 랜덤하게 문자를 삽입
* Hashing: 여러 값을 하나의 row key로 사용하는 겨우 유용하고, 예측가능한 길이의 값 을 얻을 수 있음
* Reversing the Key: 유사한 row의 데이터가 서로 인접하게 되고, 압축률이 높아짐

#### Monotonically Increasing Row Keys/Timeseries Data
순차적으로 증가되는 key를 부여한다면, 최신의 새로운 사용자가 더 활동적인 경향을 보이기 때문에 대부분의 트레픽이 한정된 수의 노드에 집중되는 결과를 초래할 수 있다. 이런 상황에선 뒤집어진 ID를 사용해 모든 노드로 데이터가 분산되도록 고려해야 한다.

#### Try to minimize row and column sizes
* Column Families: 가급적이면 이름을 짧게 하자. (예: data는 d)
* Attributes: 예로 myVeryImportantAttribute가 읽기 쉽더라도 via로 저장하는 것이 좋다.
* Rowkey Length: 키는 가능한  짧게 하자. (트레이드 오프 고려)
* Byte Patterns: 8바이트의 long을 저장하면 3x바이트가 된다.

#### Reverse Timestamps
일반적인 문제로 database processing에서 가장 최근의 값을 빠르게 찾는 것이다.
뒤집어진 timestamp를 키의 일부로 사용하는 technique은 이 문제를 푸는 특별한 사례이다. (Long의 최대 값에서 timestamp를 빼는 등의 방법)

####  Rowkeys and ColumnFamilies
Rowkey도 ColumnFamily은 범위에 있다. 그러므로 한 table에서 충돌이 없는 상태로 존재하는 ColumnFamily이다.

#### Immutability of Rowkeys
Row key는 변할 수 없고, "changed"는 삭제하고 다시 삽입하는 것이다.

## Apache HBase Coproccessors
Coprocessor framework는 data를 관리하는 RegionServer에 직접 작성한 커스텀 코드를 동작시키 우한 메커니즘을 제공한다.

### Overview
RDBMS는 SQL 쿼리를 쓰는 반면 HBase는 Data를 가져오기 위해 `Get` 또는 `Scan`를 사용한다. RDBMS에서 `WHERE`를 쓰는 것과 달리 적절한 data만을 가져오기 위해 HBase Filter를 사용한다.

Data를 가지고 와서 직접 계산을 실행한다. 이 페러다임은 몇 천개 row와 몇 column으로 이루어진 "Small data"는 잘 동작한다. 그러나 10억단위의 row와 백만단위의 column의 대량의 data가 network에서 보틀렉을 만들어 내고, client는 충분히 powerful하고 충분히 메모리를 가지고 대량의 데이터를 다루어야 한다.

비지니스 계산 코드를 coprocessor로써 집어넣어 RegionServer 위에서 data와 같은 위치로 계산하여 결과를 바로 client 에게 반환해 줄 수 있다. 

#### Coprocessor Analogies

##### Triggers and Stored Procedure
Observer coprocessor는 특정 event가 발생할 전후에 custom code가 실행되는 RDBMS의 trigger와 비슷하고 endpoint coprocessor는 RDBMS의 저장 프로시저와 비슷하다.

##### MapReduce
MapReduce는 계산이 Data가 있는 위치로 이동하는 원칙으로 동작한다. Coprocessor 역시 같은 원칙으로 동작한다. 

##### AOP
Aspect Oriented Programming (AOP)이 익숙하다면, Coprocessor를 마지막 도착지로 가기전에 요청을 가로채고 custom code를 실행하도록 적용하는 것으로으 볼 수 있다.

#### Coprocessor Implementation Overview
1. Coprocessor interface 중 하나를 구현
2. HBase Shell에 정적 또는 동적으로 coprocessor 로드
3. client-side code 에서 호출

### Type of Coprocessors

#### Observer Coprocessors
Event가 발생 전후에 Trigger되어 사용한다.
* **RegionObserver**: `Get`과 `Put` operation 같이 Region에서의 Event를 Observe하기 위해 사용
* **RegionServerObserver**: 시작 또는 정지, Merge, Commit, Rollback 등 RegionSercer의 운영에 관계된 Event를 Observe하기 위해 사용
* **MasterObserver**: Table 생성또는 삭제, 수정 등 HBase Master와 관계된 Event를 Observe하기 위해 사용
* **WalObserver**: Write-Ahead Log로 쓰기 것에 관계된 Event를 Observe하기 위해 사용

#### Endpoint Coprocessor
Endpoint Coprocessor는 Data의 위치에서 바로 계산을 실행할 수 있도록 사용한다.

## Reference
* https://hbase.apache.org/book.html
* https://cloud.google.com/bigtable/docs/schema-design
* https://www.slideshare.net/madvirus/hbase-29278429
* http://blog.embian.com/10
* https://www.joinc.co.kr/w/man/12/hadoop/hbase/about
* http://bitnine.tistory.com/264
* http://jdm.kr/blog/189
