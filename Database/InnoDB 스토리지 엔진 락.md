# Record, Gap, Next Key, Auto Increment

## InnoDB 스토리지 엔진의 잠금
스토리지 엔진 레벨의 잠금은 테이블의 데이터를 다루기 위한 락이다. 
- Record Lock
- Gap Lock
- Next Key Lock
- Auto Increment Lock

## Record Lock
레코도 자체만을 잠그는 것을 레코드 락이라고 한다. 잠금 정보가 상당히 작은 공간으로 관리되기 때문에 레코드 락이 페이지 락, 테이블 락으로 레벨업되는 경우는 없다. 다른 DBMS의 레코드 락과 동일한 역할을 하지만 한 가지 중요한 차이는 ***레코드 자체가 아니라 인덱스의 레코드***를 잠근다는 점이다.
인덱스가 하나도 없는 테이블이라도 내부적으로 자동 생성된 클러스터링 인덱스(PK)를 이용해 잠금을 설정한다.

InnoDB에서는 대부분 보조 인덱스를 이용한 변경 작업은 넥스트 키 락, 갭 락을 사용하지만 PK, 유니크 인덱스에 의한 변경 작업에서는 갭에 대해서는 잠그지 않고 레코드 자체에만 락을 건다.

인덱스 레코드에 락을 거는 것과 레코드를 잠그는 것은 큰 차이가 있다. 즉 변경해야 할 레코드를 찾기 위해 검색한 인덱스의 레코드를 모두 걸어야 한다.

이해를 위해 예를 들어보자.

`employees` 테이블에는 `first_name(이름)`, `last_name(성)` 컬럼이 있고, `index(first_name)`이 걸려 있다.
`employees` 테이블에서 `first_name='Georgi'`인 사원은 200명이 있고, `last_name='Klassen'` 이고 `first_name='Georgi"`인 사원은 딱 1명만 있다

```sql
# 200건
SELECT COUNT(*) FROM employees WHERE first_name='Gerogi';
# 1건
SELECT COUNT(*) FROM employees WHERE first_name='Gerogi' AND last_name='Klassen';
```

`employees` 테이블에서 `fisrt_name=‘Georgi’` 이고 `last_name=‘Klassen’`인 사원의 입사 일자를 오늘로 변경하는 쿼리를 실행해보자

```sql
UPDATE employees SET hire_date=NOW() WHERE first_name='Gerogi' AND last_name='Klassen';
```

이 쿼리에 의해 영향을 받는 쿼리는 1건이다. 하지만 이 1건을 업데이트 하기 위해 200건의 인덱스 레코드에 락이 걸린다. 왜냐하면 `last_name` 컬럼에는 인덱스가 없기 때문 `fisrt_name=‘Gerogi’` 인 레코드 200건의 레코드가 모두 잠긴다. 

UPDATE 문장을 위해 적절히 인덱스가 준비돼 있지 않는다면 동시성이 상당히 떨어지게 된다.
레코드 락은 SELECT FOR UPDATE, UPDATE, DELETE에서 자동으로 실행되는 락이다.

## Gap Lock
갭 락은 레코드 자체가 아니라 레코드와 바로 인접한 레코드 사이의 간격만을 잠그는 것을 의미한다. 갭 락의 역할은 레코드와 레코드 사이의 간격에 새로운 레코드가 생성, 수정, 삭제되는 것을 제어한다.

Account 테이블에 index(money)가 존재하고, 아래와 2개의 인덱스 레코드가 존재한다.
**인덱스 리프 노드**
| money | PK  |
|-------|-----|
|   x   | x   |
| 10000 | 5   |
| 14000 | 6   |
|   x   | x   |
|   x   | x   |
|   x   | x   |

이 때 아래 쿼리를 실행한다.
```sql
SELECT * FROM account a WHERE a.money >= 10000 AND a.money <= 20000 
```
이 때 조건에 해당하는 곳에는 레코드 락이 걸리고, 아직 실존하지 않고 나중에 추가될 수 있는 공간에는 갭 락이 걸린다.

**Gap Lock 시나리오**
1. `WHERE member_id = ? FOR UPDATE` 
   - 특정 행에 Record Lock, 새 데이터 삽입 가능
2. `WHERE member_id > ? FOR UPDATE` 
   - 조건에 해당하고 존재하는 행에는 Record Lock, 존재하지 않는 레코드에는 Gap Lock 걸린다.
3. `WHERE member_name = ? FOR UPDATE` 
   - 단일 조건으로 해도 보조 인덱스 같은 경우에는 Gap Lock 발생 가능성 있다.
```sql
SELECT * FROM member WHERE member_name='김철수' FOR UPDATE;
```
이 경우 member_name=‘김철수’에 대해서는 Record Lock이 걸리고 보조 인덱스 ‘김철수’ 이후의 Gap을 보호하기 위해 Gap Lock이 걸릴 수 있다.

## Next Key Lock
레코드 락과 갭 락을 합쳐 놓은 형태의 잠금을 넥스트 키 락이라고 한다. InnoDB의 갭 락이나 넥스트 키 락은 바이너리 로그에 기록되는 쿼리가 레플리카 서버에서 실행될 때 소스 서버에서 만들어 낸 결과와 동일한 결과를 만들어내도록 보장하는 것이 주목적이다. 그런데 의외로 넥스트 키 락과 갭 락으로 인해 데드락이 발생하거나 다른 트랜잭션을 기다리게 만드는 일이 자주 발생한다. 가능하다면 바이너리 로그 포맷을 ROW 형태로 바꿔서 넥스트 키 락이나 갭 락을 줄이는 것이 좋다.

> MySQL 8.0에서는 ROW 포맷의 바이너리 로그가 기본 설정으로 변경되었다.


## Auto Increment Lock
MySQL에서는 자동 증가하는 숫자 값을 추출하기 위해 `AUTO_INCREMENT`라는 컬럼 속성을 제공한다.
`AUTO_INCREMENT` 컬럼이 사용된 테이블에 동시에 여러 레코드가 INSERT되는 경우, 저장되는 각 레코드는 중복되지 않고 저장된 순서대로 증가하는 일련번호 값을 가져야한다. 이를 위해 `AUTO_INCREMENT` 락이라고 하는 테이블 수준의 잠금을 사용한다. 
해당 락은 `INSERT`, `REPLACE`와 같이 새로운 레코드를 저장하는 쿼리에서만 필요하다. `AUTO_INCREMENT` 값을 가져오는 순간만 락이 걸렸다고 바로 해제되기 때문에 대부분의 경우 문제가 되지 않는다.

명시적으로 획득하고 해제하는 방법은 없다. 

**참고 래퍼런스**
> - [\[MySQL\] 스토리지 엔진 수준의 락의 종류\(레코드 락, 갭 락, 넥스트 키 락, 자동 증가 락\)](https://mangkyu.tistory.com/298)
> - Real MySQL
