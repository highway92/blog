pg_dump로 생성한 snapshot의 복원방법을 알아본다.

## pg_restore 실행

```bash
pg_restore -d "USER:PASSWORD@HOST:PORT/DATABASE_NAME" -j 8 <snapshot_directory_path> <OPTION>
```

- -d : url 형식으로 연결 정보를 지정하는 옵션이다. 필자의 경우에는 highway92:highway92password@localhost:5432/restore_test 로 지정했다.
- -j 8 : 한번에 8개 테이블 씩 병렬로 작업한다. max_connection을 확인하자 사용가능한 connection이 최소 8 + 1개이어야 한다.
- snapshot_directory_name: snapshot 파일들이 저장되어있는 디렉토리 path를 입력한다
필자의 경우에는 /home/highway92/psql_dump

필자가 실제 실행시킨 command는 다음과 같다.

```bash
pg_restore -d "highway92:highway92password@localhost:5432/restore_test" -j 8 /home/highway92/psql_dump --create
```

## OPTION
--clean : 데이터 베이스에 기존에 존재하던 객체를 삭제하고 실행합니다. 즉 기존 데이터베이스의 모든 정보가 사라지고 dump로 대체됩니다.
--create : 데이터 베이스 전체를 복원할 때 사용합니다. pg_restore가 데이터베이스를 직접 생성한 후 복원작업을 시작합니다. 실행하는 유저에게 데이터베이스 생성 권한이 있어야 합니다.



## 선택적 복원
`pg_restore`를 사용하면 데이터베이스 스냅샷의 일부만 선택적으로 복원할 수 있습니다. 이 때문에 `pg_dump`로 전체 데이터베이스를 스냅샷한 후, `pg_restore`를 사용해 관심 있는 부분만 선택적으로 복원할 수도 있습니다.

### 파트별 플래그와 설명

| 파트    | 플래그                     | 설명                                                                                 |
|---------|----------------------------|--------------------------------------------------------------------------------------|
| 스키마  | `-s` `--schema-only`        | 스키마만 복원합니다.                                                                 |
| 데이터  | `-a` `--data-only`          | 테이블 데이터, 대용량 객체, 시퀀스 값을 복원합니다(백업 파일에 포함된 경우).           |
| 테이블  | `-t table` `--table=table`  | 테이블, 뷰, 머티리얼라이즈드 뷰, 시퀀스, 외부 테이블을 포함합니다. <br> 여러 `-t` 플래그로 여러 테이블을 지정할 수 있습니다. <br> 테이블 종속성 충족되지 않을 경우 오류가 발생할 수 있습니다. |
| 인덱스  | `-I index` `--index=index`  | 지정한 인덱스를 복원합니다. <br> 여러 `-I` 플래그로 여러 인덱스를 지정할 수 있습니다.  |
| 트리거  | `-T trigger` `--trigger=trigger` | 지정한 트리거를 복원합니다. <br> 여러 `-T` 플래그로 여러 트리거를 지정할 수 있습니다. |
| 섹션    | `--section=section_name`    | 지정한 섹션을 복원합니다. [pre-data, data, post-data] 중 하나일 수 있습니다. <br> 여러 `--section` 플래그로 여러 섹션을 지정할 수 있습니다. |
