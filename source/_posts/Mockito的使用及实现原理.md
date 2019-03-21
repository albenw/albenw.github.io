title: Mockito的使用及实现原理
author: alben.wong
abbrlink: 9758301d
tags:
  - mockito
  - mock
categories:
  - java
  - mockito
date: 2019-03-19 19:20:00
keywords: mockito原理 mock
description: Mockito是一个强大的mock工具，本文将重点讲述Mockito的基本使用及注意事项，以及其实现mock的原理。
---
## 概要
Mockito是一个强大的mock工具，本文将重点讲述Mockito的基本使用及注意事项，以及其实现mock的原理。

## 使用

### 应用场景
- 开发完成之后，我们都需要经过测试才敢把代码发布到线上。一个很普遍的问题是，我们要测试的类可能会有很多对上游依赖，这些依赖包括类、对象、资源、数据等等，正常来说如果我们缺少这些依赖的话，我们将无法完成测试。这时候，我们一般的处理方法是mock，mock本地方法的返回，mock上游接口的返回等等，使这些方法返回假定的数据，好让我们流程可以继续走下去，完成测试。
- Mock对于TDD来说也是很重要的一个功能。
- 关于Mock在单元测试中的作用，可以看看[Martin Fowler对它的讨论](https://martinfowler.com/articles/mocksArentStubs.html)

### 简单例子
事例代码基于Mockito2.18.3版本。
通常来说Mockito分为两大功能点，一是verify验证，一是stub打桩（我不知道怎么翻译好，跟立flag差不多意思）
先看看一个入门的例子
```java
    @Test
    public void mockitoTest(){
        //生成一个mock对象
        List<String> mockedList = Mockito.mock(List.class);
        //打印mock对象的类名，看看mock对象为何物
        System.out.println(mockedList.getClass().getName());
        //操作mock对象
        mockedList.add("one");
        mockedList.get(0);
        mockedList.clear();
        //verify验证，mock对象是否发生过某些行为
        //如果验证不通过，Mockito会抛出异常
        Mockito.verify(mockedList).add("one");
        Mockito.verify(mockedList).get(0);
        //stub打桩，指定mock对象某些行为的返回，也就是我们常用的mock数据
        Mockito.when(mockedList.get(1)).thenReturn("13");
        //这里打印出“13”（但我们知道mockedList实际是没有下标为1的元素，这就是mock的功能）
        System.out.println(mockedList.get(1));
    }
```
Mockito的使用很简单吧！相信经过上述代码的演示还有注释的说明，相信大家已经对Mockito有一个大概的认识。
当然，上述代码只是演示了Mockito的一小部分功能，它还有更多、更强大的功能，我这里不将一一介绍。更多API请参考[官方文档](https://static.javadoc.io/org.mockito/mockito-core/2.25.1/org/mockito/Mockito.html)

### 注解使用
在我们的工程中，90%的情况都是基于Spring，使用JUnit作为单元测试框架，我们看看Mockito怎么与它们结合一起使用。
看一个完整的JUnit单元测试类的例子（假设InjectMockService依赖MockServiceA和SpyServiceB）
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = { "classpath*:applicationContext.xml" })
public class MockitoTest {

    @Autowired
    private CommonService commonService;

    @Mock
    private MockServiceA mockServiceA;

    @Autowired
    @Spy
    private SpyServiceB spyServiceB;

    @Autowired
	@InjectMocks
	private InjectMockService injectMockService; 

    @Before
    public void mockInit(){
    	MockitoAnnotations.initMocks(this);
        MockServiceData();
        //...and so on
    }

    private void MockServiceData() {
        Object mockDataYouWantReturn = generateMockData();
        Mockito.when(mockServiceA.methodYouWantMock()).thenReturn(mockDataYouWantReturn);
    }

}
```
上述代码是我瞎编的，没有跑过。不过一般的使用场景也应该长的类似。
对Mockito的注解说明一下：
- @Mock，你需要mock的对象，其原理跟上述提到的`Mockito.mock(List.class);`差不多，即把注解标记的这个对象转换成Mockito的对象
- @Spy，你需要进行部分mock的对象。这个Spy（翻译就是间谍啦）跟Mock相比，Mock是对对象的所有方法都进行mock处理的，即方法不会真正的执行，举之前的那个例子，`mockedList.add("one");`执行后不会把“one”加入到List中，但如果`mockedList`是一个Spy的话就可以，即把`Mockito.mock(List.class);`改为`Mockito.spy(ArrayList.class);`，那么add方法就会真正的执行，这里把List改为ArrayList是因为如果遇到abstract方法也不会执行的。Spy是有使用场景的，当我们需要mock一部分方法，而另外的方法需要正常执行时就需要用到Spy了，不然你就要mock全部用到的方法了
- @InjectMocks，你需要注入@Mock对象的对象，即@InjectMocks这个对象依赖其他mock对象。这个有点像依赖注入，它就是用来解决这个问题的。举个例子说明，ServiceB依赖ServiceA，你是需要测试ServiceB中的某个方法，但是这个方法依赖到ServiceA了，ServiceA的返回你不可控，你需要mock它，这时就要把@Mock作用于ServiceA，@InjectMocks作用于ServiceB。

### 注意踩坑
这是我在使用Mockito遇到的一些坑，希望大家在实践中也注意一下。

#### @InjectMocks
@InjectMocks的作用是把@Mock的对象注入到属性中去，我们如果只写成（假设InjectMockService依赖MockServiceA）
```java
    @InjectMocks
	private InjectMockService injectMockService; 

    @Mock
    private MockServiceA mockServiceA;
```
这样的话，虽然InjectMockService会做初始化，而且MockServiceA会被注入到InjectMockService里，但是InjectMockService中其他的依赖对象会为null，因为你都没做其他初始化动作嘛。这时你就需要跟Spring的注入一起用，加上@Autowired注解，如下所示
```java
    @InjectMocks
    @Autowired
	private InjectMockService injectMockService; 
```
从上Mockito的初始化得知，即上述代码的`mockInit()`方法，Mockito的初始化是在Spring的后面的，所以InjectMockService会被Spring初始化，然后再被Mockito修改依赖的Mock对象。

#### 深层次对象
实际应用时，如果你想mock的对象在比较深的调用层次，那么做法可以是：
获取依赖这个对象的对象（通过Spring的@Autowired），即mock对象的上一层对象，然后使用一些手段把mock对象替换调。
例如使用`ReflectionTestUtils.setField(upperObject, "fieldName", mockObject);`把mock设置进去替换掉原来的对象。

#### Spy对象执行报错
在上面的例子介绍的用法中
`Mockito.when(mockedList.get(1)).thenReturn("13");`
但是如果mockedList是一个Spy对象的话，可能会有问题。
这是因为@Spy对象会真正的实际执行该方法（@Mock则不会），但这是你要mock的方法，那么就有可能有问题。所以如果你不想方法实际执行，需要改变一下用法：
//不会调用stub的方法
`Mockito.doReturn(false).when(spyJack).go();`

#### 与Spring AOP
如果你Mock的对象被Spring AOP进行过处理，例如加了`@Transactional`或者自己做了切面等等，这时Spring会生成代理对象，这时要注意对你Mock对象有没有产生影响。

## 实现原理
源码基于Mockito2.18.3版本。
上面介绍了Mockito的基本用法及注意事项。
通过API的使用，大家一定很好奇Mockito究竟是怎么实现的。
我们先尝试通过表象来推测其实现原理，然后再分析代码证实猜想。

### 表象猜测
如果你有多做测试，细心观察，就会发现通过Mockito生成的mock对象是一个“假”对象。
对于文章开头的例子，我们可以发现
- `mockedList.add("one");`不会实际往list增加一个“one”的元素（Spy则会），所以add之后你再get也是得不到结果的。即你无论怎么操作list都是徒劳的。
- `Mockito.verify(mockedList).add("one");`这句很明显就是会校验list之前执行过的方法及入参。
- `Mockito.when(mockedList.get(1)).thenReturn("13");`这句看上有点奇怪，怎么会用`mockedList.get(1)`作为入参呢？其实想想就知道不可能的，所以这里应该也是跟上面一样，只是get方法做了记录，然后再return值。

### 生成Mock对象
通过上面的猜测，我们很容易猜到Mockito用了代理生成了mock对象。
我们直接跟踪`Mockito.mock`方法，到`MockitoCore`的mock方法
```java
    public <T> T mock(Class<T> typeToMock, MockSettings settings) {
        if (!MockSettingsImpl.class.isInstance(settings)) {
            throw new IllegalArgumentException("Unexpected implementation of '" + settings.getClass().getCanonicalName() + "'\n" + "At the moment, you cannot provide your own implementations of that class.");
        }
        MockSettingsImpl impl = MockSettingsImpl.class.cast(settings);
        MockCreationSettings<T> creationSettings = impl.build(typeToMock);
        //创建mock对象
        T mock = createMock(creationSettings);
        mockingProgress().mockingStarted(mock, creationSettings);
        return mock;
    }
```
`MockUtil`的createMock方法
```java
    public static <T> T createMock(MockCreationSettings<T> settings) {
        //创建一个MockHandler，很关键的一个类
        MockHandler mockHandler =  createMockHandler(settings);
        //创建mock对象，这里是调用SubclassByteBuddyMockMaker的createMock
        T mock = mockMaker.createMock(settings, mockHandler);

        Object spiedInstance = settings.getSpiedInstance();
        if (spiedInstance != null) {
            new LenientCopyTool().copyToMock(spiedInstance, mock);
        }
        return mock;
    }

    //MockHandlerFactory
    //这里包了很多层，但是handle最终是调用MockHandlerImpl的handle方法
    //MockHandlerImpl是核心中核心类，等下会讲到
    public static <T> MockHandler<T> createMockHandler(MockCreationSettings<T> settings) {
        MockHandler<T> handler = new MockHandlerImpl<T>(settings);
        MockHandler<T> nullResultGuardian = new NullResultGuardian<T>(handler);
        return new InvocationNotifierHandler<T>(nullResultGuardian, settings);
    }

```
`SubclassByteBuddyMockMaker`
```java
    @Override
    public <T> T createMock(MockCreationSettings<T> settings, MockHandler handler) {
        //生成代理类，调用SubclassBytecodeGenerator的mockClass
        Class<? extends T> mockedProxyType = createMockType(settings);

        Instantiator instantiator = Plugins.getInstantiatorProvider().getInstantiator(settings);
        T mockInstance = null;
        try {
            //为代理类生成实例对象
            mockInstance = instantiator.newInstance(mockedProxyType);
            MockAccess mockAccess = (MockAccess) mockInstance;
            //MockMethodInterceptor-代理类的拦截器
            mockAccess.setMockitoInterceptor(new MockMethodInterceptor(handler, settings));

            return ensureMockIsAssignableToMockedType(settings, mockInstance);
        } catch (ClassCastException cce) {
            throw new MockitoException(join(
                    "ClassCastException occurred while creating the mockito mock :",
                    "  class to mock : " + describeClass(settings.getTypeToMock()),
                    "  created class : " + describeClass(mockedProxyType),
                    "  proxy instance class : " + describeClass(mockInstance),
                    "  instance creation by : " + instantiator.getClass().getSimpleName(),
                    "",
                    "You might experience classloading issues, please ask the mockito mailing-list.",
                    ""
            ), cce);
        } catch (org.mockito.creation.instance.InstantiationException e) {
            throw new MockitoException("Unable to create mock instance of type '" + mockedProxyType.getSuperclass().getSimpleName() + "'", e);
        }
    }
```
`SubclassBytecodeGenerator`
```java
    @Override
    public <T> Class<? extends T> mockClass(MockFeatures<T> features) {
        String name = nameFor(features.mockedType);
        DynamicType.Builder<T> builder =
                byteBuddy.subclass(features.mockedType)
                         .name(name)
                         .ignoreAlso(isGroovyMethod())
                         .annotateType(features.stripAnnotations
                             ? new Annotation[0]
                             : features.mockedType.getAnnotations())
                         .implement(new ArrayList<Type>(features.interfaces))
                         .method(matcher)
                           .intercept(to(DispatcherDefaultingToRealMethod.class))
                           .transform(withModifiers(SynchronizationState.PLAIN))
                           .attribute(features.stripAnnotations
                               ? MethodAttributeAppender.NoOp.INSTANCE
                               : INCLUDING_RECEIVER)
                         .method(isHashCode())
                           .intercept(to(MockMethodInterceptor.ForHashCode.class))
                         .method(isEquals())
                           .intercept(to(MockMethodInterceptor.ForEquals.class))
                         .serialVersionUid(42L)
                         .defineField("mockitoInterceptor", MockMethodInterceptor.class, PRIVATE)
                         .implement(MockAccess.class)
                           .intercept(FieldAccessor.ofBeanProperty());
        if (features.serializableMode == SerializableMode.ACROSS_CLASSLOADERS) {
            builder = builder.implement(CrossClassLoaderSerializableMock.class)
                             .intercept(to(MockMethodInterceptor.ForWriteReplace.class));
        }
        if (readReplace != null) {
            builder = builder.defineMethod("readObject", void.class, Visibility.PRIVATE)
                    .withParameters(ObjectInputStream.class)
                    .throwing(ClassNotFoundException.class, IOException.class)
                    .intercept(readReplace);
        }
        ClassLoader classLoader = new MultipleParentClassLoader.Builder()
            .append(features.mockedType)
            .append(features.interfaces)
            .append(currentThread().getContextClassLoader())
            .append(MockAccess.class, DispatcherDefaultingToRealMethod.class)
            .append(MockMethodInterceptor.class,
                MockMethodInterceptor.ForHashCode.class,
                MockMethodInterceptor.ForEquals.class).build(MockMethodInterceptor.class.getClassLoader());
        if (classLoader != features.mockedType.getClassLoader()) {
            assertVisibility(features.mockedType);
            for (Class<?> iFace : features.interfaces) {
                assertVisibility(iFace);
            }
            builder = builder.ignoreAlso(isPackagePrivate()
                .or(returns(isPackagePrivate()))
                .or(hasParameters(whereAny(hasType(isPackagePrivate())))));
        }
        return builder.make()
                      .load(classLoader, loader.resolveStrategy(features.mockedType, classLoader, name.startsWith(CODEGEN_PACKAGE)))
                      .getLoaded();
    }
```
生成代理类的关键代码。Mockito使用的是ByteBuddy这个框架，它并不需要编译器的帮助，而是直接生成class，然后使用ClassLoader来进行加载，感兴趣的可以深入研究，其地址为:https://github.com/raphw/byte-buddy。如果你有使用过其他字节码生成框架，如Cglib或Javassist，就可以大概猜到这里做了什么事情。
这里很重要的一点就是把`MockMethodInterceptor`作为代理类的拦截器。
最后就是生成字节码然后动态加载到JVM中。

### Mockito生成的代理类
那么Mockito利用ByteBuddy生成的代理类是长怎么样子的，才能知道它是怎么做拦截的。就好像你看到JDK Proxy生成的代理类，就会更清楚InvocationHandler的实现一样。
还是文章开头的例子

![upload successful](/images/Mockito的使用及实现原理__1.png)

我们可以看到几点关键的
- 持有一个MockMethodInterceptor对象
- 继承了MockAccess
- List原来的方法都被DispatcherDefaultingToRealMethod拦截了


### 代理拦截interceptor
好，我们现在有目标了。
`DispatcherDefaultingToRealMethod`是`MockMethodInterceptor`的内部类，最终它还是调用MockMethodInterceptor的doIntercept
```java
    Object doIntercept(Object mock,
                       Method invokedMethod,
                       Object[] arguments,
                       RealMethod realMethod,
                       Location location) throws Throwable {
        //最终调用的是MockHandler的handle方法
        //MockHandler最终还是调用MockHandlerImpl的，上面提过了。
        //MockHandlerImpl是核心，下面会讲
        return handler.handle(createInvocation(mock, invokedMethod, arguments, realMethod, mockCreationSettings, location));
    }
```

### verify
说了mock对象是如何生成的，就可以开始说用法的实现了。
上面我们猜测verify是对调用做了记录，下面我们来证实一下。

#### verify start
`Mockito`的verify
```java
    @CheckReturnValue
    public static <T> T verify(T mock) {
        return MOCKITO_CORE.verify(mock, times(1));
    }
    @CheckReturnValue
    public static VerificationMode times(int wantedNumberOfInvocations) {
        //通过VerificationModeFactory创建了一个VerificationMode
        //VerificationMode在verify行为中是一个重要的概念。这里times表示次数，因为verify还可以校验调用了多少次
        return VerificationModeFactory.times(wantedNumberOfInvocations);
    }
```
`VerificationMode`有很多中类型，代表了verify的种类。

![upload successful](/images/Mockito的使用及实现原理__0.png)

从名字中大概可以猜到有什么用，大家可以去尝试一下。

继续跟踪verify到`MockitoCore`的verify
```java
    public <T> T verify(T mock, VerificationMode mode) {
        if (mock == null) {
            throw nullPassedToVerify();
        }
        MockingDetails mockingDetails = mockingDetails(mock);
        if (!mockingDetails.isMock()) {
            throw notAMockPassedToVerify(mock.getClass());
        }
        MockHandler handler = mockingDetails.getMockHandler();
        //通知listener
        mock = (T) VerificationStartedNotifier.notifyVerificationStarted(
            handler.getMockSettings().getVerificationStartedListeners(), mockingDetails);
        //初始化ThreadLocal，这里是MockingProgressImpl。这个对象的作用可以理解为mock的“进度”，无论verify还是stub都经过它
        MockingProgress mockingProgress = mockingProgress();
        
        VerificationMode actualMode = mockingProgress.maybeVerifyLazily(mode);
        //标记verify开始。new一个MockAwareVerificationMode，实际还是开头创建的那个VerificationMode，只是封装了一下。
        mockingProgress.verificationStarted(new MockAwareVerificationMode(mock, actualMode, mockingProgress.verificationListeners()));
        return mock;
    }
```
`MockingProgressImpl`的verificationStarted
```java
    public void verificationStarted(VerificationMode verify) {
        //校验一下状态
        validateState();
        //还会重置stub
        resetOngoingStubbing();
        //把MockingProgressImpl的verificationMode设置为开头创建的那个
        verificationMode = new Localized(verify);
    }
```
这里注意，MockingProgressImpl是ThreadLocal的，而且它的verificationMode只有一个，就说明verify之后紧接着的mock调用就是针对这次verify的，如果多次verify的话，后者会覆盖前者。
verify start就这么多了，最重要的就是记录`VerificationMode`。

#### verify match
记录了VerificationMode，那么如果下次调用怎么去匹配是刚刚的verify的呢？
上面说了所有mock对象的方法调用都会被`MockMethodInterceptor`拦截，而MockMethodInterceptor最终会调到`MockHandlerImpl`的handle方法，一直憋了很久的MockHandlerImpl终于要出场了。
这里除了verify，还有stub的，因为这个方法是包含了mock相关的所有核心逻辑了。
```java
//Invocation即InterceptedInvocation，把代理类拦截时的方法调用即参数封装在一起
//它包含一下几个对象：真正的方法realMethod，Mockito的方法MockitoMethod，参数arguments，以及mock对象mockRef
/** 
    private final MockReference<Object> mockRef;
    private final MockitoMethod mockitoMethod;
    private final Object[] arguments, rawArguments;
    private final RealMethod realMethod;
*/
public Object handle(Invocation invocation) throws Throwable {
        //stub
        if (invocationContainer.hasAnswersForStubbing()) {
            // stubbing voids with doThrow() or doAnswer() style
            InvocationMatcher invocationMatcher = matchersBinder.bindMatchers(
                    mockingProgress().getArgumentMatcherStorage(),
                    invocation
            );
            invocationContainer.setMethodForStubbing(invocationMatcher);
            return null;
        }
        //获取VerificationMode，VerificationMode至多一个
        VerificationMode verificationMode = mockingProgress().pullVerificationMode();
        
        InvocationMatcher invocationMatcher = matchersBinder.bindMatchers(
                mockingProgress().getArgumentMatcherStorage(),
                invocation
        );
        //校验状态，如果处于不争取的状态，会抛异常
        mockingProgress().validateState();

        //如果之前调用过verify方法的话，这里就不为null
        // if verificationMode is not null then someone is doing verify()
        if (verificationMode != null) {
            // We need to check if verification was started on the correct mock
            // - see VerifyingWithAnExtraCallToADifferentMockTest (bug 138)
            if (((MockAwareVerificationMode) verificationMode).getMock() == invocation.getMock()) {
                VerificationDataImpl data = createVerificationData(invocationContainer, invocationMatcher);
                //最终调用VerificationMode的verify去验证
                verificationMode.verify(data);
                return null;
            } else {
                // this means there is an invocation on a different mock. Re-adding verification mode
                // - see VerifyingWithAnExtraCallToADifferentMockTest (bug 138)
                mockingProgress().verificationStarted(verificationMode);
            }
        }

        //下面的代码等下会讲到

        // prepare invocation for stubbing
        invocationContainer.setInvocationForPotentialStubbing(invocationMatcher);
        OngoingStubbingImpl<T> ongoingStubbing = new OngoingStubbingImpl<T>(invocationContainer);
        //记下ongoingStubbing，每次调用都会刷新最新的ongoingStubbing
        mockingProgress().reportOngoingStubbing(ongoingStubbing);

        // look for existing answer for this invocation
        StubbedInvocationMatcher stubbing = invocationContainer.findAnswerFor(invocation);
        notifyStubbedAnswerLookup(invocation, stubbing);

        if (stubbing != null) {
            stubbing.captureArgumentsFrom(invocation);
            try {
                //这里是对stub的返回，即thenReturn的返回，下面的代码会讲到
                return stubbing.answer(invocation);
            } finally {
                //Needed so that we correctly isolate stubbings in some scenarios
                //see MockitoStubbedCallInAnswerTest or issue #1279
                mockingProgress().reportOngoingStubbing(ongoingStubbing);
            }
        } else {
            //下面代码意思是对mock对象的默认返回
            //测试中我们知道，mock对象的每一个操作都是“无效”的，而且都会返回一个相应类型的默认值（stub除外）。
            Object ret = mockSettings.getDefaultAnswer().answer(invocation);
            DefaultAnswerValidator.validateReturnValueFor(invocation, ret);

            //Mockito uses it to redo setting invocation for potential stubbing in case of partial mocks / spies.
            //Without it, the real method inside 'when' might have delegated to other self method
            //and overwrite the intended stubbed method with a different one.
            //This means we would be stubbing a wrong method.
            //Typically this would led to runtime exception that validates return type with stubbed method signature.
            invocationContainer.resetInvocationForPotentialStubbing(invocationMatcher);
            return ret;
        }
    }
```
好，我们看看Times这个VerificationMode的verify方法
```java
    @Override
    public void verify(VerificationData data) {
        //获取所有的调用
        List<Invocation> invocations = data.getAllInvocations();
        //获取的verify的调用
        MatchableInvocation wanted = data.getTarget();

        if (wantedCount > 0) {
            //两者做匹配
            checkMissingInvocation(data.getAllInvocations(), data.getTarget());
        }
        checkNumberOfInvocations(invocations, wanted, wantedCount);
    }

```
`MissingInvocationChecker`
```java
    public static void checkMissingInvocation(List<Invocation> invocations, MatchableInvocation wanted) {
        //匹配所有的调用和目标调用
        List<Invocation> actualInvocations = findInvocations(invocations, wanted);
        //如果找到则“安全”返回return
        if (!actualInvocations.isEmpty()){
            return;
        }
        //否则会抛出异常

        Invocation similar = findSimilarInvocation(invocations, wanted);
        if (similar == null) {
            throw wantedButNotInvoked(wanted, invocations);
        }

        Integer[] indexesOfSuspiciousArgs = getSuspiciouslyNotMatchingArgsIndexes(wanted.getMatchers(), similar.getArguments());
        SmartPrinter smartPrinter = new SmartPrinter(wanted, similar, indexesOfSuspiciousArgs);
        throw argumentsAreDifferent(smartPrinter.getWanted(), smartPrinter.getActual(), similar.getLocation());
    }

```
通过调试发现在`InvocationMatcher`判断两个Invocation是否匹配
```java
@Override
    public boolean matches(Invocation candidate) {
        //可以看出就是通过判断mock对象是否一样，方法及参数是否一样
        return invocation.getMock().equals(candidate.getMock()) && hasSameMethod(candidate) && argumentsMatch(candidate);
    }
```
verify的过程就是这样了，如果找到匹配的Invocation则正常返回，找不到Mockito是会抛出异常提示信息，像assert那样子，大家可以尝试一下。

### stub
stub就是可以指定返回，一般我们用的更多，从上面verify的实现过程，我们应该也能大概猜到stub也有点类似。

#### when
跟踪when方法直到`MockitoCore`的when
```java
    public <T> OngoingStubbing<T> when(T methodCall) {
        //初始化MockingProgressImpl
        MockingProgress mockingProgress = mockingProgress();
        //标记stub开始
        mockingProgress.stubbingStarted();
        //获取最新的ongoingStubbing，这个ongoingStubbing会在每次MockHandlerImpl的handle都会新建一个，然后set到ThreadLocal的mockingProgress中，所以这里取出来的就是上一次的调用，这里也证明了其实when的参数是没用的，只要mock对象有方法调用就可以了。
        @SuppressWarnings("unchecked")
        OngoingStubbing<T> stubbing = (OngoingStubbing<T>) mockingProgress.pullOngoingStubbing();
        if (stubbing == null) {
            mockingProgress.reset();
            throw missingMethodInvocation();
        }
        //返回OngoingStubbing
        return stubbing;
    }
```
when方法就是返回上次mock方法调用封装好的OngoingStubbing。

#### thenReturn
thenReturn是BaseStubbing的方法，其实它也是一个OngoingStubbing
OngoingStubbing的继承关系

![upload successful](/images/Mockito的使用及实现原理__2.png)

```java
    @Override
    public OngoingStubbing<T> thenReturn(T value) {
        return thenAnswer(new Returns(value));
    }

    //OngoingStubbingImpl
    @Override
    public OngoingStubbing<T> thenAnswer(Answer<?> answer) {
        if(!invocationContainer.hasInvocationForPotentialStubbing()) {
            throw incorrectUseOfApi();
        }
        //把这个answer加入到
        invocationContainer.addAnswer(answer);
        return new ConsecutiveStubbing<T>(invocationContainer);
    }

    //InvocationContainerImpl
    public StubbedInvocationMatcher addAnswer(Answer answer, boolean isConsecutive) {
        //获取stub的调用
        Invocation invocation = invocationForStubbing.getInvocation();
        //标记stub完成
        mockingProgress().stubbingCompleted();
        if (answer instanceof ValidableAnswer) {
            ((ValidableAnswer) answer).validateFor(invocation);
        }
        //把stub的调用和answer加到StubbedInvocationMatcher的list
        //这里的意思就是把调用和返回绑定，如果下次调用匹配到了，就返回对应的answer
        synchronized (stubbed) {
            if (isConsecutive) {
                stubbed.getFirst().addAnswer(answer);
            } else {
                stubbed.addFirst(new StubbedInvocationMatcher(invocationForStubbing, answer));
            }
            return stubbed.getFirst();
        }
    }
```

那么下次的调用怎么匹配到answer呢？
还是在`MockHandlerImpl`的handle，这里只截图相关代码
```java
    if (stubbing != null) {
            stubbing.captureArgumentsFrom(invocation);
            try {
                //就是返回stub的answer方法
                return stubbing.answer(invocation);
            } finally {
                //Needed so that we correctly isolate stubbings in some scenarios
                //see MockitoStubbedCallInAnswerTest or issue #1279
                mockingProgress().reportOngoingStubbing(ongoingStubbing);
            }
        }

    //StubbedInvocationMatcher
    public Object answer(InvocationOnMock invocation) throws Throwable {
        //see ThreadsShareGenerouslyStubbedMockTest
        Answer a;
        //从之前放进来的answers获取最新的那个
        synchronized(answers) {
            a = answers.size() == 1 ? answers.peek() : answers.poll();
        }
        return a.answer(invocation);
    }      

```
先看看Answer的继承关系，看看Mockito支持什么样的对stub的返回

![upload successful](/images/Mockito的使用及实现原理__3.png)

我们用回刚刚的例子，对应的Answer就是Returns
```java
    public Object answer(InvocationOnMock invocation) throws Throwable {
        //就是直接返回我们指定的值
        return value;
    }
```

## 优点
Mockito最大的优点是其简单，实用，优雅的API，这是作者的初衷。
Java的Mock工具还有JMock，easymock等等，大家可以对比一下。

## 总结
本文介绍了Mockito的基本使用及注意事项，相信看了之后你已经可以在日常工作中用上。还源码分析了其实现mock的原理，通过源码看到，表面看上去挺神奇的一个东西，其实现也不是很难的，主要用到代理+状态来控制mock对象。


## 参考资料
https://infoq.cn/article/mockito-design
https://blog.saymagic.cn/2016/09/17/understand-mockito.html