##Motivation

Dependency Injection can be performed in a number of ways like setter-injection, constructor-injection and field-injection. 
Symphony uses field-injection heavily which makes autowiring beans a cakewalk, but can pose some issues while attempting
to thoroughly unit-test the code.

The intent of this exercise is to pick a particular class and subject it a variety of different "flavors" of testing. This should hopefully give the reader some ideas on how to better set up their tests for current and future development.

## Table of Contents
* **[Picking a class](#picking-a-class)**
* **[Writing a simple test with Mockito but not Spring](#writing-a-simple-test-with-mockito-but-not-spring)**
* **[Writing the same test with Spring but not Mockito](#writing-the-same-test-with-spring-but-not-mockito)**
* **[Writing the same test with Spring while overriding imported beans](#writing-the-same-test-with-spring-while-overriding-imported-beans)**
* **[Checkpoint - What is a unit?](#checkpoint---what-is-a-unit)**
* **[Rewriting the test correctly](#rewriting-the-test-correctly)**

##Picking a class

I'm picking the [PaymentProviderHelper](https://github.paypal.com/sahjain/paymentapiplatformserv/blob/develop/app-core/src/main/java/com/paypal/platform/payments/core/provider/impl/PaymentProviderHelper.java) class for the purposes of this exercise.
If you scan through all 29 dependencies that have been injected in it, you'll notice that they can be roughly classified as:
* Mappers (ex: ShippingAddressMapper, ApiErrorMap)
* Services that are simple (ex: SessionService)
* Services with common interfaces (ex: salePaymentService, orderPaymentService, authPaymentService and legacyPayService are all ExecutePaymentServices underneath. They've been annotated with qualifiers to resolve bean ambiguity)
* Services that wrap DB interactions (ex: IdempotenceService)
* Services that wrap configurations (ex: PaymentsPathService, AppConfiguration)
* Services that wrap security (ex: PayIdSecurity)
* Helpers (ex: ProviderHelper)
* Threadlocal contexts (ex: PAPSRequestContext)

Barring `bridges`, this class seems to hold all flavors of dependencies we'd typically see in any Symphony class. This is promising,
because if we can write good tests for this class, then we should be able to write good tests for any other class.

##Writing a simple test with Mockito but not Spring

Let's say we're interested in testing the `incrementAndValidatePaymentAttempts` [method](https://github.paypal.com/sahjain/paymentapiplatformserv/blob/develop/app-core/src/main/java/com/paypal/platform/payments/core/provider/impl/PaymentProviderHelper.java#L787). A number of test cases should spring to mind, but to keep things simple, let's say that we are just interested in testing whether we can make our "first" payment attempt. Here's what a test without using Spring may look like:

```java
import static org.mockito.Mockito.*;

public class PaymentProviderHelperTest {

    @Mock
    private AppVarDataVOHelper appVarDataVOHelper;

    @InjectMocks
    private PaymentProviderHelper payProviderHelper;

    @Before
    public void prepare() {
        MockitoAnnotations.initMocks(this);
    }

    @Test
    public void testFirstPaymentAttempt() {
        // setup input context and mock behavior
        UCPFlowControlVO ucpFlowControlVO = new UCPFlowControlVO();
        ucpFlowControlVO.setPaymentAttempts(null);
        SessionVarDataVO sessionVarDataVO = mock(SessionVarDataVO.class);
        when(appVarDataVOHelper.getUCPFlowControlVO(sessionVarDataVO)).thenReturn(ucpFlowControlVO);
        
        // call the method
        payProviderHelper.incrementAndValidatePaymentAttempts(sessionVarDataVO);
        
        // assert
        Assert.assertEquals(Long.valueOf(1l), ucpFlowControlVO.getPaymentAttempts());
    }
}
```
The important takeaway here is how `@Mock`, `@InjectMocks` and `MockitoAnnotations.initMocks(this)` come into play. Essentially, as a part of the `MockitoAnnotations.initMocks(this)`, we're telling Mockito to find all fields in this test class marked with the `@Mock` annotation and automatically mock them, but most importantly inject these mocks into the class with the annotation `@InjectMocks`. As an extension of this, the `@InjectMocks` creates a real instance of the `paymentProviderHelper` which has now been autowired with the mocked instance of `appVarDataVOHelper`.

The advantage here is that you can tweak the behavior of the `appVarDataVOHelper` for every single test and make it return different values of the `ucpFlowControlVO`. This is great because it gives you a lot of control over the initial test setup context without incurring any overhead of managing your spring beans punctiliously. I would encourage you read this [article published by Martin Fowler](http://martinfowler.com/articles/mocksArentStubs.html) that makes an important and subtle disctinction between state-based and behavior-based verification.

Let's drive this point home by tweaking the mock's behavior per-test as follows:

```java
import static org.mockito.Mockito.*;

public class PaymentProviderHelperTest {

    @Mock
    private AppVarDataVOHelper appVarDataVOHelper;

    @InjectMocks
    private PaymentProviderHelper payProviderHelper;

    @Before
    public void prepare() {
        MockitoAnnotations.initMocks(this);
    }

    @Test
    public void testFirstPaymentAttempt() {
        // setup input context and mock behavior
        UCPFlowControlVO ucpFlowControlVO = new UCPFlowControlVO();
        ucpFlowControlVO.setPaymentAttempts(null);
        SessionVarDataVO sessionVarDataVO = mock(SessionVarDataVO.class);
        when(appVarDataVOHelper.getUCPFlowControlVO(sessionVarDataVO)).thenReturn(ucpFlowControlVO);
        
        // call the method
        payProviderHelper.incrementAndValidatePaymentAttempts(sessionVarDataVO);
        
        // assert
        Assert.assertEquals(Long.valueOf(1l), ucpFlowControlVO.getPaymentAttempts());
    }
    
    @Test
    public void testPaymentAttemptNearThresholdLimit() {
        // setup input context and mock behavior
        UCPFlowControlVO ucpFlowControlVO = new UCPFlowControlVO();
        ucpFlowControlVO.setPaymentAttempts(PaymentProviderHelper.MAX_PAYMENT_ATTEMPTS - 1L);
        SessionVarDataVO sessionVarDataVO = mock(SessionVarDataVO.class);
        // modify the mock behavior by returning a different ucpFlowControlVO
        // we always have test-local control to influence the SUT's direct dependency's behavior
        when(appVarDataVOHelper.getUCPFlowControlVO(sessionVarDataVO)).thenReturn(ucpFlowControlVO);
        
        // call the method
        payProviderHelper.incrementAndValidatePaymentAttempts(sessionVarDataVO);
        
        // assert
        assertEquals(new Long(PaymentProviderHelper.MAX_PAYMENT_ATTEMPTS), ucpFlowControlVO.getPaymentAttempts());
    }
}
```

##Writing the same test with Spring but not Mockito

Let's say we want to do things in a more traditional Spring way. You'll notice that the "setup" code has changed where effort is being applied to put together a proper session as opposed to constructing appropriate mocks/replay behavior. Here's what a simple unit test would look like:

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration
public class PaymentProviderHelperTest {

    @Configuration
    @Import(ProviderTestConfig.class) // import the whole world
    static class MyLocalConfig {

    }

    @Inject
    private PaymentProviderHelper payProviderHelper;

    private AppVarDataVOHelper appVarDataVOHelper;

    @Before
    public void prepare() {
        appVarDataVOHelper = new AppVarDataVOHelper();
    }

    @Test
    public void testFirstPaymentAttempt() {
        // setup input context
        SessionVarDataVO sessionVarDataVO = getSessionWithUCP();

        // call the method
        payProviderHelper.incrementAndValidatePaymentAttempts(sessionVarDataVO);

        // assert
        Assert.assertEquals(Long.valueOf(1l), appVarDataVOHelper.getUCPFlowControlVO(sessionVarDataVO)
                .getPaymentAttempts());
    }

    /**
     * Some method that loads a session configured in a particular way. You could
     * load it from a flat file if you wish, but you should really have a good reason
     * for doing something like that...
     * @return
     */
    private SessionVarDataVO getSessionWithUCP() {
        SessionVarDataVO sessionVarDataVO = new SessionVarDataVO();
        UCPFlowControlVO ucpFlowControlVO = new UCPFlowControlVO();
        ucpFlowControlVO.setPaymentAttempts(null); // explicitly showing no payment attempts
        appVarDataVOHelper.setUCPFlowControlVOToSessionVarDataVO(sessionVarDataVO, ucpFlowControlVO);
        return sessionVarDataVO;
    }
}
```
The important takeaway from this code is that we've annotated the test class with `@RunWith(SpringJUnit4ClassRunner.class)` which requires us to load an applicationContext. We do this by using the `@ContextConfiguration` which can take a list of classes that host the bean configurations. If no arguments are passed, then Spring looks for a local class with the `@Configuration` annotation to supply all the necessary beans. In this case, instead of outlining all the beans we need, we side-step the problem by importing the ProviderTestConfig via the `@Import(ProviderTestConfig.class)`. Why we're doing this like this instead of applying the imports at the top will make sense in the next example.

##Writing the same test with Spring while overriding imported beans

Sometimes you import a number of beans for your test, but you want to specify a particular behavior for one of the dependencies. Perhaps you'd like this bean mocked in a particular way, but you want to make sure there's no ambiguity in which bean gets loaded in the applicationContext for your test class. You could try something like this:

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration
public class PaymentProviderHelperTest {

    @Configuration
    @Import(ProviderTestConfig.class)
    static class MyLocalConfig {

        @Bean
        @Primary // We want to supply our own mocked version of appVarDataVOHelper
        public AppVarDataVOHelper appVarDataVOHelper() {
            return mockedAppVarDataVOHelper;
        }
    }

    private static AppVarDataVOHelper mockedAppVarDataVOHelper;

    static {
        mockedAppVarDataVOHelper = mock(AppVarDataVOHelper.class);
    }

    @Inject
    private PaymentProviderHelper payProviderHelper;

    @Test
    public void testFirstPaymentAttempt() {
        // setup input context and mock behavior
        SessionVarDataVO sessionVarDataVO = mock(SessionVarDataVO.class);
        UCPFlowControlVO ucpFlowControlVO = new UCPFlowControlVO();
        ucpFlowControlVO.setPaymentAttempts(null);
        when(mockedAppVarDataVOHelper.getUCPFlowControlVO(sessionVarDataVO)).thenReturn(ucpFlowControlVO);
        
        // call the method
        payProviderHelper.incrementAndValidatePaymentAttempts(sessionVarDataVO);
        
        // assert
        Assert.assertEquals(Long.valueOf(1l), ucpFlowControlVO.getPaymentAttempts());
    }
    
    @Test
    public void testPaymentAttemptNearThresholdLimit() {
        // setup input context and mock behavior
        SessionVarDataVO sessionVarDataVO = mock(SessionVarDataVO.class);
        UCPFlowControlVO ucpFlowControlVO = new UCPFlowControlVO();
        ucpFlowControlVO.setPaymentAttempts(PaymentProviderHelper.MAX_PAYMENT_ATTEMPTS - 1L);
        when(mockedAppVarDataVOHelper.getUCPFlowControlVO(sessionVarDataVO)).thenReturn(ucpFlowControlVO);

        // call the method
        payProviderHelper.incrementAndValidatePaymentAttempts(sessionVarDataVO);

        // assert
        Assert.assertEquals(Long.valueOf(PaymentProviderHelper.MAX_PAYMENT_ATTEMPTS),
                ucpFlowControlVO.getPaymentAttempts());
    }

}
```
Now the notion of the `MyLocalConfig` static inner class makes sense. We use it to not only "import the world" but also selectively override the beans that are interesting to us. For instance, for this test class we wanted to mock the behavior of `AppVarDataVOHelper` so we made sure that we referenced the appopriate instance (in this case, `mockedAppVarDataVOHelper`) inside the bean defined in the `MyLocalConfig` class. Moreover, we explicitly resolve any bean ambiguity by annotating the specific bean as `@Primary`.

This is a particularly flexible approach because you get to combine the best of both worlds. Specifically, after defining a simple mock skeleton, you can vary its behavior on a per-test case basis. Please note though: once you've decided to "mock" a SUT dependency, you're committing to "handling" the mock (by applying when/thenReturns) per test case. 

##Checkpoint - What is a unit?

So far, we looked at 3 approached to test the same simple `incrementAndValidatePaymentAttempts` method: 
* Using pure mocks (where the dependency's method behavior was mocked out based on a specific input data i.e. `appVarDataVOHelper`'s `getUCPFlowControlVO` method's behavior was mocked based on a mocked `SessionVarDataVO` being passed)
* Using pure spring (where the input data was pre-built without directly manipulating the methods' behavior)
* Using a spring + mock hybrid (which resembles using pure mocks with the exception that you're still working with an actual applicationContext that's been loaded)

It's important to pause here and consider what we truly consider the System under Test (SUT). Surely, in these examples, we wanted to test the PaymentProviderHelper's `incrementAndValidatePaymentAttempts` method. This method internally used one wired dependency of the `appVarDataVOHelper` (which is a mapper utility of sorts) which returns a usable UCPFlowControlVO. Now the question is, "should" we use a mock of the `appVarDataVOHelper`'s `getUCPFlowControlVO` method? A number of arguments can be made, so imagine a conversation between 2 people: 

* __[Person A]__: The SUT here is the PaymentProviderHelper so all of its internal dependencies *must* be mocked, ergo, we should mock the appVarDataVOHelper's method behavior. 
* __[Person B]__: It is only appropriate to mock an "external" dependency that does not simply entail the transformation of 1 datastructure into another. For all other things, use the underlying implementation! 
* __[Person A]__: But then haven't we expanded the scope of the SUT (also including the `AppVarDataVOHelper` in addition to the originally intended `PaymentProviderHelper`)? Seems like we're inadvertently testing much more than we had bargained for yes?
* __[Person B]__: Well, if we restrict the scope to test just the one class with dependencies always mocked, imagine what that would mean is the `AppVarDataVOHelper` implementation changes internally (where it starts sending back a `PurchaseDataVO` instead of a `UCPFlowControlVO`). Shouldn't the test experience these changes and fail because of it? 

It's important to meditate on this for some time as it has direct consequences to what your definition of a "unit" in the unit test really is. This is important not just at an individual level, but at an aggregate team level where we're consistent with our conceptual understanding of what comprises a unit.

##Rewriting the test correctly

This is an opinionated guide, so my defintion of a unit tells me that I should be able to comfortably rely on the actual implementations of underlying mappers/utils that transform the datastructures of choice. This means that in a "good" test I should not try to mock the appVarDataVOHelper's `getUCPFlowControlVO` method. Instead, I need to bear the onus of supplying an appropriate session.

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration
public class PaymentProviderHelperTest {

    @Configuration
    @Import(ProviderTestConfig.class)
    static class MyLocalConfig {

    }

    @Inject
    private AppVarDataVOHelper localAppVarDataVOHelper;

    @Inject
    private PaymentProviderHelper payProviderHelper;

    @Test
    public void testFirstPaymentAttempt() {
        // setup input context
        SessionVarDataVO sessionVarDataVO = getSessionWithUCP();
        UCPFlowControlVO ucpFlowControlVO = localAppVarDataVOHelper.getUCPFlowControlVO(sessionVarDataVO);
        ucpFlowControlVO.setPaymentAttempts(null); // explicitly set to null

        // call the method
        payProviderHelper.incrementAndValidatePaymentAttempts(sessionVarDataVO);

        // assert
        Assert.assertEquals(Long.valueOf(1l), ucpFlowControlVO.getPaymentAttempts());
    }

    @Test
    public void testPaymentAttemptNearThresholdLimit() {
        // setup input context and mock behavior
        SessionVarDataVO sessionVarDataVO = getSessionWithUCP();
        UCPFlowControlVO ucpFlowControlVO = localAppVarDataVOHelper.getUCPFlowControlVO(sessionVarDataVO);
        ucpFlowControlVO.setPaymentAttempts(PaymentProviderHelper.MAX_PAYMENT_ATTEMPTS - 1L); // explicitly set value

        // call the method
        payProviderHelper.incrementAndValidatePaymentAttempts(sessionVarDataVO);

        // assert
        Assert.assertEquals(Long.valueOf(PaymentProviderHelper.MAX_PAYMENT_ATTEMPTS),
                ucpFlowControlVO.getPaymentAttempts());
    }

    private SessionVarDataVO getSessionWithUCP() {
        SessionVarDataVO sessionVarDataVO = new SessionVarDataVO();
        UCPFlowControlVO ucpFlowControlVO = new UCPFlowControlVO();
        localAppVarDataVOHelper.setUCPFlowControlVOToSessionVarDataVO(sessionVarDataVO, ucpFlowControlVO);
        return sessionVarDataVO;
    }

}
```

Here's an example where we shift our attention to a separate method in this class called `sendMSPNotificationEvent`. Take a moment to review the method and think about the kinds of tests you could possible write. Here's one such test:

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration
public class PaymentProviderHelperTest {

    @Configuration
    @Import(ProviderTestConfig.class)
    static class MyLocalConfig {

        @Bean
        @Primary
        public NotificationService notificationService() {
            return notificationService;
        }
        
        @Bean
        @Primary
        public PAPSRequestContext papsRequestContext() {
            return papsRequestContext;
        }
    }

    @Inject
    private PaymentProviderHelper payProviderHelper;

    private static NotificationService notificationService;   
    private static PAPSRequestContext papsRequestContext;

    static {
        notificationService = mock(NotificationService.class);
        papsRequestContext = mock(PAPSRequestContext.class);
    }

    @Test
    public void testSendMSPNotificationEvent() {
        // setup input context
        SessionVarDataVO sessionVarDataVO = getSessionWithBuyerButNoShippingAddress();
        Map<String, Object> notificationDataMap = new HashMap<>();
        Payment payment = getPaymentWithTxns();
        // Note: Why do this instead of actually using a "real" papsRequestContext?
        when(papsRequestContext.getAttribute(PAPSAttributes.X_PAYPAL_SECURITY_CONTEXT)).thenReturn(
                getDummySecurityCtx());

        // call the method
        payProviderHelper.sendMSPNotificationEvent(sessionVarDataVO, payment, notificationDataMap);

        // Since we're not returning any value, we will assert the mutations applied to the map
        // and ensure that the the send method was called just once
        verify(notificationService, times(1)).sendNotification(notificationDataMap,
                sessionVarDataVO.getBuyer().getUser().getPaypalAccountNumber().toString(),
                Topic.EMAIL_BUYER_RECEIPT_MULTI_PAYMENT.getTopicName(), getDummySecurityCtx());
        assertEquals(notificationDataMap.keySet().size(), 3);
        assertEquals(notificationDataMap.get(NotificationKey.BUYER_NAME.getKeyName()), sessionVarDataVO.getBuyer()
                .getUser().getName());
        assertEquals(notificationDataMap.get(NotificationKey.PAYMENT_AMOUNT.getKeyName()), "0 "); // where did this come from? please refactor
        assertEquals(notificationDataMap.get(NotificationKey.IS_MULTI_TRANSACTION.getKeyName()), "true");
    }

    private Payment getPaymentWithTxns() {
        Payment payment = new Payment();
        Transaction txn1 = new Transaction();
        payment.setTransactions(Collections.singletonList(txn1));
        return payment;
    }

    private SessionVarDataVO getSessionWithBuyerButNoShippingAddress() {
        SessionVarDataVO sessionVarDataVO = new SessionVarDataVO();
        UserVO userVO = new UserVO();
        userVO.setPaypalAccountNumber(new BigInteger("1539671732305563784"));
        userVO.setName("Mogambo");
        BuyerVO buyerVO = new BuyerVO();
        buyerVO.setUser(userVO);
        sessionVarDataVO.setBuyer(buyerVO);
        return sessionVarDataVO;
    }
    
    private String getDummySecurityCtx() {
        return "{\"actor\":{\"account_number\":\"xxxx\",\"party_id\":\"xxxx\",\"auth_claims\":[\"AUTHORIZATION_CODE\"],\"auth_state\":\"ANONYMOUS\",\"client_id\":\"Aries\"},\"auth_token\":\"PD94bWwgdmVyc2lvbj0iMS4wIiBlbmNvZGluZz0iVVRGLTgiPz48VXNlcl9Vc2VyQXV0aFRva2VuVk8gaWQ9IjEiPjxhY2NvdW50X251bWJlciB0eXBlPSJ1aW50NjQiPjIwODgyNjQ0MzU1MTk3MjAwNTA8L2FjY291bnRfbnVtYmVyPjxsb2dpbl9pZCB0eXBlPSJ1aW50NjQiPjIxNDE2NzwvbG9naW5faWQ-PHNlc3Npb25fbnVtYmVyIHR5cGU9InVpbnQzMiI-MjQwMDQ0OTA2Mjwvc2Vzc2lvbl9udW1iZXI-PHNlc3Npb25fdGltZSB0eXBlPSJ1aW50MzIiPjE0MDM2NDY1MDI8L3Nlc3Npb25fdGltZT48YXV0aF90eXBlIHR5cGU9InNpbnQ4Ij44MjwvYXV0aF90eXBlPjxzZXNzaW9uX2lkIHR5cGU9IlN0cmluZyI-VEF2VDNTTlRMUVMtNmtkOUludk03SHExT3A4aVlWPC9zZXNzaW9uX2lkPjxhdXRoX2NyZWRlbnRpYWwgdHlwZT0ic2ludDgiPjgyPC9hdXRoX2NyZWRlbnRpYWw-PGF1dGhfbW9kaWZpZXJfZmxhZ3MgdHlwZT0idWludDMyIj4wPC9hdXRoX21vZGlmaWVyX2ZsYWdzPjxhdXRoX2xldmVsIHR5cGU9InVpbnQzMiI-MDwvYXV0aF9sZXZlbD48dG9rZW5fc3RhdHVzIHR5cGU9InNpbnQ4Ij44OTwvdG9rZW5fc3RhdHVzPjxjaGFubmVsIHR5cGU9IlN0cmluZyI-V0VCPC9jaGFubmVsPjwvVXNlcl9Vc2VyQXV0aFRva2VuVk8-\",\"auth_token_type\":\"ACCESS_TOKEN\",\"global_session_id\":\"Ssk0vAStTAIA\",\"last_validated\":1395350654,\"scopes\":[\"https://api.paypal.com/v1/payments/.*\",\"https://api.paypal.com/v1/vault/credit-card/.*\",\"openid\",\"https://uri.paypal.com/services/payments/futurepayments\",\"https://api.paypal.com/v1/vault/credit-card\",\"https://api.paypal.com/v1/payments/.*\"],\"subjects\":[{\"subject\":{\"account_number\":\"xxxx\",\"client_id\": \"Aries\",\"party_id\":\"xxxx\",\"auth_claims\":[\"PASSWORD\"],\"auth_state\":\"LOGGEDIN\"}}]}";
    }

}
```

You'll notice here that for a downstream service, we did indeed use a mock and softened our verification since it didn't have any return value. The reader is encouraged to test the `executePayment` method as an exercise that will bring all these concepts together.