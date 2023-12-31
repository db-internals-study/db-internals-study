## 6장 B-트리의 변형
- 트리 구현 기법:
  - Copy-On-Write B-Tree: 인플레이스 업데이트 지원하지 않는다.
  - Lazy B-Tree: 동일한 노드에 대한 연속된 쓰기 작업의 I/O 요청 횟수를 줄이기 위해 수정 내용을 버퍼에 저장
  - FD-Tree: LSM 와 유사한 버퍼 메커니즘을 사용한다. 버퍼가 가득차면 불변 형태로 기록, 상위레벨에서 하위레벨로 전파
  - Bw-Tree: 여러 노드에 대한 쓰기 작업을 배치 단위로 처리해 비용을 낮춘다.
  - Cache-Oblivious B-Tree: 인메모리 자료구조처럼 사용한다.

### 쓰기 시 복사
- 장점: 단순함, 래치가 필요 없어 리더를 위해 동기화하지 않아도 된다.
- 단점: 더 많은 메모리가 필요하다.

#### 쓰기 시 복사 방식 구현: LMDB
- Lightning Memory-Mapped Database
- OpenLDAP 프로젝트에서 사용하는 키-값 데이터베이스
- 페이지 캐시와 선행 기록 록, 체크포인트, 컴팩선을 사용하지 않는다.
- 모든 쓰기 작업은 루트에서 시작한다.
- 이전 트리를 참조하는 읽기 작업이 끝나는 즉시 페이지를 회수하고 재사용할 수 있다.
- 업데이트 시 루트에서 리프노드까지의 경로의 모든 노드를 복사한다.
- 최신 버전과 변경 사항이 커밋될 버전이다.

### 노드 업데이트 추상화
- 메모리에 저장된 노드에 접근하는 방법:
  - 캐시된 버전에 바로 접근하는 방법:
    - 대부분 페이지 캐시가 관리하는 메모리 영역을 가리키거나 메모리 매핑을 사용한다
  - 기반 언어로 인메모리 객체를 생성하는 방법
  - 래퍼 객체를 사용하는 방법

### 지연형 B-트리
#### 와이어드 타이거
- MongoDB 기본 스토리지 엔진
- 업데이트 버퍼는 읽기 작업시 접근된다. 버퍼된 내용과 원본 디스크 페이지를 합쳐서 가장 최신 데이터를 반환한다.
- 업데이트 버퍼는 스킵리스트를 기반으로 한다.
- 와이어드타이거의 가장 큰 장점은 페이지 업데이트와 구조 변경은 백그라운드 쓰레득 ㅏ처리하기 때문에 읽기와 쓰기 작업은 다른 쓰레드가 완료될때까지 기다릴 필요가 없다.

### 지연 적응형 트리
- Lazy-Adaptive Tree (LA-Tree)
- 새로운 데이터 레코드를 삽입할 때 우선 루트 노드의 업데이트 버퍼에 저장
- 가득차면 하위 레벨 버퍼로 복사 및 이동해 공간 확보
- 계단식 버퍼

- 버퍼링 방법은 결국 추가적인 인메모리 버퍼 탐색과 원본 데이터와의 병합/조정 단계가 필요하다.

### FD-Tree
- 버퍼링은 소규모 랜덤 쓰기를 피하게 해준다.
- LSM 트리와 비슷한 방식으로 데이터를 인덱싱 한다.
- 작은 가변 헤드 트리와 여러 개의 정렬된 불변 배열로 구성된다.

#### 부분적 캐스케이딩
- Fractional Cascading
- Gap 을 최소화 하기 위해 인근 레벨의 배열을 브리지를 통해 연결해 레벨 사이에 지름길을 만든다.

#### 로그 배열

### Bw-Tree
- Write amplification
- Space amplification
- 동시성 문제와 래치 사용의 복잡성
- Buzzword-Tree

#### 체인 업데이트
- 변경사항과 원본 노드를 따로 저장한다.
- 변경사항은 체인을 형성한다. 델타노드
- 델타노드에는 삽입, 업데이트, 삭제를 모두 포함단다.
- 원본가 델타노드의 크기는 페이지 크기와 일치하지 않을 가능성이 높기 때문에 연속된 공간에 저장하는 것이 합리적이다.
- 논리를 물리적 개체가 아닌 논리적 개체로 사용하는 방식

#### CAS 연산으로 동시성 문제 해결
- 노드 업데이트 알고리즘 단계:
  1. 루트 노드에서 리프 노드까지 순회하면서 대상 논리적 리프 노드를 찾는다. 매핑 테이블에는 원본 노드 또는 업데이트 체인에서 가장 최신 델타 노드를 가리키는 가상 링크를 저장한다.
  2. 1단계에서 찾은 원본 노드를 가르키는 새로운 델타 노드를 생성한다.
  3. 2단계에서 생성한 델타 노드를 가리키는 포인터를 매핑 테이블에 업데이트 한다.
- 3단계를 CAS 연산으로 작업

#### 구조 변경 작업
- SMO: Structure Modification Operation
- 분할 SMO:
  1. 분할 대상 노드의 원본 노드에 델타를 반영해 논리적 상태를 최신 상태로 업데이트하고 분할 지점의 오른쪽에 새로운 페이지를 추가한다.
  2. 분할: 분할델타를 끝에 추가
  3. 부모 노드 업데이트
- 병합 SMO:
  1. 형제 노드 제거
  2. 병합
  3. 부모 노드 업데이트

#### 노드 통합과 가비지 컬렉션
- 델타 체인의 길이를 적당하게 유지하기 위해
- 에포크 기반의 교정 기법

#### Bw 트리의 장단점
- 쓰기 증폭 문제를 해결
- Non blocking 액세스와 캐시 친화적

#### 캐시 비인지형 B-트리
- Cache-Oblivious 자료구조

#### 반 엠데보아스 레이아웃

