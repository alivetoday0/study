# 데이터베이스의 Transaction
데이터베이스의 기준은 MySQL이다. 

## 목적 
포인트를 순차적으로 증감하기 위해 트랜잭션의 isolation 레벨을 serializable을 사용하였지만 내가 원한 데이터가 나오지 않는다. 대신 Primitive Lock을 사용하여 원하는 결과를 얻었다. 내가 알고 있는 것과 달라 다시 한 번 공부하고 나중에 잊지 않기 위해 이 문서를 작성한다. 

- 소스 : https://github.com/alivetoday0/transaction-test

## 분석 
### 1) MySql 설정 문제일까? 
MySQL에서 Serializable 레벨을 사용하려면 auto commit이 false이어야 한다. 

Spring은 
- hikari CP에서 auto commit을 ture가 기본값이다. 
- Hibernate는 위 기본 값을 기반으로 
  - ture이면 set auto_commit을 0으로 한 후 쿼리 실행 및 commit 후 auto_commit을 1로 변경한다. 
> `Tip` hikari CP의 auto commit을 false로 하고 Hibernate의 connection.provider_disables_autocommit을 true로 하면 위와 같은 auto_commit 문을 실행하지 않는다. 

### 1) 결론

## 기반 지식
### ACID 
자세한 내용은 [위키](https://ko.wikipedia.org/wiki/ACID)를 참고하라. 
- Atomicity(원자성)
- Consistency(일관성)
- Isolation(격리성)
- Durability(지속성)

#### Isolation 
**REPETBLE READ**
Consistent read는 같은 트랜잭션에서 처음 읽을 때 만들어진 스냅샷을 읽는다.   
Locking Reads(SELECT with FOR UPDATE or LOCK IN SHARE MODE) 일 경우
- unique index가 조건일 경우 index record 락이 걸린다. 
- index range 스캔일 경우 insert를 막기 위해 gap lock 혹은 next-key lock이 걸린다.

**READ COMMITED**   
같은 트랙잰션 내에서도 새로운 스냅샷을 설정하고 읽는다.   
Locking Reads 일 경우 index record 락이 걸린다.
- gap locking은 외래키 제약 검사와 중복키 검사에만 쓰인다.
- gap locking이 없어 phantom read가 발생한다. 

추가적인 영향
- UPDATE 또는 DELETE 구문에서 수정 혹은 삭제되는 행을 잠금니다. 일치하지 않는 행은 where 조건문 평가 후 잠금을 해제합니다. 적은 확률로 DEADLOCK이 발생할 수 있습니다. 
- UPDATE 경우 최신 커밋된 버전을 반환하여 where 조건문과 일치하는지 검사한다. 일치하면 행을 다시 읽고 잠그거나 잠길 때까지 기다린다. 

**READ UNCOMMITED**
잠금 해제 방식으로 수행되어 이전 버전의 행을 사용할 수 있다. 이로 인하여 drity read가 발생할 수 있다. 

**SELIALIZABLE**   
This level is like REPEATABLE READ, but InnoDB implicitly converts all plain SELECT statements to SELECT ... FOR SHARE if autocommit is disabled. If autocommit is enabled, the SELECT is its own transaction. It therefore is known to be read only and can be serialized if performed as a consistent (nonlocking) read and need not block for other transactions. `(To force a plain SELECT to block if other transactions have modified the selected rows, disable autocommit.)`

> 구글 번역   
이 수준은 REPEATABLE READ와 비슷하지만 자동 커밋이 비활성화된 경우 InnoDB는 모든 일반 SELECT 문을 SELECT ... FOR SHARE로 암시적으로 변환합니다. 자동 커밋이 활성화된 경우 SELECT는 자체 트랜잭션입니다. 따라서 읽기 전용으로 알려져 있으며 일관된(비잠금) 읽기로 수행되는 경우 직렬화될 수 있으며 다른 트랜잭션에 대해 차단할 필요가 없습니다. `(다른 트랜잭션이 선택한 행을 수정한 경우 일반 SELECT를 강제로 차단하려면 자동 커밋을 비활성화합니다.)`


### 참고문서
- [Transaction Propagation and Isolation in Spring @Transactional](https://www.baeldung.com/spring-transactional-propagation-isolation)
- [Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
- [autocommit 튜닝](https://pkgonan.github.io/2019/01/hibrnate-autocommit-tuning)