# 들어가기 앞서
postgresql을 optimizing하기 위해선 테스트와 파라미터 튜닝이 필요하다. postgresql.conf의 기본값들은 어떠한 환경에서도 무난하게 실행될 수 있도록 구성되어져 있기 때문이다.
아래 글에서는 성능에 영향을 주는 파라미터들과 pgbench를 이용해서 테스트를 진행하는 방법을 다룬다.

[참조 블로그1](https://vladmihalcea.com/postgresql-performance-tuning-settings/)
[참조 블로그2](https://www.enterprisedb.com/postgres-tutorials/comprehensive-guide-how-tune-database-parameters-and-configuration-postgresql)


## 파라미터

### max_connections
역할 : postgresql서버에 동시에 접속할 수 있는 최대 클라이언트 연결수, 높게 설정할수록 좋을것 같지만 connection 자체가 메모리와 네트워크 리소스를 점유하기 때문에 메모리, cpu를 기준으로 최적값을 찾는게 필요함
설정권장 : 기본적으로는 물리코어 * 2 를 기본값으로 세팅하는것이 좋다고 생각한다. 8코어 cpu라면 최소 max_connection 16을 시작으로 조금씩 늘려가며 테스트를 진행한다.(connection pooling을 고려한 값 설정이 필요함)

### work_mem
역할 : 정렬, 해시 작업 등 메모리를 사용하는 연산(ORDER BY, DISTINCT, JOIN 쿼리 등)에서 각 작업이 사용할 수 있는 메모리 크기 설정
- 값이 너무 작으면 DISK I/O 발생이 잦고, 너무 높으면 메모리 점유가 높아진다.
- 이 값은 쿼리 내에서 각 작업 단위별로 적용되므로, 병렬 작업이 많은 환경에서는 신중히 설정해야한다.
- postgres wiki의 예시 => work_mem을 50MB로 설정 한후 30개의 client가 쿼리 요청을 하면 1.5GB의 메모리를 점유하게 된다. 게다가 쿼리에 8개 테이블의 merge sort가 포함되어있다면 8배를 곱해야한다..

설정권장 : default 4MB에서 +4MB씩 하면서 테스트를 진행한다. 워낙에 많은 요소들이 연관되어 있는 값이라 설정이 쉽지않다. postgresql.conf의 값은 전역 설정 값이며 각 연결 session, 쿼리, 유저에 맞게 work_mem을 세팅할 수 도 있다. 필자의 개인적인 전역설정 권장은 4MB or 8MB이다.

### shared_buffers
역할 : 데이터를 캐싱하기 위해 사용하는 메모리 영역의 크기, 말 그대로 캐시의 크기다
- default는 128MB로 되어있다. 이 역시 시스템 호환성을 위해 낮게 측정된 값이다.
설정권장 : postgres wiki에는 1GB이상의 RAM을 사용시 1/4 크기부터 설정하여 테스트하기를 권장하고 있다. ex) 16GB RAM => 4GB로 설정

### wal_buffers
역할 : 트랜잭션 로그(Write-Ahead Log)를 디스크에 기록하기 전에 메모리에 임시 저장하는 버퍼 크기
- WAL로그는 psql의 데이터 무결성과 복구를 위한 핵심 요소이다.
설정 권장 : shared_buffersdml 1/32 크기로 설정

### maintenance_work_mem
역할 : 인덱스 재작성, VACUUM, CREATE INDEX, ANALYZE, ALTER TABLE FOREIGN KEY 같은 유지보수 작업에서 사용할 수 있는 메모리 크기
설정권장 : 작업 빈도와 데이터베이스 크기를 고려해 기본값(64MB)에서 시작하고, VACUUM이나 인덱스 생성 작업이 느리다면 점차 값을 높여 테스트하는 것이 좋다.

### effective_cache_size
역할 : 데이터 베이스 서버가 디스크 데이터를 캐시하기 위해 사용할 수 있는 전체 메모리 크기 참조값 
- 이는 postgresql이 관리하는 shared_buffer + OS의 파일 시스템 캐시 입니다.
- postgresql에 쿼리 실행을 요청하면 쿼리 플래너가 각종 값들을 바탕으로 Sequential Scan을 할지 index Scan을 할지 실행계획을 세우게 되는데 이때 참조하게 되는 값이다.
- effective_cache_size 값이 크면 인덱스를 사용하는 계획을 선호 / 값이 작으면 Sequential Scan을 선호하게 된다.
- effective_cache_size 설정은 실제 메모리를 예약하거나 할당하지 않는다. 다만 값이 적절하지 않을 경우 쿼리 플래너가 잘못된 판단을 내려 실행속도가 느려질 수 있다.
설정권장 : 총 서버 메모리의 50%을 시작으로 테스트 하며 조절(최대 75%까지) ex) 총 메모리가 16GB라면 8GB로 설정해서 테스트

### effective_io_concurrency
역할 : 데이터베이스에서 디스크 I/O 작업을 동시에 수행할 수 있는 작업의 최대 값을 설정하는 파라미터
- 0으로 되어있을 경우 작업을 병렬처리 하지않는다. (하드디스크 처럼 디스크 성능이 낮거나 여러 I/O 작업을 수행 할 수 없는경우)
- 1 이상의 정수 : 설정된 값만큼 병렬로 처리 (SSD의 경우 200으로 설정해서 테스트를 통해 적절한 값을 찾는다.)

### random_page_cost
역할 : 쿼리 플래너가 쿼리 실행계획을 생성할때 참조하는 값 / 테이블 풀스캔을 할지 인덱스를 사용할지 같은 것들을 결정할때 참조되는 값이다.
- ssd를 사용할 경우에는 seq_page_cost나 random_page_cost가 별차이가 없다고 한다.
설정권장 : ssd를 사용할 경우에는 1 혹은 2로 hdd를 사용할 경우에는 그대로 두자.

### log_min_duration_statement 
역할 : 설정된 값보다 시간이 오래 걸리는 쿼리를 로그로 남기기위한 파라미터
- default는 -1로 되어있는데 이는 사용하지 않겠다는 뜻
- 필자의 경우 1000(단위 ms)로 설정하여 슬로우 쿼리를 로그하고 개선하는데 사용하고 있음.
