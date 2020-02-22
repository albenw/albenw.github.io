title: 规则引擎EasyRules
author: alben.wong
abbrlink: 9c1674eb
tags:
  - 规则引擎
categories:
  - java
date: 2020-02-20 16:13:00
keywords: easyrules ruleengine
description: EasyRules是一个简单实用的规则引擎，Rule Engine的作用就是分离判断逻辑和对应的执行逻辑的，即不要写一堆的if...else代码，这样会让人很难理解与维护。EasyRules就可以帮你做到这点。
---
## 概要

EasyRules是一个简单实用的规则引擎，Rule Engine的作用就是分离判断逻辑和对应的执行逻辑的，即不要写一堆的if...else代码，这样会让人很难理解与维护。EasyRules就可以帮你做到这点。

Easy Rules的灵感来自于Martin Fowler的文章Should I use a Rules Engine。

文章中这句话正是描述了Easy Rules所做的事情：

>
>
>You can build a simple rules engine yourself. All you need is to create a bunch of objects with conditions and actions, store them in a collection, and run through them to evaluate the conditions and execute the actions.



## 应用场景

现在大多数企业公司的业务系统的业务规则都是很复杂而且变化很快的，开发人员又必须要快速的响应这些需求，那么如果代码写的不好，需要在复杂繁乱的规则中修改或新增，则容易不小心踩坑，引起线上bug。即使你小心翼翼的改代码，也会存在维护成本，导致效率低下。

hard code带来的问题：

![upload successful](/images/规则引擎EasyRules__0.png)

(来源：网络)



## 使用

EasyRules的使用还是比较简单的，不然就对不起easy这个字了。

使用前要清楚一些概念：规则（Rule）、条件（Condition）、行为（Action）。在一个规则中，达到了某个条件，就会触发一些行为。

例子：

以下是我定义的两个规则

```java
@Rule(name = "my rule1", description = "my rule description", priority = 1)
public class MyRule1 {

    @Condition
    public boolean when(@Fact("type") Integer type) {
        if(type == 1){
            return true;
        }
        return false;
    }

    @Action(order = 1)
    public void execute1(Facts facts) throws Exception {
        log.info("MyRule1 execute1, facts={}", facts);
    }

    @Action(order = 2)
    public void execute2(Facts facts) throws Exception {
        log.info("MyRule1 execute2, facts={}", facts);
    }
}

@Slf4j
@Service
@Rule(name = "my rule2", description = "my rule description", priority = 1)
public class MyRule2 {

    @Condition
    public boolean when(@Fact("type") Integer type) {
        if(type == 2){
            return true;
        }
        return false;
    }

    @Action(order = 1)
    public void execute(Facts facts) throws Exception {
        log.info("MyRule2 execute, facts={}", facts);
    }
}
```



跑起来试试：

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = { "classpath*:applicationContext.xml" })
public class EasyRulesTest {

    @Autowired
    private MyRule1 myRule1;
    @Autowired
    private MyRule2 myRule2;

    @Before
    public void init(){
    }

    @Test
    public void easyRuleWithSpringTest(){
        //参数
        RulesEngineParameters parameters = new RulesEngineParameters();
        parameters.skipOnFirstAppliedRule(true);
        parameters.skipOnFirstFailedRule(true);
        parameters.skipOnFirstNonTriggeredRule(true);
        parameters.priorityThreshold(3);

        RulesEngine rulesEngine = new DefaultRulesEngine(parameters);
        //创建规则
        Rules rules = new Rules();
        rules.register(myRule1);
        rules.register(myRule2);
        //设置真实数据
        Facts facts = new Facts();
        facts.put("type", 1);
        rulesEngine.fire(rules, facts);
    }
}

```

由于type==1，所以触发MyRule1的行为。很easy是吧。

除了这个简单的演示外，EasyRules还是支持：

- 编程式定义规则，其中包括Fluent API、MVEL
- 支持yml文件加载规则
- RuleListener
- 等等

更详细的使用方法请自行看官方文档



## 实现原理

在EasyRules中有两个核心的配置，一个是条件Condition，一个是执行方法Action。

EasyRules的实现是通过实现Rule接口，或者通过代理 + 注解来找到Condition和Action，再用Facts对象作为类似context来传递上下文的信息。

在使用的角度来看，EasyRules无非做了两件事情，一是加载规则，二是执行规则，下面我们从两个方面入手看看它的实现原理。（源码基于：org.jeasy:easy-rules-core:3.4.0）



### 加载规则

可以看到register用来注册或加载规则：

```java
     //规则集合
     private Set<Rule> rules = new TreeSet<>();
     /**
     * Register a new rule.
     *
     * @param rule to register
     */
    public void register(Object rule) {
        Objects.requireNonNull(rule);
        //根据传入规则生一个代理对象，这个代理对象统一为Rule类型
        rules.add(RuleProxy.asRule(rule));
    }
```



RuleProxy.asRule方法：

```java
    /**
     * Makes the rule object implement the {@link Rule} interface.
     *
     * @param rule the annotated rule object.
     * @return a proxy that implements the {@link Rule} interface.
     */
    public static Rule asRule(final Object rule) {
        Rule result;
        if (rule instanceof Rule) {
            //如果传入的对象为Rule类型，则直接返回
            result = (Rule) rule;
        } else {
            //主要校验有没有@Rule、@Condition、@Action等注解
            ruleDefinitionValidator.validateRuleDefinition(rule);
            //使用JDK代理生成代理对象
            //RuleProxy继承了InvocationHandler
            result = (Rule) Proxy.newProxyInstance(
                    Rule.class.getClassLoader(),
                    new Class[]{Rule.class, Comparable.class},
                    new RuleProxy(rule));
        }
        return result;
    }
```



RuleProxy的invoke方法：

我们重点看看代理后的evaluate和execute，它们分别代表Condition方法和Action方法

```java
    @Override
    public Object invoke(final Object proxy, final Method method, final Object[] args) throws Throwable {
        String methodName = method.getName();
        switch (methodName) {
            case "getName":
                return getRuleName();
            case "getDescription":
                return getRuleDescription();
            case "getPriority":
                return getRulePriority();
            case "compareTo":
                return compareToMethod(args);
            case "evaluate":
                return evaluateMethod(args);
            case "execute":
                return executeMethod(args);
            case "equals":
                return equalsMethod(args);
            case "hashCode":
                return hashCodeMethod();
            case "toString":
                return toStringMethod();
            default:
                return null;
        }
    }
    
    //evaluateMethod
    private Object evaluateMethod(final Object[] args) throws IllegalAccessException, InvocationTargetException {
        //传入的参数是固定的，第一个是Facts对象（看下面执行规则）
        Facts facts = (Facts) args[0];
        //获取带有@Condition注解的Method
        Method conditionMethod = getConditionMethod();
        try {
            //根据@Fact注解从facts对象里获取到真正的入参
            //从使用方法就大概知道它的原理，Facts相当于一个Map
            List<Object> actualParameters = getActualParameters(conditionMethod, facts);
            //调用真正的方法
            return conditionMethod.invoke(target, actualParameters.toArray()); 
        } catch (NoSuchFactException e) {
            LOGGER.error("Rule '{}' has been evaluated to false due to a declared but missing fact '{}' in {}",
                    getTargetClass().getName(), e.getMissingFact(), facts);
            return false;
        } catch (IllegalArgumentException e) {
            String error = "Types of injected facts in method '%s' in rule '%s' do not match parameters types";
            throw new RuntimeException(format(error, conditionMethod.getName(), getTargetClass().getName()), e);
        }
    }

    //executeMethod，这个方法的原理跟上面差不多
    private Object executeMethod(final Object[] args) throws IllegalAccessException, InvocationTargetException {
        Facts facts = (Facts) args[0];
        //从@Action注解获取真正的Method，然后获取到真正的入参进行调用
        for (ActionMethodOrderBean actionMethodBean : getActionMethodBeans()) {
            Method actionMethod = actionMethodBean.getMethod();
            List<Object> actualParameters = getActualParameters(actionMethod, facts);
            actionMethod.invoke(target, actualParameters.toArray());
        }
        return null;
    }
```



### 执行规则

执行规则看RulesEngine的fire方法：

```java
    @Override
    public void fire(Rules rules, Facts facts) {
        //调用RuleListener
        triggerListenersBeforeRules(rules, facts);
        doFire(rules, facts);
        //调用RuleListener
        triggerListenersAfterRules(rules, facts);
    }

    //doFire
    void doFire(Rules rules, Facts facts) {
        if (rules.isEmpty()) {
            LOGGER.warn("No rules registered! Nothing to apply");
            return;
        }
        logEngineParameters();
        log(rules);
        log(facts);
        LOGGER.debug("Rules evaluation started");
        //遍历每个规则
        for (Rule rule : rules) {
            final String name = rule.getName();
            final int priority = rule.getPriority();
            if (priority > parameters.getPriorityThreshold()) {
                LOGGER.debug("Rule priority threshold ({}) exceeded at rule '{}' with priority={}, next rules will be skipped",
                        parameters.getPriorityThreshold(), name, priority);
                break;
            }
            //根据before listener是否允许执行
            if (!shouldBeEvaluated(rule, facts)) {
                LOGGER.debug("Rule '{}' has been skipped before being evaluated",
                    name);
                continue;
            }
            //执行condition方法
            if (rule.evaluate(facts)) {
                LOGGER.debug("Rule '{}' triggered", name);
                triggerListenersAfterEvaluate(rule, facts, true);
                try {
                    triggerListenersBeforeExecute(rule, facts);
                    //执行action方法
                    rule.execute(facts);
                    LOGGER.debug("Rule '{}' performed successfully", name);
                    triggerListenersOnSuccess(rule, facts);
                    if (parameters.isSkipOnFirstAppliedRule()) {
                        LOGGER.debug("Next rules will be skipped since parameter skipOnFirstAppliedRule is set");
                        break;
                    }
                } catch (Exception exception) {
                    LOGGER.error("Rule '" + name + "' performed with error", exception);
                    triggerListenersOnFailure(rule, exception, facts);
                    if (parameters.isSkipOnFirstFailedRule()) {
                        LOGGER.debug("Next rules will be skipped since parameter skipOnFirstFailedRule is set");
                        break;
                    }
                }
            } else {
                LOGGER.debug("Rule '{}' has been evaluated to false, it has not been executed", name);
                triggerListenersAfterEvaluate(rule, facts, false);
                if (parameters.isSkipOnFirstNonTriggeredRule()) {
                    LOGGER.debug("Next rules will be skipped since parameter skipOnFirstNonTriggeredRule is set");
                    break;
                }
            }
        }
    }
```



## 规则引擎

EasyRules的特点就是easy，但是一般人们提到“规则引擎”可能是一个很“大”的东西，例如像drools那样，这种规则引擎无疑功能强大，但也很“笨重”，学习成本高。就像Martin Fowler说的，现在很多人都倾向于使用轻量级的、灵活的、embedded的规则引擎，总之合适的就是最好的。

一套完整的规则引擎可能是一个很复杂系统，它包括不限于建立规则语言标准，动态编辑，自动化执行等等，它还可以提供一个平台给业务、产品人员或者开发人员编写规则，根据特定的算法推理出模型。

所以，EasyRules是否你需要的呢？



## 参考资料

[**Should I use a Rules Engine?**](https://martinfowler.com/bliki/RulesEngine.html)

[规则引擎概述](https://blog.csdn.net/express_wind/article/details/77141674)

[Java规则引擎与其API(JSR-94)](https://www.ibm.com/developerworks/cn/java/j-java-rules/)



