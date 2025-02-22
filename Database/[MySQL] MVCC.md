# [MySQL] MVCC
## MVCC(Multi Version Concurrency Control)
**동시 접근을 허용하는 데이터베이스에서 동시성을 제어하기 위해 사용하는 방법 중 하나이다.**

동시성을 제어하기 위한 방법 중 하나인 Lock-based 방법은 동일한 데이터에 대해 읽기와 쓰기가 동시에 수행되면 한 작업이 실행되는 동안 다른 작업이 블록되어 전체 처리량이 저하되고 성능이 떨어지는 문제가 발생할 수 있다. 이를 해결하기 위해 MVCC 기술이 등장했다.

MVCC의 가장 큰 목적은 잠금을 사용하지 않는 일관된 읽기를 제공하는 데 있다. InnoDB는 언두 로그(Undo log)를 이용해 이 기능을 구현한다.

## 동작과정
InnoDB 스토리지 엔진에서 MVCC가 어떻게 동작되는지 예시를 보자.
아래와 같이 member 테이블에 한 건의 레코드를 INSERT를 한다.
```sql
INSERT INTO member (m_id, m_name, m_area) VALUES (12,'홍길동','서울');
```

INSERT 문이 실행되면 데이터베이스 상태는 아래 그림과 같이 바뀐다.
메모리와 디스크에 해당 데이터가 동일하게 저장된다.

![image 2](https://github.com/user-attachments/assets/3c600ae8-0793-442c-888b-28760c735718)

```sql
UPDATE member SET area = '경기' WHERE id = 12;
```
아래와 같이 UPDATE 문을 실행시킨 결과는 아래 그림과 같다.

![image](https://github.com/user-attachments/assets/b75180f7-77f4-4368-b451-1523a0d3b59c)

- UPDATE 문을 실행하면 COMMIT 실행 여부와 관계없이 InnoDB 버퍼 풀은 새로운 값인 ‘경기’로 업데이트 된다.
- 언두 로그에는 변경 전의 값들만 복사된다.

아직 COMMIT이나 ROLLBACK이 되지 않은 상태에서 다른 사용자가 다음과 같은 쿼리로 데이터를 조회한다면 어떻게 될까?
```sql
SELECT * FROM member WHERE m_id = 12;
```

그 결과는 트랜잭션의 격리 수준에 따라 다르다. 
- READ UNCOMMITTED
  - InnoDB 버퍼 풀이 현재 가지고 있는 변경된 데이터를 읽어서 반환한다.
- READ COMMITTED, REPEATABLE READ, SERIALIZABLE
  - 아직 커밋되지 않았기 때문에 변경되기 이전의 언두 로그 영역의 데이터를 반환하게 된다.

## MVCC에서 생길 수 있는 문제
### Lost Update

Lost Update에 상황을 살펴보자

DB에 X = 50, Y = 10이 저장되어있고, TX1에서는 X -> Y 로 40을 이체, TX2에서는 X에 30을 입금한다. 정상적인 결과는 X = 40, Y = 50이 되어야 한다.

아래 과정은 문제가 되는 상황이다.

<img width="2268" alt="스크린샷 2025-02-22 오후 8 35 09" src="https://github.com/user-attachments/assets/e5b58eea-446a-4270-bdf3-750835163898" />

1. TX2에서 먼저 x의 값 50을 읽고,  write lock을 획득한 뒤 x의 값을 30을 더한다.
   - 바뀐 x 값은 InnoDB 버퍼 풀에 반영되고, 언두 로그에 변경 전 데이터가 저장된다.
2. TX1에서 x의 값을 읽어온다. 이 때 TX2가 커밋되기 전이기 때문에 언두 로그 영역에 변경 전 데이터 x = 50을 읽어온다.
3. TX1에서 x -> y로 40을 이체한다. x = 10, y = 50으로 변경하고 커밋한다.
4. 최종 결과 x = 10, y = 50 잘못된 결과가 DB에 저장된다.

이를 해결하는 방법으로는 Locking Read를 통해 해결한다.
TX2에서 `read(x) = 50`을 할 때 Lock을 걸어 TX1에서 `read(x) = 50`을 하지 못하도록 하면 된다.
