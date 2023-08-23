# 1부 스토리지 엔진

- 데이터베이스 종류:
  - BerkeleyDB, LevelDB -> RocksDB, LMDB
  - limdbx, sophia, HaloDB
  - InnoDB, MyISAM, RocksDB
  - WirtedTiger, In-Memory, MMAPv1

## 데이터베이스 비교
- 데이터베이스 비교를 위한 변수:
  - 스키마와 레코드 크기
  - 클라이언트 수
  - 쿼리 형식과 접근 패턴
  - 읽기와 쓰기 쿼리 비율
  - 위 변수들의 변동폭
- YCSB(Yahoo! Cloud Serving Benchmark)
- TPC-C:
  - TPC(Transaction Processing Performance Council)
- SLA(Service-Level Agreement)
- 데이터베이스는 업그레이드 전략을 세워두는 것이 좋다.

## 장단점 비교
- 물리적 데이터 레이아웃 설계와 포인터 관리
- 직렬화 방식 및 데이터 가비지 컬렉션 방식 정의
- 데이터베이스 시스템의 시맨틱에 맞는 스토리지 엔진 구현
- 동시성 지원

## 1장 소개 및 개요
- OLTP(온라인 트랜잭션 처리) 데이터베이스
- OLAP(온라인 분석 처리) 데이터베이스
- HTAP(하이브리드 트랜잭션/분석 처리) 데이터베이스

### DBMS 구조
- Transport:
  - 클러스터 통신, 클라이언트 통신
- Query Processor:
  - Query Parser, Query Optimizor:
    - Query Parser:
      - https://github.com/mysql/mysql-server/blob/ea1efa9822d81044b726aab20c857d5e1b7e046a/sql/sql_yacc.yy
      - https://github.com/postgres/postgres/blob/master/src/pl/plpgsql/src/pl_gram.y
- Execution Engine:
  - Remote Exuector, Local Executor
- Storage Engine:
  - Transaction Manager, Lock Manager, Access Method, Buffer Manager, Recovery Manager


- https://dev.mysql.com/doc/dev/mysql-server/latest/CODE_PATH_CREATE_TABLE.html#CREATE_TABLE_PARSER

- 각 컴포넌트 설명:
  - Transaction Manager: 트랜잭션을 스케줄링하고 데이터베이스 상태의 논리적 일관성을 보장한다.
  - Lock Manager: 트랜잭션에서 접근하는 데이터베이스 객체에 대한 잠금을 제어한다. 동시 수행 작업이 물리적 데이터 무결성을 침해하지 않도록 제어한다.
  - Access Method: 디스크에 저장된 데이터에 대한 접근 및 저장 방식을 정의한다. 힙파일과 B-Tree 또는 LSM Tree 등의 자료구조를 사용한다.
  - Buffer Manager: 데이터 페이지를 메모리에 캐시한다.
  - Recovery Manager: 로그를 유지 관리하고 장애 발생 시 시스템을 복구한다.

### In-Memory DBMS vs Disk based DBMS
- 저장 매체가 다름에 따라 내부 자료구조 및 설계 및 최적화 방식도 모두 다르다.
- 메모리 제어가 디스크 제어보다 프로그래밍적으로 더 간단하기도 하다.
- NVM 스토리지 기술은 더 대중화되어야지 쓸수 있음...

#### 인메모리 데이터베이스의 지속성
- 지속성을 보장하지 않고 모든 데이터를 메모리에 저장하는 데이터베이스도 있기는 하다.
- 선행 기록 로그(write-ahead log)
- 로그 레코드는 일반적으로 배치 단위로 백업한다.:
  - 스냅숏, 체크포인트
- 디스크 기반 자료 구조는 넓고 낮은 트리, 인메모리 자료구조는 다양한 형태가 존재한다.

#### 칼럼형 DBMS 대 로우형 DBMS
- MySQL과 PostgreSQL 등 대부분의 전통적인 관계형 데이터베이스는 로우형 DMBS다.

##### 로우형 데이터 레이아웃
- 로우 단위로 저장하면 공간 지역성을 극대화할 수 있다.

##### 컬럼형 데이터 레이아웃
- 같은 컬럼끼리 디스크에 연속해 저장하는 방식
- 쿼리에 명시되지 않은 컬럼은 읽지 않아도 된다.
- 집계 분석 작업에 적합하다.

##### 차이점과 최적화 기법
- 컬럼형에서 CPU 벡터 연산을 통해서 한번에 많은 데이터를 처리할 수 있다.
- 자료형 별로 저장하면 압축률도 증가한다.
- 일반 쿼리와 범위스캔 요청이 많다면, 로우형 DBMS
- 많은 로우를 스캔하거나 일부 칼럼에 대한 집계 작업이 많다면, 칼럼형 DBMS

##### 와이드 칼럼 스토어
- 데이터를 다차원 맵으로 표현하고, 여러 칼럼을 칼럼 패밀리 단위로 저장한다.
- HBase 는 컬럼 기반인가?:
  - https://stackoverflow.com/a/11817030
  - https://deview.kr/2017/schedule/188
- 와이드 칼럼 스토어의 논리적 구조는 이해하기 쉽지만 실제 저장 방식은 복잡하다.

#### 데이터 파일과 인덱스 파일
- 저장 효율성: 데이터레코드의 저장 오버헤드를 최소화하는 방식으로 파일을 구성할 수 있다.
- 접근 효율성: 최소한의 단계로 원하는 레코드를 찾을 수 있다.
- 갱신 효율성: 디스크 쓰기를 최소화하는 방식으로 레코드를 갱신할 수 있다.

##### 데이터파일
- 인덱스 구조형 테이블(IOT, Index-Organized Table):
  - 인덱스에 실제 레코드를 저장, 데이터는 키 순서로 정렬, 범위스캔 가능
- 힙 구조형 테이블(Heap-Organized Table):
  - 대체로 삽입 순서대로 저장, 별도 인덱스 필요
- 해시 구조형 테이블(Hash-Organized Table):
  - 해시 값에 해당하는 버킷, 삽입 순서대로 또는 키 순서로 정렬 저장하여 조회속도 향상

- 인덱스에 데이터 레코드를 저장하면 디스크 탐색 횟수를 줄일수 있다.

##### 인덱스 파일
- 기본 인덱스, 보조 인덱스
- 보조 인덱스는 데이터 레코드를 직접 가르키거나 해당 레코드의 기본키를 저장한다.
- Clustered Index, Non-Clustered Index

##### 기본 인덱스를 통한 간접 참조
- 데이터 레코드를 직접 참조해야하는지, 기본키 인덱스를 통해 접근해야 하는지 의견이 갈린다.
- 레코드를 갱신할때 비용을 낼것인지, 레코드를 조회할때 비용을 낼것인지
- 두가지 모두를 사용해서, 오프셋을 사용해 조회하고 오프셋이 유효하지 않다면 기본키 인덱스로 조회한다.

##### 버퍼링과 불변성, 순서화
- 버퍼링: 데이터를 디스크에 쓰기 전에 일부를 메모리에 저장하는 것을 의미한다.
- 가변성: 파일 일부를 읽고 갱신한 뒤에 똑같은 자리에 다시 쓸지에 대한 여부를 나타내는 속성이다.:
  - COW
- 순서화: 디스크 페이지에 데이터 레코드를 키 순서로 저장하는 것을 의미한다.


## 2장 B-트리 개요
### 디스크 기반 스토리지용 트리
- 메모리 기반 트리를 사용하지 못하는 이유:
  - 지역성: 자식노드와 부모노드가 가까운 위치에 있지 않을수 있다.
  - 트리의 높이, 팬아웃: 트리의 높이가 곧 디스크 접근 횟수를 결정한다.
- 따라서 디스크 저장에 적합한 트리는:
  - 인접한 키의 지역성을 높이기 위한 높은 팬아웃
  - 트리 순회 중 디스크 탐색 횟수를 줄이기 위한 낮은 트리 높이

### 디스크 기반 자료구조
- HDD, SSD:
  - 개발자를 위한 SSD: https://tech.kakao.com/2016/07/13/coding-for-ssd-part-1/
  - 수명 관련: http://www.kpug.kr/kpugknow/420143
- 가장 작은 작업 단위: 블록:
  - 심지어, cpu 레벨에서 자기 멋대로 실행하는 것때문에 디스크에 똑바로 안들어갈수도 있다.
  - https://stackoverflow.com/questions/50307693/does-an-x86-cpu-reorder-instructions

- B-트리 계층:
  - 루트 노드: 트리의 최상위 노드로 부모노드가 없음
  - 내부 노드: 루트와 리프 노드를 연결하는 모든 노드, 트리에는 일반적으로 한 레벨 이상의 내부 노드가 있음
  - 리프 노드: 자식 노드가 없는 트리의 최하위 계층 노드
- B+-트리: 사실상 흔히 말하는 B-Tree:
  - 리프노드에 값을 저장하기 때문에 모든 작업(삽입, 업데이트, 삭제, 검색)은 리프노드에만 영향을 미치며, 상위 레벨의 노드는 노드 분할 혹은 병합이 일어날 때에만 영향을 받는다.

- B-트리 알고리즘은 생략, 필요시 책 확인
