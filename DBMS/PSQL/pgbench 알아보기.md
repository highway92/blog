# 들어가기 앞서
아래 글에서는 pgbench를 이용해서 테스트를 진행하는 방법을 다룬다.

## pgbench

### 들어가기 앞서
pgbench는 잘못 사용하면 의미 없는 숫자를 만들기 쉽다. 따라서 신뢰할 수 있는 결과를 내기 위한 몇가지 가이드라인이 필요하다.

1. 테스트 시간을 충분히 확보 : 적어도 몇 분 ~ 몇 시간의 테스트를 반복적으로 수행하여 결과가 일관적인지 확인해야 한다.

2. -s, -c 설정을 조심 : -c가 -s보다 크면 업데이트 쿼리시 충돌이 발생하고, 많은 트랜잭션이 block되어 동시성 충돌만 측정하는 꼴이 된다.

3. Vacuum 및 autovacuum이 테스트 결과에 영향을 줄 수 있다. 따라서 테스트를 실행 할때 총 업데이트 횟수와 vacuum이 언제 실행되는지 추적하면 더 정확한 결과를 얻을 수 있다.

4. 네트워크 지연이 낮아햐 한다. 네트워크가 병목이 되어 결과가 잘못 측정되지 않도록 네트워크 지연이 낮은 환경에서 테스트를 진행한다. 


### pgbench 실행

1. pgbench를 사용하기 위해서는 pgbench를 위한 테이블이 생성되어야 한다. 새로운 테스트용 데이터 베이스를 만드는걸 추천한다. 필자는 pgbenchtest라는 데이터 베이스를 만들었다. 유저는 postgres를 사용했다.

```SQL
CREATE DATABASE pgbenchtest;
```

2. pgbench를 실행시키기 위한 테이블을 초기화 한다. -i 옵션을 사용하면 되는데 terminal에서 아래의 커맨드를 사용해보자

```bash
pgbench -h localhost -U postgres -i -s 10 pgbenchtest
```
h 옵션은 database host 옵션이다. 로컬에서 실행중이라면 localhost 아니라면 대상 서버의 ip를 넣으면 된다.
U 옵션은 psql user 옵션이다. postgres유저를 사용했다.
i 옵션은 초기화 설정이다.(테이블 드랍, 테이블 생성, 더미 데이터 삽입, vacumming, 외래키 지정 등의 과정을 거친다.)
s 옵션은 dummy 데이터를 추가하는 옵션인데 default dummy의 수는 100,000개로 -s 10을 지정해주면 *10 인 1,000,000개의 row가 생성된다.

3. pgbench 실행시키기
```bash
pgbench -U postgres -c 32 -j 8 -T 60 pgbenchtest
```
U 옵션은 데이터 베이스 유저 지정옵션
c 옵션은 client의 수를 설정하는 옵션이다. max_connections와 연관이 있다. 만약 이 값이 max_connections보다 높다면 psql에서 에러가 발생할 것이다.
j 옵션은 스레드 지정옵션이다. 예시에서는 8로 지정해뒀는데 이 뜻은 동시에 8개의 질의가 병렬처리 될 수 있음을 뜻한다.
T 옵션은 시간 설정이다 예시에서는 60으로 지정되었는데 60초 동안 bench를 실행한다.
t 옵션은 트랜젝션의 갯수를 설정하는 옵션이다. 이는 T옵션과 동시에 사용될수 없다. 위의 예시에서 -t 10000으로 지정한다면 총 320000개의 트랜젝션을 하게 되는데 이는 client 수 * t 옵션수로 계산된다.

이외 에도 다양한 옵션들이 있으나 주로 사용하는 옵션이 위와 같다.
> 참고로 pgbench를 실행하게 되면 select, update, insert 쿼리가 랜덤으로 실행된다. 이에 대한 다양한 옵션들이 존재하지만 여기서는 생략한다.

4. 결과
실행시키면 아래와 같이 결과를 볼 수 있다.
```bash
pgbench -U postgres -c 32 -j 8 -T 60 pgbenchtest
number of transactions actually processed: 117458
latency average = 16.357 ms
tps = 1956.403744 (including connections establishing)
tps = 1956.471612 (excluding connections establishing)
```
- number of transactions actually processed: 이 숫자는 pgbench가 실제로 처리한 트랜잭션의 총 수. 즉, 벤치마킹 중에 처리된 트랜잭션의 수가 117,458건이라는 뜻.
- latency average : 평균 지연 시간. 트랜잭션을 처리하는 데 걸린 평균 시간이 16.357밀리초(ms)라는 의미
- tps(including connections establishing) : TPS(Transactions Per Second)는 초당 처리된 트랜잭션 수. 즉, 서버와의 연결을 설정하는 시간도 계산하여 초당 처리된 트랜잭션 수가 약 1956.4개라는 것을 나타냄.


## 실제 테스트
CPU : intel i7-12700 (16+4 쓰레드)
RAM : 40GB
Storage : SSD
psql version: 12

각 테스트를 실행하기전 pgbench -h localhost -U postgres -i -s 10 pgbenchtest 커맨드로 초기화를 진행해주었다.

### 테스트 1(default 옵션)
max_connections=100
work_mem=4MB
shared_buffers=128MB
wal_buffers=-1
maintenance_work_mem=64MB
effective_cache_size=4GB
effective_io_concurrency=1
random_page_cost=4.0

```bash
pgbench -U postgres -c 32 -j 8 -T 60 pgbenchtest
number of transactions actually processed: 117458
latency average = 16.357 ms
tps = 1956.403744 (including connections establishing)
tps = 1956.471612 (excluding connections establishing)
```

### 테스트 2
max_connections=40
work_mem=4MB
shared_buffers=8GB
wal_buffers=256MB
maintenance_work_mem=64MB
effective_cache_size=16GB
effective_io_concurrency=100
random_page_cost=1.0

```bash
pgbench -U postgres -c 32 -j 8 -T 60 pgbenchtest
number of transactions actually processed: 116220
latency average = 16.530 ms
tps = 1935.917815 (including connections establishing)
tps = 1935.991740 (excluding connections establishing)
```

## 결론
처음 pgbench를 접했을 때는 postgresql.conf의 optimizing 도구로 사용하려고 했으나, pgbench 만으로는 한계가 있다.

### 한계점
- 단순히 pgbench 점수만 보고 이 설정이 더 좋다라고 판단하는건 위험하다.
- 위의 테스트를 보면 shared_buffers, effective_cache_size 등에 드라마틱한 변화를 주었으나 테스트 결과는 거의 비슷하다. 이는 pgbench가 기본적으로 SELECT, INSERT, UPDATE가 섞인 단순한 트랜젝션을 반복하기만 할 뿐이라 그렇다.

### pgbench의 의미
- pgbench는 특정 설정이 성능에 긍정적 영향을 주는지 부정적영향을 주는지 빠르게 확인할 수 있다는 장점이 있다. 예를 들어 특정 설정을 했을 경우 cpu 사용량이 100%까지 치솟는다면 그 설정을 낮춰야 할 필요가 있을것이다. 
> 따라서 필자는 TPS와 더불어 cpu사용량, ram 사용량, storage 사용량을 유심히 보는 편이다.

- 필요에 따라서 custom query를 작성하고 이를 테스트하는 것도 가능하기 때문에 슬로우 쿼리를 테스트하고 개선하는데 사용할 수 도 있다.