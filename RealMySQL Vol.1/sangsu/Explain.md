# 통계 정보

MySQL은 실행 계획을 수립하기 위해 테이블과 인덱스에 대한 개괄적인 정보를 갖고있다.
이러한 통계 정보를 5.7 버전까지는 정보를 직접 파악하거나 메모리로만 수집해 두었는데, 이러한 단점을 해결하기 위해 이후 버전부터는 히스토그램이 도입되었다. 또한 서버 재시작 시 휘발하는 것을 막기 위해 별도의 테이블로 히스토그램을 관리한다.

## 히스토그램

- 싱글톤 히스토그램(Singleton Histogram)
    - 각 값들의 분포도, 비율
    - 유니크한 값이 상대적으로 적을 때 사용됨
- 높이 균형 히스토그램(Equi-Height)
    - 컬럼 값의 범위에 대해 레코드 건 수 비율이 누적으로 표시

### 히스토그램 용도

범위별로 구분 된 분포값을 활용해 옵티마이저에게 최적의 실행계획을 수립하도록 돕는 기능
보통 인덱스 되지 않은 컬럼이 검색조건으로 활용될 때 히스토그램을 활용하게 된다.

## 코스트 모델

서버는 실행계획을 세울 때 작업의 코스트가 얼마나 들지 예측하고, 이를 토대로 실행계획을 잡는다

- 디스크로부터 데이터 페이지 읽기
- 메모리로부터 데이터 페이지 읽기
- 인덱스 키 비교
- 레코드 평가
- 메모리 임시 테이블 작업
- 디스크 임시 테이블 작업

또한, 5.7버전부터 코스트 모델의 가중치를 사용자가 설정할 수 있게 되었는데(이전엔 불가능) 두 분류로 구분된다

- server_cost
- engine_cost

## 실행 계획 분석

Explain analyze를 통해 쿼리의 depth별로 실행시간을 대략 확인이 가능하다

이는 실제 실행을 시켜서 확인하는 방식이며, MySQL 8.0.18버전부터 지원하는 기능이다.

**캐시 관련 쿼리**

서브쿼리는 실행 계획에서 나오는 select_type이 `SUBQUERY`인 경우 한번 조회 후 내부적으로 해당 쿼리를 계속 사용할 수 있도록 캐시해서 사용한다.

select_type이 UNCACHEABLE SUBQUERY 타입이 간혹 있는데 이 경우에는 서브쿼리가 캐시되지 않는 경우를 뜻한다.

**서브쿼리가 캐시되지 않는 경우**

- 사용자 변수가 서브쿼리에 사용된 경우
- NOT-DETERMINISTIC 속성의 스토어드 루틴이 서브쿼리 내에 사용된 경우
- UUID()나 RAND()와 같이 결괏값이 호출할 때마다 달라지는 함수가 서브쿼리에 사용된 경우

**실행 계획의 type 칼럼**

MySQL 에서는 해당 칼럼을 조인 타입으로 명시하였다.

- system : 레코드가 0건 or 1건 - InnoDB 스토리지엔진에서는 볼 수 없음
- const : PK나 유니크 키 칼럼을 이용하는 조건절 + 반드시 1건만 반환
- eq_ref : 조인되는 드리븐 테이블에서 PK나 유니크 키 칼럼의 검색 조건을 활용할 때 1:1로 매핑되는 경우
- ref : 동등 조건으로 비교되는 경우
- fulltext : Full-text Search 인덱스가 활용될 때
- ref_or_null : ref방식이거나 null 비교가 포함된 경우
- unique_subquery : 중복되지 않는 유니크 값이 반환될 때 해당 타입
- index_subquery : 중복된 값 포함된 서브쿼리
- range : 인덱스 레인지 스캔
- index_merge : 2개 이상의 인덱스를 이용해 검색 결과를 만들어낸 후 결과를 병합
- index : 인덱스 풀 스캔
- ALL : 테이블 풀 스캔

**key 칼럼**

실제 실행할 때 최종 선택될 인덱스 키 - 인덱스를 전혀 타지 못하면 해당 컬럼은 null로 표시

**key_len 칼럼**

다중 칼럼 인덱스에서 몇개의 키를 활용해서 값을 처리했는지를 알려줌 그런데 길이가 데이터 타입의 크기를 말하기 때문에 계산 잘 해야한다. ex) varchar(4) + INTEGER = 20

**ref 칼럼**

참조 조건으로 어떤 값이 제공됐는지 확인하는 컬럼
유심히 봐야할 점으로는 func이 들어갔을 때는 함수가 호출되는 경우이므로 지양하도록 준비해야 함

**rows 칼럼**

처리를 위해 어느정도의 row를 읽어올지에 대한 예측 값

**filtered 칼럼**

key에 있는 인덱스를 활용해 조인 후 버려지는 row의 비율

**Extra 칼럼**

- const row not found
- Deleting all rows
- Distinct
- FirstMatch
- Full scan on Null key
- Impossible HAVING
- Impossible WHERE
- LooseScan
- no matching row in const table
- No matching rows after partition pruning
- No tables used
- Not exist
- Plan isn’t ready yet
- Range checked for each record
- Recursive
- Rematerialize
- Select tables optimized away
- Start temporary, End temporary
- unique row not found
- Using filesort
- Using index(커버링 인덱스)
- Using index condition
- Using index for group-by
- 타이트 인덱스 스캔을 통한 Group by 처리
- 루스 인덱스 스캔을 통한 Group by 처리
- Using index for skip scan
- Using join buffer(Block Nested Loop), Using join Buffer(Batched Key Access), Using join Buffer(hash join)
- Using MRR
- Using sort_union, Using union, Using intersect
- Using where
- Zero limit
