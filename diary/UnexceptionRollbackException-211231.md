## 문제
Entity에 낙관적 잠금을 설정하였고 try-catch 구문으로 예외를 발생하지 않도록 하였다. Repository 단위 테스트 시에도 OptimisticException이 발생하고 정상적으로 종료되었다. 하지만 서비스 테스트에서는 다음과 같은 에러가 발생하였다. 

>Transaction rolled back because it has been marked as rollback-only
org.springframework.transaction.UnexpectedRollbackException: Transaction rolled back because it has been marked as rollback-only

롤백을 해야하는건 맞지만 굳이 왜 이런 메시지를 뿌려주는가?

**Entity**
```java
class TestEntity {
    @Id
    private Long id;

    private String name;

    @Version
    private Long version;
}
```

**Repository**
```java
interface TestEntityRepository implements JpaRepository<TestEntity, Long> {

}
```

**Service**
예제 구성을 위해 save 메소드에서 Parameter를 entity로 하였다. 서비스 단에서 엔티티를 직접 받으면 안 된다.
```java
@Service
class TestService {
    
    private final TestEntityRepository repository;

    TestService(TestEntityRepository repository) {
        this.repository = repository;
    }

    @Transaction
    public void save(TestEntity entity) {
        try {
            repository.save(entity);
        } catch (OptimisticLockException e) {
            System.out.println("낙관적 락이 발생하였습니다.");
        }
    }
}
```

## 원인 분석
- spring-tx:5.3.14 기준
**TransactionAspectSupport**
```java


/**
* Handle a throwable, completing the transaction.
* We may commit or roll back, depending on the configuration.
* @param txInfo information about the current transaction
* @param ex throwable encountered
*/
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
if (txInfo != null && txInfo.getTransactionStatus() != null) {
    if (logger.isTraceEnabled()) {
        logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
                "] after exception: " + ex);
    }
    if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
        try {
            txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
        }
        catch (TransactionSystemException ex2) {
            logger.error("Application exception overridden by rollback exception", ex);
            ex2.initApplicationException(ex);
            throw ex2;
        }
        catch (RuntimeException | Error ex2) {
            logger.error("Application exception overridden by rollback exception", ex);
            throw ex2;
        }
    }
    else {
        // We don't roll back on this exception.
        // Will still roll back if TransactionStatus.isRollbackOnly() is true.
        try {
            txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
        }
        catch (TransactionSystemException ex2) {
            logger.error("Application exception overridden by commit exception", ex);
            ex2.initApplicationException(ex);
            throw ex2;
        }
        catch (RuntimeException | Error ex2) {
            logger.error("Application exception overridden by commit exception", ex);
            throw ex2;
        }
    }
}
}
```

**DefaultTransactionAttribute**
```java
/**
* The default behavior is as with EJB: rollback on unchecked exception
* ({@link RuntimeException}), assuming an unexpected outcome outside of any
* business rules. Additionally, we also attempt to rollback on {@link Error} which
* is clearly an unexpected outcome as well. By contrast, a checked exception is
* considered a business exception and therefore a regular expected outcome of the
* transactional business method, i.e. a kind of alternative return value which
* still allows for regular completion of resource operations.
* <p>This is largely consistent with TransactionTemplate's default behavior,
* except that TransactionTemplate also rolls back on undeclared checked exceptions
* (a corner case). For declarative transactions, we expect checked exceptions to be
* intentionally declared as business exceptions, leading to a commit by default.
* @see org.springframework.transaction.support.TransactionTemplate#execute
*/
@Override
public boolean rollbackOn(Throwable ex) {
    return (ex instanceof RuntimeException || ex instanceof Error);
}
```

**AbstractPlatformTransactionManage**
```java
/**
* This implementation of rollback handles participating in existing
* transactions. Delegates to {@code doRollback} and
* {@code doSetRollbackOnly}.
* @see #doRollback
* @see #doSetRollbackOnly
*/
@Override
public final void rollback(TransactionStatus status) throws TransactionException {
    if (status.isCompleted()) {
        throw new IllegalTransactionStateException(
                "Transaction is already completed - do not call commit or rollback more than once per transaction");
    }

    DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
    processRollback(defStatus, false);
}

/**
* Process an actual rollback.
* The completed flag has already been checked.
* @param status object representing the transaction
* @throws TransactionException in case of rollback failure
*/
private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
    try {
        boolean unexpectedRollback = unexpected;

        try {
            triggerBeforeCompletion(status);

            if (status.hasSavepoint()) {
                if (status.isDebug()) {
                    logger.debug("Rolling back transaction to savepoint");
                }
                status.rollbackToHeldSavepoint();
            }
            else if (status.isNewTransaction()) {
                if (status.isDebug()) {
                    logger.debug("Initiating transaction rollback");
                }
                doRollback(status);
            }
            else {
                // Participating in larger transaction
                if (status.hasTransaction()) {
                    if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
                        if (status.isDebug()) {
                            logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
                        }
                        doSetRollbackOnly(status);
                    }
                    else {
                        if (status.isDebug()) {
                            logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
                        }
                    }
                }
                else {
                    logger.debug("Should roll back transaction but cannot - no transaction available");
                }
                // Unexpected rollback only matters here if we're asked to fail early
                if (!isFailEarlyOnGlobalRollbackOnly()) {
                    unexpectedRollback = false;
                }
            }
        }
        catch (RuntimeException | Error ex) {
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
            throw ex;
        }

        triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);

        // Raise UnexpectedRollbackException if we had a global rollback-only marker
        if (unexpectedRollback) {
            throw new UnexpectedRollbackException(
                    "Transaction rolled back because it has been marked as rollback-only");
        }
    }
    finally {
        cleanupAfterCompletion(status);
    }
}
```


## 참고 사이트
[우아한형제 기술블로그 - 응? 이게 왜 롤백되는거지?](https://techblog.woowahan.com/2606/)