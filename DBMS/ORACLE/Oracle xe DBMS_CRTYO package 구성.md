Docker container 기준으로 설명합니다.


1. docker ps를 해서 container Id를 알아냅니다.
```bash
docker ps
```

2. container 내부 터미널로 붙습니다.
```bash
docker exec -it ${CONTAINER_ID} /bin/sh
```

3. sysdba 권한으로 oracle db process에 접속합니다.
```bash
sqlplus "/ as sysdba"
```

4. DBMS_CRYPTO 패키지 추가
```sql
@$ORACLE_HOME/rdbms/admin/dbmsobtk.sql
@$ORACLE_HOME/rdbms/admin/prvtobtk.plb
```

5. package 사용권한 추가
```sql
grant execute on dbms_crypto to ${DB_USER_NAME};
```

6. package 선언(declare)
```sql

CREATE OR REPLACE PACKAGE pkg_crypto
IS
  FUNCTION encrypt (
    input_string IN VARCHAR2,
    key_data     IN VARCHAR2 := 'test202309120000'
  ) RETURN RAW;

  FUNCTION decrypt (
    input_string IN VARCHAR2,
    key_data     IN VARCHAR2 := 'test202309120000'
  ) RETURN VARCHAR2;
END pkg_crypto;
/
```

7. package body 선언(구현부)
```sql

CREATE OR REPLACE PACKAGE BODY pkg_crypto
IS
  -- 에러 발생시에 error code 와 message 를 받기 위한 변수 지정.
  SQLERRMSG VARCHAR2(255);
  SQLERRCDE NUMBER;
  
  -- 암호화 함수 선언 (key_data default 값: 12345678)
  FUNCTION encrypt (input_string IN VARCHAR2, key_data IN VARCHAR2 := 'test202309120000')
    RETURN RAW
  IS
	-- 평문 암호화키 RAW 변환
	key_data_raw  RAW(40) := UTL_I18N.STRING_TO_RAW(key_data, 'AL32UTF8');
    input_raw     RAW(4000);
    encrypted_raw RAW(4000);
	-- AES 알고리즘, CBC 체인, PKCS5 패딩방식 사용
	AES_CBC_PKCS5 CONSTANT PLS_INTEGER := DBMS_CRYPTO.ENCRYPT_AES128 
                                        + DBMS_CRYPTO.CHAIN_CBC 
                                        + DBMS_CRYPTO.PAD_PKCS5;

  BEGIN
    IF input_string IS NULL THEN
      RETURN NULL;
    END IF;
	
    -- 암호화 대상 data RAW 변환
    input_raw := UTL_I18N.STRING_TO_RAW(input_string, 'AL32UTF8');
    encrypted_raw := DBMS_CRYPTO.ENCRYPT(
                     src => input_raw,
                     typ => AES_CBC_PKCS5,
                     key => key_data_raw);
  RETURN encrypted_raw;

  EXCEPTION
    WHEN OTHERS THEN 
    RETURN input_string;
  END encrypt;

  FUNCTION decrypt (input_string IN VARCHAR2 , key_data IN VARCHAR2 := 'test202309120000')
    RETURN VARCHAR2
  IS
    -- 평문 암호화키 RAW 변환
    key_data_raw     RAW(40) := UTL_I18N.STRING_TO_RAW(key_data, 'AL32UTF8');
	decrypted_raw    RAW(4000);
    decrypted_string VARCHAR2(4000);
    -- AES 알고리즘, CBC 체인, PKCS5 패딩방식 사용
    AES_CBC_PKCS5 CONSTANT PLS_INTEGER := DBMS_CRYPTO.ENCRYPT_AES128 
                                        + DBMS_CRYPTO.CHAIN_CBC 
                                        + DBMS_CRYPTO.PAD_PKCS5;

  BEGIN
	IF input_string IS NULL THEN
      RETURN NULL;
    END IF;	
    
    decrypted_raw := DBMS_CRYPTO.DECRYPT(
           src => input_string,
           typ => AES_CBC_PKCS5,
           key => key_data_raw);
    
	-- 복호화한 RAW 데이터 UTF-8 형식의 문자열로 변환
    decrypted_string := UTL_I18N.RAW_TO_CHAR(decrypted_raw, 'AL32UTF8');    
    RETURN decrypted_string;

  EXCEPTION
    WHEN OTHERS THEN
    RETURN input_string;
  END decrypt;
END pkg_crypto;
/
```

8. 테스트
```sql
create table t1 (c1 number, c2 varchar2(200));
insert into t1 values (1, pkg_crypto.encrypt('aaaaaaaaasdfaaaa'));
select c1, pkg_crypto.decrypt(c2) c2 from t1;
```