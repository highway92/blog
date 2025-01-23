[참조 블로그](https://vladmihalcea.com/maximum-database-connections/)

## 들어가기앞서
DBMS 성능 최적화와 관련된 세팅을 하다 보면 항상 마주하게 되는 것이 max connection이다. 아래 글에서는 Connection의 개념과 Max Connection을 어떻게 설정해야 하는지에 대한 내용을 다룬다.


## DBMS의 Connection 개념
DB Connection은 어플리케이션(프로세스)이 데이터베이스(DBMS)와 통신하기 위한 논리적인 연결이다.
Connection에는 Host 정보, Port 번호, 사용자 인증 정보, DB 이름과 같은 요소가 있으며, 이를 바탕으로 DBMS는 인증을 진행하고 TCP 연결을 한다.

### Connection 의 동작 과정
웹 어플리케이션을 예시로 Connection의 동작을 살펴보면,

1. 애플리케이션은 DB 드라이버를 통해 인증 정보와 함께 연결을 요청한다.
2. DBMS는 인증 정보를 검토하고 TCP 소켓을 열어 연결을 승인한다.
3. 애플리케이션은 SQL 쿼리를 전송하고, DBMS는 처리 결과를 반환한다.
4. 작업이 완료되면 연결을 종료하고 TCP 소켓을 닫는다.

### Connection의 특징
1. 리소스 소모: 네트워크 소켓(TCP 소켓) 및 DBMS의 리소스를 소비한다.
2. Connection 수가 증가하면 DBMS 서버의 부하가 증가할 수 있다.
3. 동시성: DBMS는 여러 클라이언트의 요청을 처리하기 위해 다수의 Connection을 지원한다.

따라서 대부분의 애플리케이션은 Connection Pool을 사용해 Connection을 효율적으로 관리한다. 대표적인 라이브러리로는 HikariCP가 있다.


## Max Connection
Oracle은 1000, SQL Server는 32,767, PostgreSQL은 100, MySQL은 151(버전에 따라 다를 수 있음)의 기본 Max Connection이 설정되어 있다. 그렇다면 우리는 이 설정을 그대로 사용해도 되느냐? 그렇게 해도 되지만, 어떻게 값을 설정해야 하는지 알아야 할 필요가 있다.

### Max Connection에 대한 오해
이 부분에서 가장 충격을 받았던 사실은, 필자가 Max Connection이 높을수록 좋다고 생각했다는 것이다. Connection을 생성하고 유지하는 데는 컴퓨팅 리소스가 필요한데, 더 많은 CPU 코어를 가진 서버일수록 높은 Max Connection을 설정할 수 있고, 이게 DBMS 성능과 이어진다고 생각했기 때문이다.

하지만 여기에는 한 가지 간과된 사실이 있다. 바로 Connection을 추가하고 해제하는 비용이 생각보다 크다는 점이다. 따라서 Max Connection이 적게 설정되어 있어도 적절히 설정되어 있다면, 더 높은 성능을 발휘할 수 있다.
(자세한 설명은 참조 블로그의 What limits the maximum number of connections? 부분을 살펴보자.)

### Max Connection의 설정
Max Connection 설정은 성능 테스트를 통해서 결정하는 방법밖에 없다. 배포 환경이 매우 다르기 때문이다. (컴퓨팅 파워, OS, 다른 프로세스의 호스팅 여부 등) 그렇기 때문에 값을 늘리거나 줄이며 최적의 성능을 찾아내는 노력이 필요하다.

기본적으로 Max Connection의 기본 설정이 너무 크게 되어 있는 경우가 많다. 값이 크게 설정되어있다면 어떻게 좋지 않을까? 위에서 언급했든 connection은 TCP 통신이다. TCP통신을 유지하기 위해서는 주기적으로 ack(신호)를 보내야 하고 이는 네트워크 리소스를 소비하게 된다. 또한 DBMS는 각 연결마다 상태정보를 메모리에 저장하기 때문에 연결이 되어있는 동안 일정 메모리를 점유하게 된다.

## 결론

DBMS의 Max Connection 설정은 단순히 서버의 CPU나 메모리 리소스에 비례해 설정할 수 있는 값이 아니다. 오히려 동시 연결의 생성과 해제에 드는 비용을 고려해야 하며, 이로 인해 Max Connection을 너무 높게 설정하는 것이 반드시 성능 향상으로 이어지지 않을 수 있다. 최적의 Max Connection 값은 성능 테스트를 통해 구체적인 서버 환경에 맞게 조정되어야 한다.

따라서 테스트와 모니터링을 반복하면서, 애플리케이션의 실제 사용 패턴에 맞는 적절한 값을 찾는 것이 중요하다. 지나치게 높은 설정은 불필요한 리소스 소비를 초래할 수 있고, 너무 낮은 값은 연결 대기 시간을 늘려 시스템의 성능을 저하시킬 수 있기 때문이다.
