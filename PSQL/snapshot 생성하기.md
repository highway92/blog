데이터 베이스 스냅샷을 위한 pg_dump 기본 사용법에 대한 정리 

[참고](https://dbsnapper.com/blog/how-to-create-a-postgresql-database-snapshot)

필자의 환경

OS : WSL Ubuntu 20.04
Postgres Version : 12

Target Database : AWS RDS로 Hosting 중
Target Postgres Version : 14

## 버전 맞추기 
스냅샷을 시작하기전에 psql의 버전을 맞추는 것이 좋다. 필자의 로컬에는 12버전이, 대상 데이터베이스는 14버전의 psql을 사용하고 있기 때문에 pg_dump command가 오류를 일으킨다.
1. 로컬 버전 확인
```bash
pg_dump --version
```

2. Postgre 14 저장소를 추가한다.
```bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
```

1. 패키지 리스트를 업데이트한다.
```bash
sudo apt update
```

1. postgres 14 클라이언트를 설치한다.
```bash
sudo apt install postgresql-client-14
```

## dump를 시작하기 전
snapshot을 생성하기 전에 pg_dump에 대해 알아야 할 것
- pg_dump는 다른 사용자가 데이터베이스를 읽거나 쓰는 것을 block 하지 않습니다. 
- pg_dump는 SELECT 쿼리를 실행시킵니다. 그렇기에 최소한 SELECT에 대한 권한이 필요합니다.
- snapshot시 여러가지 형식을 지원하는데 plan-text 형식의 경우 psql 유틸리티를 사용해서 복원 할 수 있습니다. 아카이브 형식의 경우는 pg_restore command로 복원할 수 있습니다.


## dump 실행
바로 커맨드 부터 알아보자
```bash
pg_dump -d "postgres://USER:PASSWORD@HOST:PORT/DATABASE_NAME?sslmode=[require | disable]" -j 8 -Fd -f SNAPSHOT_DIRECTORY_PATH"
```

dump옵션은 너무 많기에 사용한 옵션들만 설명한다.
- -d "postgres://USER:PASSWORD@HOST:PORT/DATABASE_NAME" : 대상 데이터베이스에 대한 정보를 url형식으로 쓸수 있게 하는 옵션이다. ?뒤의 sslmode는 보안과 관련된 사항인데 필자의 경우는 사용하지 않았다.
- -j 8 : 병렬처리를 위한 옵션 한번에 8개 테이블씩 작업하겠다는 뜻이다. 그러니 최소한 connection이 8 + 1개 이상이어야 한다. max_connections 를 확인하고 사용하자.
- -Fd 스냅샷 작업의 출력형식 지정
- -f 스냅샷파일들이 저장되는 path를 입력한다.

필자가 사용한 command

```bash
pg_dump -d "postgres://highway92:highway92password@highway92-rds.dkcuqkj155792q.ap-northeast-2.rds.amazonaws.com:5555/highway_database" -j 8 -Fd -f /home/highway92/psql_dump"
```

