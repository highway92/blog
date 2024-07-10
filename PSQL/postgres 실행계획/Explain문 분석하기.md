```sql
Gather Merge  (cost=14540.36..15486.59 rows=8110 width=27) (actual time=21.085..23.433 rows=10000 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Sort  (cost=13540.33..13550.47 rows=4055 width=27) (actual time=11.525..11.717 rows=3333 loops=3)
        Sort Key: created_at DESC
        Sort Method: quicksort  Memory: 671kB
        Worker 0:  Sort Method: quicksort  Memory: 197kB
        Worker 1:  Sort Method: quicksort  Memory: 202kB
        ->  Parallel Seq Scan on test  (cost=0.00..13297.33 rows=4055 width=27) (actual time=0.265..10.978 rows=3333 loops=3)
              Filter: (user_id = 15)
              Rows Removed by Filter: 330000
Planning Time: 0.184 ms
Execution Time: 23.713 ms
```
위 PSQL 실행계획과 결과를 해석할 수 있는가? 가능하다면 굳이 이 글을 볼 필요는 없다. 이 포스팅의 목적은 위의 암호문 같은 결과를 읽을 수 있을 정도의 지식 전달이다. 목차는 아래와 같다.

- Cost의 숫자는 어떤 걸 뜻하는가?
- psql의 Scan 종류 && 실행계획 보고 이해하기


&nbsp;
&nbsp;
&nbsp;


## Cost 이해하기
Explain 구문을 실행시키면 도처에 cost라는 단어가 있는걸 볼 수 있습니다. psql은 쿼리를 실행할 떄 각 세부 실행 계획의 비용(cost)들을 여러가지 조건들로 계산하고 가장 cost가 저렴한 실행계획을 세웁니다. 

이 cost의 기준이 되는것이 **"데이터 페이지 (8kb씩 나눠진 테이블의 한 영역) 하나를 주 기억장치(SSD, 하드디스크 등..) 메모리로 가져오는 작업을 1이라고 하자"** 하는 것입니다.

> Query step : 쿼리를 실행할 때 각 세부 실행 계획  
> Data Block : psql의 개념은 아니지만 다른 RDB에서 부르는 8KB씩 나눠진 테이블의 한 영역


아래 쿼리를 실행시켜보면 설정된 비용들을 확인해볼 수 있습니다. (PostgreSQL 15.6)
```sql
SELECT name, setting,short_desc FROM pg_settings WHERE NAME LIKE '%cost';
```


| name                    | setting       | short_desc                                                                                         |
|---------------------------|---------|-----------------------------------------------------------------------------------------------|
| cpu_index_tuple_cost       | 0.005   | 실행 계획기의 비용 계산에 사용될 인덱스 스캔으로 각 인덱스 항목을 처리하는 예상 처리 비용을 설정합니다.  |
| cpu_operator_cost          | 0.0025  | 실행 계획기의 비용 계산에 사용될 함수 호출이나 연산자 연산 처리하는 예상 처리 비용을 설정합니다.      |
| cpu_tuple_cost             | 0.01    | 각 튜플(행)에 대한 실행 계획기의 예상 처리 비용을 설정합니다.                                     |
| jit_above_cost             | 100000  | 쿼리 수행 예상 비용이 이 값보다 크면, JIT 짜깁기를 수행합니다.                                    |
| jit_inline_above_cost      | 500000  | 쿼리 수행 예상 비용이 이 값보다 크면, JIT 인라인 작업을 수행합니다.                                |
| jit_optimize_above_cost    | 500000  | 쿼리 수행 예상 비용이 이 값보다 크면, JIT-컴파일된 함수를 최적화합니다.                             |
| parallel_setup_cost        | 1000    | 병렬 쿼리를 위해 작업자 프로세스를 시작하는데 드는 예상 비용을 실행 계획기에 설정합니다.              |
| parallel_tuple_cost        | 0.1     | 각 튜플(행)을 작업자에서 리더 백엔드로 보내는 예상 비용을 실행 계획기에 설정합니다.                   |
| random_page_cost           | 4       | 비순차적으로 접근하는 디스크 페이지에 대한 실행 계획기의 예상 비용을 설정합니다.                     |
| seq_page_cost              | 1       | 순차적으로 접근하는 디스크 페이지에 대한 실행 계획기의 예상 비용을 설정합니다.                       |

&nbsp;
>좀 재미난 부분이 이 값들을 수정 할 수 있다는 겁니다.  
예를 들어 위의 parallel_setup_cost를 0.0001로 수정해버리면 psql은 병렬처리를 하는 것이 오히려 더 cost(실제 cost)가 많이 듦에도 불구하고 병렬처리를 자주 시도하게 됩니다.

조금 더 풀어서 설명해보자면 이렇습니다.  
- seq_page_cost의 cost는 1입니다.(Data Block을 주 기억장치로부터 메모리로 가져오는 cost와 같음)  
- random_page_cost는 4입니다.(seq_page_cost의 4배라고 생각할 수 있음.)
- parallel_setup_cost는 1000입니다.(병렬 실행 setup cost 자체가 크기 때문에 테이블 크기가 웬만큼 크지 않으면 병렬적으로 Query step 구성을 하지 않습니다.)

## Scan의 종류와 실행계획 보고 이해하기
실제 실행계획을 보면서 Scan종류를 설명하고자 한다. Scan의 종류는 크게 분류하면 아래의 5가지 이다.
- Sequential Scan(순차탐색)
- Index Scan
- Index Only Scan
- Bitmap Scan
- TID Scan

### 들어가기 앞서
쿼리를 직접 돌려가며 실행시켜 보는것을 추천한다.
사용한 테이블 생성 쿼리와 더미 데이터 생성 쿼리는 다음과 같다.
```sql
CREATE TABLE test (
	id SERIAL PRIMARY KEY,
	title VARCHAR(255),
	user_id integer,
	created_at timestamp
);
```

```sql
DO $$
 DECLARE
 	i	INTEGER := 1;
 BEGIN
 	WHILE i < 1000000 LOOP
 		INSERT INTO test(title, user_id, created_at)
 			VALUES(CONCAT('title', i), i % 100, NOW() + (RANDOM() * (INTERVAL '1 year')));
 		i := i+1;
 	END LOOP;
END $$;
```

- test table의 총 row는 1,000,000만 이다.
- user는 0~99까지이다.
- 각 user는 10,000개의 row를 생성했다.
- created_at은 현재 날짜기준 + 1년 이내이다.

&nbsp;

### Sequential Scan : 순차적인 탐색
```sql
EXPLAIN ANALYZE
SELECT * FROM test WHERE user_id = 15;
```

```sql
Gather  (cost=1.00..14271.63 rows=9733 width=27) (actual time=1.849..33.807 rows=10000 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on test  (cost=0.00..13297.33 rows=4055 width=27) (actual time=0.523..21.965 rows=3333 loops=3)
        Filter: (user_id = 15)
        Rows Removed by Filter: 330000
Planning Time: 0.347 ms
Execution Time: 34.224 ms
```

자 이제 해석해보자.

&nbsp;

```sql
Gather  (cost=1.00..14271.63 rows=9733 width=27) (actual time=1.849..33.807 rows=10000 loops=1)
```
- Gather : 병렬 작업의 결과를 모으는 단계임을 뜻한다.
- cost=1.00..14271.63: 이 단계의 추정 비용이다. 1.00은 시작비용 .. 뒤에 이어지는 14271.63은 총 비용을 뜻한다.
- rows, width : 예상되는 rows의 수 / 각 row의 예상 크기(27Byte)
- time=1.849..33.807 : 실제 소요된 시간이다. 1.849는 시작 시간 33.807은 종료시간이며 단위는 ms이다.
- rows , loops : rows는 실제 반환된 행 / loops 는 이 단계가 실행된 횟수이다.

&nbsp;


```sql
Workers Planned: 2
Workers Launched: 2
```
- Workers Planned, Workers Launched : 계획한 병렬 Worker의 수 / 실제 사용한 병렬 Worker의 수

&nbsp;


```sql
-> Parallel Seq Scan on test  (cost=0.00..13297.33 rows=4055 width=27) (actual time=0.523..21.965 rows=3333 loops=3)
```
- Parallel Seq Scan on test: test 테이블에 대한 병렬 순차 스캔을 의미한다. 각 병렬 작업자가 테이블의 일부를 스캔한다.
- (cost=0.00..13297.33 rows=4055 width=27) : 추정시작비용(0.00), 추정 총비용(13297.33), 추정 행 수 4055행, 각 행의 추정 크기 27byte
- (actual time=0.523..21.965 rows=3333 loops=3) : 실제 시작 시간 0.0523, 실제 총 시간 21.965, 실제 반한된 행 3333행, 3번의 이 단계 (Seq Scan) 수행


&nbsp;


```sql
Filter: (user_id = 15)
Rows Removed by Filter: 330000
```
- Filter: user_id = 15 조건을 만족하는 행만 선택.
- Rows Removed by Filter: 필터 조건에 의해 제거된 행 수 : 330000이 3번 버려졌으니 990000의 행이 버려졌다. 우리의 쿼리 결과 row인 10000과 합치면 총 1,000,000 즉 모든 row를 순회한 것이다.

**Rows Removed by Filter 키워드는 중요하다. 일반적으로는 이 수가 적을 수록 최적화된 쿼리이다. (쓸모없는 데이터를 순회하지 않는다는 걸 뜻하기 떄문에)**

&nbsp;


```sql
Planning Time: 0.347 ms
Execution Time: 34.224 ms
```
- Planning Time: 쿼리 계획을 세우는 데 걸린 시간
- Execution Time: 쿼리를 실행하는 데 걸린 전체 시간

### Sequential Scan 종합

이제 세세한 부분들은 살펴봤으니 중요한 부분들을 중점적으로 보자
- Rows Removed by Filter가 너무 크다. 우리는 여기서 WHERE 조건문을 좀 더 개선해야겠다고 생각할 수 있다.
- Parallel Seq Scan on test를 보고 순차탐색이 이뤄졌음을 알 수 있다. 만약 필요하다면 user_id에 index를 추가하여 index Scan을 하도록 유도할 수 도 있을 것이다.
- **sequential scan은 주로 index를 사용할 수 없는 경우에 사용되지만 꼭 그런건 아니다.** 자료량이 정말 적을 경우에는 index를 사용하는것이 더 비용이 들 수 도 있기 때문이다.

&nbsp;

### Index Scan
```sql
EXPLAIN ANALYZE
SELECT * FROM test WHERE id = 15;
```
```sql
Index Scan using test_pkey on test  (cost=0.42..8.44 rows=1 width=27) (actual time=0.011..0.012 rows=1 loops=1)
  Index Cond: (id = 15)
Planning Time: 0.172 ms
Execution Time: 0.035 ms
```
index를 탐색하는 방식의 Scan이다. 주로 조건문에 index가 생성된 column이 들어가있을 경우에 Index Scan을 사용한다.
하지만 꼭 그런것은 아니다 아래를 실행해보자

```sql
EXPLAIN ANALYZE
SELECT * FROM test WHERE id > 400000;
```

```sql
Seq Scan on test  (cost=0.00..19852.99 rows=601794 width=27) (actual time=10.511..37.969 rows=599999 loops=1)
  Filter: (id > 400000)
  Rows Removed by Filter: 400000
Planning Time: 0.045 ms
Execution Time: 49.054 ms
```
분명 id에는 index가 있음에도 index Scan을 사용하지 않는다. 이는 이 쿼리에서는 index Scan보다 Seq Scan의 비용이 적기 때문이다.

아래 쿼리는 어떨까?
```sql
EXPLAIN ANALYZE
SELECT * FROM test WHERE id > 600000;
```

```sql
Index Scan using test_pkey on test  (cost=0.42..14341.83 rows=400023 width=27) (actual time=0.023..29.785 rows=399999 loops=1)
  Index Cond: (id > 600000)
Planning Time: 0.058 ms
Execution Time: 37.146 ms
```
이번에는 index를 사용했다. 이처럼 psql은 필요 cost를 예측하여 최적으로 실행시킨다는걸 알 수 있다.

&nbsp;
&nbsp;

### Index only Scan
**Index only Scan은 인덱스에 필요한 데이터가 다 있을 경우에 사용되는 방식이다.**  
index Scan과 Index only Scan은 둘다 Index를 이용한 Scan이지만 큰 차이점이 있다. 우선 이를 이해하기 위해선 ctid라는것을 알아야한다. ctid는 psql에서 사용하는 가장 저수준의 단위다. 
```sql
SELECT ctid, * FROM test;
```
을 해보면 ctid를 볼 수 있는데 (number, number)의 형태일 것이다. 앞 숫자는 Data Block의 번호, 뒷 숫자는 Data Block에서 몇 번째 데이터인지 정도만 알고 있자.

그럼 이제 두 Scan의 차이점을 살펴보자.
#### index Scan
1. 인덱스 검색: 인덱스를 사용해서 조건에 맞는 인덱스 항목을 찾고 ctid를 얻는다.
2. ctid를 이용해서 테이블에서 row를 읽는다. (디스크 I/O발생)
3. 결과 반환

#### index only Scan
1. 인덱스 검색
2. 인덱스에서 데이터 읽기: 필요한 데이터가 모두 인덱스에 포함되어 있기 때문에 테이블에 접근해서 읽을 필요가 없다.(디스크 I/O 발생하지 않음)
3. 결과 반환

그래서 일반적으로는 index only scan이 비용이 더 적게든다고 할 수 있다. 하지만 또 꼭 그런것은 아닌데 다음의 쿼리들을 실행시켜보자.
```sql
EXPLAIN ANALYZE SELECT id FROM test WHERE id > 600000;
```
```sql
Index Only Scan using test_pkey on test  (cost=0.42..14341.83 rows=400023 width=4) (actual time=0.020..34.622 rows=399999 loops=1)
  Index Cond: (id > 600000)
  Heap Fetches: 399999
Planning Time: 0.051 ms
Execution Time: 41.847 ms
```

두번째 쿼리
```sql
EXPLAIN ANALYZE SELECT id, title FROM test WHERE id > 600000;
```
```sql
Index Scan using test_pkey on test  (cost=0.42..14341.83 rows=400023 width=15) (actual time=0.016..28.070 rows=399999 loops=1)
  Index Cond: (id > 600000)
Planning Time: 0.048 ms
Execution Time: 35.283 ms
```
분명 index only가 더 빨라야 하는데 왜 그럴까 이는 실행계획에 있는 Heap Fetches 때문이다. Heap Fetches가 쓰여있다는건 테이블을 참조했다는 뜻이다. 이는 VACUUM작업이 잘 되지 않아서 인데 (VACUUM은 다른 포스트에서 설명한다.) 이처럼 실행환경이나, 캐싱, 쿼리 실행시점등 다양한 요소들이 영향을 미치기 때문에 그냥 이론적으로는 그렇다고 이해하는 것이 좋다.

### Bitmap Scan
Index를 이용한 Scan에서 기억해야 하는 것이 index를 사용해서 tid를 찾으면 Data Block에 접근하여 row를 준비한다는 점이다. 그런데 한번 방문했던 Data Block을 다시 방문해야하는 일들이 생길 수 있는데 이를 해결하기 위한 Scan 방식이 Bitmap Scan이다.  
**그래서 Bitmap Scan은 주로 특히 넓은 범위 검색이나 여러 조건을 포함한 검색에 이용된다.**

#### 작동방식
1. 인덱스를 탐색하면서 조건에 맞는 자료는 1, 틀린 것은 0으로 하는 페이지 단위 Bitmap을 만든다.
2. 이 Bitmap과 각 페이지 ctid 정보가 담긴 부분을 AND 연산하여 결과를 뽑아낸다. (즉 Bitmap Scan도 index 기반이다.)

이게 어느상황에 Bitmap Scan을 쓸지는 psql이 정하는거라 억지로 한번 만들어봤다.  
아래 쿼리를 실행시켜보자
```sql
CREATE INDEX idx_user_id ON test (user_id);
CREATE INDEX idx_created_at ON test (created_at);

EXPLAIN ANALYZE
SELECT id, title, user_id, created_at
FROM test
WHERE user_id BETWEEN 10 AND 50
  AND created_at BETWEEN NOW() AND NOW() + INTERVAL '6 month';
```

```sql
Bitmap Heap Scan on test  (cost=10589.11..31750.85 rows=205257 width=27) (actual time=33.344..70.987 rows=204507 loops=1)
  Recheck Cond: ((created_at >= now()) AND (created_at <= (now() + '6 mons'::interval)))
  Filter: ((user_id >= 10) AND (user_id <= 50))
  Rows Removed by Filter: 295278
  Heap Blocks: exact=7353
  ->  Bitmap Index Scan on idx_created_at  (cost=0.00..10537.79 rows=502136 width=0) (actual time=32.753..32.754 rows=499785 loops=1)
        Index Cond: ((created_at >= now()) AND (created_at <= (now() + '6 mons'::interval)))
Planning Time: 0.195 ms
Execution Time: 74.986 ms
```

### TID Scan
tid Scan은 tid를 이용해서 Scan하는 것인데 위에서 설명한 ctid와 비슷하지만 다른 개념이다. ctid를 조건문에 넣어서 쿼리를 날리면 Tid Scan을 볼 수 있다.   ctid는 데이터의 물리적 위치를 나타내는 정보이므로 이를 직접적으로 이용하는 것으로 보인다.
글을 작성하면서도 잘 이해하지 못한 Scan방식이다. 자세한 내용은 공식문서를 참고하자 

```sql
EXPLAIN ANALYZE
SELECT ctid, * FROM test WHERE ctid = '(1,1)';
```
```sql
Tid Scan on test  (cost=0.00..4.01 rows=1 width=33) (actual time=0.005..0.006 rows=1 loops=1)
  TID Cond: (ctid = '(1,1)'::tid)
Planning Time: 0.090 ms
Execution Time: 0.035 ms
```

## 마무리
어쩐지 뒤로 내용이 뒤로 갈 수록 어려워 지기도 하고 필자가 이해를 못한 부분이 많기도 해서 용두사미가 된 거 같다.
그러나 기본적으로 실행계획을 볼 수 있는 정보를 얻어가기를 바라며 마지막으로 관련된 몇가지 팁을 남긴다.

- 필자는 실행계획 분석시에 GPT를 많이 사용한다. 실행계획을 던져주고 분석해달라고 하면 잘 해준다.(이 글 작성시에도 많은 도움을 받았다.)
- [실행계획 시각화 사이트](https://explain.dalibo.com/plan/g1f045gc11e1ab1g) 도 한번 활용해보자
- 실제로는 단순히 조건문을 수정하기보다는 index를 생성하거나 여러가지 튜닝 기법을 사용해야하는 경우가 많을 것이다.(필자도 쿼리 튜닝이나 분석에는 경험이 많지 않다.)