정리중..

트랜잭션

트랜잭션 의미, isolation, propagation, readonly, transaction lock, 사용방법, snapshot

Transaction 
 - 데이터베이스의 독립적으로 실행되는 작업 단위
 - 단일 작업 또는 여러 작업으로 구성
 
AICD
 1. 원자성(Atomicity) : 트랜잭션과 관련된 작업들이 부분적으로 실행되다가 중단되지 않는 것을 보장하는 것(All and Nothing)
 2. 일관성(Consistency) :  트랜잭션이 실행을 성공적으로 완료하면 언제나 일관성 있는 데이터베이스 상태로 유지하는 것을 의미
    - 무결성 제약 조건
 3. 독립성(Isolation) : 트랜잭션을 수행 시 다른 트랜잭션의 연산 작업이 끼어들지 못하도록 보장하는 것을 의미
 4. 지속성(Durability) : 성공적으로 수행된 트랜잭션은 영원히 반영되어야 함을 의미

ACID는 DB의 모든 연산이 한번에 실행되는 것을 권장하는데 필요한 기법이 두가지 있다.
 1. 로깅방식
    - 원자성은 DB에 데이터를 업데이트 하기 전에 로그에 모든 변경사항을 기록하는 것으로 보장
    - 충돌 현상이 발생하더라도 DB 무결성을 보장
     
 2. 새도우 패이징
    - 변경이 DB의 복사본에 저장
    - 새로운 복사본은 트랜잭션이 commit 되면 활성화
    
위의 두 방식은 lock이 필요하다. 이는 많은 수의 락을 관라 하게 되면 동시작업 수행이 어렵고 성능저하를 초래함.
 - A유저가 특정 테이블을 읽고 있다면 B유저는 A의 트랜잭션이 끝날 때까지 기다려야함

MVCC(multiversion concurrency control)
 - 락의 대안으로 수정되는 모든 데이터를 별도 복사본으로 관리
1. MVCC 데이터베이스가 데이터의 업데이트가 필요할 때, 기존 데이터 항목을 새로운 데이터가 덮어쓰는 대신 데이터 항목의 새로운 버전을 만듬
 - 즉, 여러버전이 저장됨
2. 각 트랜잭션이 주시하는 버전은 구현된 격리 레벨에 따름
3. MVCC 상태에서 읽기 트랜잭션은 일반적으로 타임스탬프나 트랜잭션 ID를 사용하여 읽을 DB의 상태를 결정하고 데이터의 버전들을 읽음
4. 읽기, 쓰기 트랜잭션은 락(lock)의 필요 없이 다른 트랜잭션과 격리됨

isolatoion
 1. READ UNCOMMITTED
 2. READ COMMITTED
 3. REPEATABLE READ
 4. SERIALIZABLE

몽고디비
 - 일관성을 보장하기 위해 잠금 및 기타 동시성 제어 수단을 사용하여 여러 클라이언트가 동일한 데이터를 동시에 수정하지 못하도록 합니다.


1. propagation


몽고디비는 document에서 하나의 로우는 원자성을 보장함.
 - 따라서 쓰기 작업의 지연시간이 오래 걸리면 읽기의 성능이 저하됨(쓰고있는 특정 데이터)

여러 document의 원자성을 보장하려면 transaction을 활용 해야함
 - 몽고디비에서는 4.0 Version 이상에서 사용할 수 있음
 - transaction을 지원하기 때문에 읽기성능이 낮아짐

transaction이 커밋되면 transaction에서 변경된 모든 데이터가 저장되고 트랜잭션 외부에 표시
 - transaction이 커밋되기 전까지 외부 transaction에 표시되지 않음

대부분의 경우 multi-document transaction은 single document write 보다 높은 비용이 발생한다.
 - schema design을 적절하게 모델링하면 transaction이 필요성을 최소화 할 수 있다.


1) Transaction 단위는 method 이다. (Spring AOP 의 특징)
2) 외부에서 호출하는 method 에 대해서만 Transaction 이 설정 된다.
3) Transaction 대상 객체 참조는 종속성 주입을 통해서 얻어야 한다.

@Transactional
public void initialPayment(PaymentRequest request) {
savePaymentRequest(request); // DB
callThePaymentProviderApi(request); // API
updatePaymentState(request); // DB
saveHistoryForAuditing(request); // DB
}

1. 트랜잭션이 시작되면 EntityManager를 만들고 connection fool에서 connection을 사용함
2. 외부 api콜이 완료 될 때 까지 connection을 유지함
3. connection을 사용하여 이후 로직 수행

이 때 문제는 외부 api콜 응답이 느려진다면 connection을 점유하고 있기 때문에 장애가 발생할 수 있음
database I/O와 다른 타입의 I/O를 조합해서 사용은 조심해야함
- 해결책은 분리하는것!
- 하지만 분리 할 수 없는 이유가 있다면? 어떻게 해야하나
- 수동으로 transaction을 관리 해야함

Using TransactionTemplate

TransactionTemplate : 트랜잭션을 관리 하기 위한 콜백 기반의 api를 제공함

1. PlatformTransactionManager

// test annotations
class ManualTransactionIntegrationTest {

    @Autowired
    private PlatformTransactionManager transactionManager;

    private TransactionTemplate transactionTemplate;

    @BeforeEach
    void setUp() {
        transactionTemplate = new TransactionTemplate(transactionManager);
    }

    // omitted
}
