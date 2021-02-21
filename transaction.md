트랜잭션

트랜잭션 의미, isolation, propagation, readonly, 사용방법

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
