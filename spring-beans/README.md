# Bean实例化过程

## 类职责

### 继承关系
AliasRegistry -> SimpleAliasRegistry
SingletonBeanRegistry | SimpleAliasRegistry -> DefaultSingletonBeanRegistry -> FactoryBeanRegistrySupport

## Bean定义

### 类继承结构

AbstractBeanDefinition -> BeanDefinition -> AttributeAccessor, BeanMetadataElement

### BeanDefinition

Bean定义（BeanDefinition接口）中包含的属性：

1. parentName - 父BeanDefinition的名称
2. beanClassName - BeanDefinition中定义的Bean的类名称
3. scope - BeanDefinition中定义的Bean的作用域
4. lazyInit - 是否延迟初始化
5. dependsOn - 当前Bean显式声明的依赖的Bean名称
6. autowireCandidate - 表示当前的Bean是否作为被自动注入其他Bean的候选Bean。只作用于基于类型（type-based）的自动注入，对于基于名称的自动注入不受影响。
7. primary - 当存在多个候选Bean可以注入的时候，表示是否为主要Bean
8. factoryBeanName - 工厂Bean的名称
9. factoryMethodName - 工厂方法名称
10. constructorArgumentValues - 显式定义的构造方法参数列表，可以在BeanFactory的post-processing阶段被修改。
11. propertyValues - 显式定义的setter成员列表，可以在BeanFactory的post-processing阶段被修改。
12. methodOverrides - 需要被重写的方法的列表，封装在`MethodOverrode`对象中。主要用于实现方法注入（Method Inject）和方法替换（Method Replace）。
13. initMethodName - Bean的初始化方法名称（从5.1开始引入）
14. destroyMethodName - Bean的析构方法名称（从5.1开始引入）
15. enforceInitMethod - 是否是默认的init初始化方法（默认值为：true）。用于标识配置的`initMethodName`是否是默认初始化方法，在Bean上单独配置的初始化方法会设置为默认值true，否则如果是从默认配置上获取值，则设置为false。
16. enforceDestroyMethod - 是否是默认的destroy方法（默认值为：true）。同`enforceInitMethod`。
17. synthetic - 是否为自动生成的Bean定义（非应用提供的Bean定义）。
18. role - 标识BeanDefinition定义的Bean的角色（ROLE_APPLICATION - 应用提供的Bean配置；ROLE_SUPPORT - 用于支持大的Bean的配置；ROLE_INFRASTRUCTURE - 支撑容器基础设施的Bean定义）。
19. description - Bean的备注。
20. resource - `Resource`对象，标识Bean定义的来源。

### SingletonBeanRegistry

负责维护单例Bean实例，在默认实现`DefaultSingletonBeanRegistry`中维护了单例的Bean。

1. 通过`singletonObjects`成员记录创建的单例。
2. 通过`singletonFactories`成员记录单例的`ObjectFactory`对象。
3. 通过`earlySingletonObjects`成员记录提前创建（未对成员进行初始化）的单例，用于解决Setter循环依赖注入的问题。
4. 通过`singletonsCurrentlyInCreation`成员记录正在被创建中的Bean，避免Bean被重复创建。
5. 通过`dependentBeanMap`和`dependenciesForBeanMap`维护创建过程中Bean的依赖关系。

### FactoryBeanRegistrySupport

`FactoryBean`通过`FactoryBeanRegistrySupport`维护，在`factoryBeanObjectCache`中维护了通过`FactoryBean`创建的单例Bean。`FactoryBeanRegistrySupport`负责从`FactoryBean`中创建出单例Bean，通过`BeanFactory`通过调用`FactoryBeanRegistrySupport`的`getObjectFromFactoryBean()`方法来获得单例Bean。


## Bean获取过程（主流程）

* 通过`AbstractBeanFactory`的`getBean()`方法获取Bean
    * 通过`AbstractBeanFactory`继承的`DefaultSingletonBeanRegistry`类的`getSingleton()`方法获取已经创建的单例Bean。
        * 在`getSingleton()`方法中，首先检查`singletonObjects`来获取已经创建的Bean。
        * 如果在`singletonObjects`中没有找到，则表示单例Bean还没有被创建，检查`earlySingletonObjects`以获取提前被创建的单例（提前被创建的单例是指：Bean已经通过构造方法创建，但是Bean实例还没有被初始化，Bean的属性还没有设置）
        * 如果在`earlySingletonObjects`中也未找到提前创建的Bean，则检查`singletonFactories`中是否有对应Bean的对象工厂`ObjectFactory`实现。如果有则通过这个对象工厂创建出该Bean实例，如果`doGetBean()`方法的`allowEarlyReference = true`，则将这个未完全初始化的Bean实例添加到`earlySingletonObjects`中，并删除对应的对象工厂，返回这个未初始化完全的Bean实例。
        * 否则返回空，表示在Bean注册表中没有该单例的Bean。
    * 判断从`getSingleton()`方法返回的Bean实例，如果不为空并且构造参数`args`为空（表示获取的是默认创建的单例），则返回该Bean实例。
        * 调用`AbstractBeanFactory`的`getObjectForBeanInstance()`方法获取真正的Bean实例。在这个方法中会处理普通Bean实例和FactoryBean实例。
    * 如果从`getSingleton()`方法中没有返回Bean实例，则创建Bean实例
        * 如果是Prototype作用域的Bean，由于不存在循环注入的情况，所以需要判断是否正在创建中，如果正在创建中则需要抛出正在创建异常。
        * 检查需要创建的Bean的BeanDefinition是否在当前BeanFactory中，如果不存在，则从父BeanFactory中查找（如果父BeanFactory存在的话）。
        * 通过`getMergedLocalBeanDefinition()`生成合并（处理Abstract BeanDefinition）后的`RootBeanDefinition`。
        * 通过`checkMergedBeanDefinition()`检查生成的RootBeanDefinition，检查该BeanDefinition是否为`Abstract`，如果是`Abstract`则抛出异常。
        * 处理BeanDefinition中声明的`dependsOn`属性。
            * 通过`isDependent()`检查是否存在循环依赖（依赖于两个存放依赖关系的Map：`dependentBeanMap` 和 `dependenciesForBeanMap`）。
            * 在`dependentBeanMap`（Key表示被依赖的Bean，Value表示依赖Key的Bean名称集合）和`dependenciesForBeanMap`（Key表示依赖的Bean名称，Value表示Key依赖的Bean的名称集合）中添加依赖关系。
            * 通过`getBean()`获取依赖的Bean，这个过程会触发被依赖的Bean的创建过程（基于依赖关系进行递归创建）。
        * 通过BeanDefinition判断Bean是否是单例，如果是单例则进行单例的创建
            * 调用`getSingleton()`方法返回指定Bean名称的单例，参数包含了一个`ObjectFactory`工厂对象（在`ObjectFactory`中的`getObject()`方法中通过在子类`AbstractAutowireCapableBeanFactory`中实现的`createBean()`方法进行Bean的创建）用于在Bean不存在的时候创建单例Bean。
                * 对`singletonObjects`加锁，保证获取（创建单例）的过程是线程安全的。
                * 尝试从`singletonObjects`中获取Bean，如果Bean已经被创建则直接返回。
                * 否则进行单例Bean的创建
                    * 检查BeanFactory的状态是否在析构中，如果是则抛出异常。
                    * 通过`beforeSingletonCreation()`标记正在创建中的Bean。
                    * 调用传入的`ObjectFactory`工厂方法创建单例Bean，并标记`newSingleton = true`。
                    * 调用`afterSingletonCreation()`将Bean的创建中标记清除。
                    * 判断`newSingleton = true`来判断Bean是否创建成功，如果成功则调用`addSingleton()`将Bean添加到BeanFactory中
                        * 添加Bean到`singletonObjects`中
                        * 删除缓存在`singletonFactories`中的`ObjectFactory`对象
                        * 删除缓存在`earlySingletonObjects`中提前创建的对象
                        * 在`registeredSingletons`中注册创建成功的Bean
                    * 返回创建的单例Bean
            * 调用`getObjectForBeanInstance()`对返回的单例Bean进行处理，返回真正的Bean实例（处理`FactoryBean`的情况）
        * 如果需要创建的Bean是Prototype，则进行Prototype Bean的创建
            * 在创建Prototype作用域的Bean之前先通过`beforePrototypeCreation()`记录正在被创建的Bean
                * 在`prototypesCurrentlyInCreation`这个`ThreadLocal`中存储了正在被创建的Bean，如果当前线程正在创建多个Prototype的Bean，则在`prototypesCurrentlyInCreation`中存储的是一个`Set`类型的集合，集合中存放了bean的名称。
            * 直接通过`createBean()`创建Bean。
            * 调用`afterPrototypeCreation()`将正在创建的Prototype作用域的Bean从`prototypesCurrentlyInCreation`中删除。
            * 调用`getObjectForBeanInstance()`对返回的单例Bean进行处理，返回真正的Bean实例（处理`FactoryBean`的情况）
        * 如果作用域是通过`Scope`定义的自定义的作用域，则创建特定作用域的Bean
            * 从Bean的BeanDefinition中获取作用域Scope的名称。
            * 通过该作用域名称从`scopes`中获取注册的`Scope`对象。
            * 调用`Scope`的`get()`方法从指定作用域中获取Bean实例，如果Bean实例不存在，则通过提供的回调参数`ObjectFactory`创建该Bean。
                * 在传入的`ObjectFactory`实现中，Bean的创建过程和创建Prototype作用域的Bean一样。
    * 返回被创建成功的Bean实例

备注：

1. 在Spring的BeanFactory实现（AbstractBeanFactory）中，默认只有两种作用域：单例（Singleton）和原型（Prototype）。其中Singleton的Bean由BeanFactory自己保存（通过`BeanFactory`继承的`SingletonBeanRegistry`来存储），而Prototype作用域的Bean则在每次需要的时候临时创建，不会缓存起来。对于介于Singleton和Prototype两种作用域之间的Beann，本质上也是Prototype作用域的一种，只不过是作用域内部是Singleton，在作用域之间是Prototype（可以认为Singleton是全局的作用域）。Spring在实现上，通过将这些Bean保存在注册到容器的Scope对象中来实现作用域内部的共享和作用域之间的隔离。所以，如果将Bean的存储从作用域的视角去看，那么`Scope`在职责上充当了一部分`SingletonBeanRegistry`的功能。

## Bean创建过程

Bean在获取的过程中被创建，Spring框架通过`ObjectFactory`接口将Bean的创建过程从Bean的存储中（`DefaultSingletonBeanRegistry.getSingleton(beanName, singletonFactory)`）解耦出来。在`AbstractAutowireCapableBeanFactory`中通过`createBean()`完成对Bean实例的创建。创建过程如下：

* 调用`createBean()`方法，传入bean名称，Bean定义`RootBeanDefinition`以及创建Bean需要的构造参数。
    * 通过`resolveBeanClass()`解析Bean的Class对象。
    * 预处理Bean定义中的方法重写（调用`AbstractBeanDefinition`中的`prepareMethodOverrides()`方法）。方法重写（MethodOverride）用于定义需要被容器重写的Bean方法，目前在Spring框架中主要用于方法注入（Method Injection）和方法替换（Method Replacement），分别对应`LookupOverride`和`ReplaceOverride`。
    * 调用`resolveBeforeInstantiation()`方法使用注册在容器中的`InstantiationAwareBeanPostProcessor`后置处理器的`postProcessBeforeInstantiation()`方法对即将创建的Bean实例进行前置处理。Spring AOP在这个阶段通过在`postProcessBeforeInstantiation()`中返回创建的Bean的方式直接跳过后续的Bean创建过程（短路，short-circuited），接下来的Bean会跳过Bean创建过程，直接返回，后续只会被`InstantiationAwareBeanPostProcessor`的`postProcessAfterInstantiation()`方法处理。
    * 调用`doCreateBean()`进行真正的Bean创建过程。
        * 检查`factoryBeanInstanceCache`中的缓存，是否包含已经创建的`FactoryBean`实例（由于FactoryBean在创建Bean的过程中只需要一次，所以这里通过`remove()`的方式获取到已经存在的FactoryBean）。
        * 如果不存在，则调用`createBeanInstance()`创建Bean实例的`BeanWrapper`对象。在这里会进行构造方法的依赖注入。
            * 通过`resolveBeanClass()`确保获取解析到的Bean的Class对象。
            * 检查Bean的Class对象的访问权限，是否和BeanDefinition的要求一致，是否为public，如果不是则抛出异常。
            * 检查BeanDefinition的InstanceSupplier是否设置，如果有则通过这个Suppiler获取到对应的Bean实例。
            * 检查BeanDefinition是否配置了工厂方法，如果是则调用`instantiateUsingFactoryMethod()`通过工厂方法创建Bean实例（在该方法中，创建过程委托给了`ConstructorResolver`的`instantiateUsingFactoryMethod()`方法来实现）。
            * 通过Bean的Class的构造方法来创建Bean（接下来开始解析Bean的构造方法和用于创建Bean的构造参数）。
                * 首先在BeanDefinition中检查是否已经解析过（通过BeanDefinition的`resolvedConstructorOrFactoryMethod`参数是否为空来判断）。如果`resolvedConstructorOrFactoryMethod`不为空，则标记当前BeanDefinition已经被解析过，也就是说该Bean的构造方法和构造参数曾经已经解析过了，不需要再次解析。
                * 如果BeanDefinition标记为被解析过，则通过判断是否需要构造方法自动注入（`autowireNecessary == true`）来决定采用何种创建Bean的方式：
                    * 如果需要构造方法自动注入，则选择`autowireConstructor()`来创建Bean实例。见 <构造方法自动注入构造过程>
                    * 否则，则表示构造过程不需要传入构造参数，通过`instantiateBean()`创建Bean实例。见<默认构造方法实例化过程>
                * 如果未被解析过，则开始进行构造方法和构造参数的解析过程：
                    * 首先利用注册的`BeanPostProcessor`来解析Bean的构造方法。这里用到的是`SmartInstantiationAwareBeanPostProcessor`（Spring容器内部的一类BeanPostProcessor后置处理器，扩展自`InstantiationAwareBeanPostProcessor`这类内部的后置处理器）后置处理器的`determineCandidateConstructors()`方法来解析被标记为候选构造器的列表。
                    * 检查是否需要进行构造方法注入：
                        * `ctros`是否非空（通过BeanPostProcessor识别出来的设置了注解（@Autowire）的构造方法）
                        * Bean定义是否开启了构造方法自动注入
                        * Bean定义是否定义了构造注入参数
                        * 创建Bean的时候传入的参数是否不为空
                    * 如果满足上述条件，则通过`autowireConstructor()`进行构造方法注入，否则尝试从RootBeanDefinition中获取推荐的构造方法列表（通过`getPreferredConstructors()`获取）。如果推荐的构造方法列表不为空，则通过`autowireConstructor()`进行构造方法注入。
                * 如果不需要进行构造方法注入，则通过无参构造方法进行默认的实例创建过程（`instantiateBean()`）。

### 构造方法自动注入构造过程

### 默认构造方法实例化过程

### Bean实例化过程中依赖的组件

#### PropertyValues和MutablePropertyValues

`PropertyValues`是`PropertyValue`的容器，包含了多个`PropertyValue`对象。`PropertyValue`对象是一个包含了`<name, value>`的元组，用于表示Bean的属性（Bean property），用于替代普通的Map对象来存储Bean属性。在`PropertyValue`中并不需要处理值的类型转换问题，Spring框架会在`BeanWrapperImpl`中集中处理值的类型转换。

`MutablePropertyValues`是`PropertyValues`默认实现，允许修改PropertyValues的值，支持通过构造方法从Map构造或深拷贝。

#### BeanDefinitionValueResolver

一个用于帮助解析Bean依赖的工具类，将值解析成对应的Bean依赖。解析过程会调用BeanFactory获取Bean实例，这个过程会触发Bean的创建。解析逻辑在`BeanDefinitionValueResolver`的`resolveValueIfNecessary`方法中。

#### ConstructorResolver

用于解析Bean的构造方法和工厂方法。包括对构造方法工厂方法的解析，方法参数的解析。

构造`ConstructorResolver`的时候需要传入`AbstractAutowireCapableBeanFactory`对象，用于帮助解析构造方法的参数依赖。

支持功能：

1. 构造方法自动注入（`autowireConstructor()`）
2. Bean实例化（`instantiate()`）
3. 解析工厂方法（`resolveFactoryMethodIfPossible()`）
4. 通过解析的工厂方法创建Bean实例（`instantiateUsingFactoryMethod()`）

##### 构造方法自动注入过程

`ConstructorResolver#autowireConstructor()`方法实现了构造方法自动注入的逻辑，注入过程如下：

1. 调用`beanFactory.initBeanWrapper()`初始化BeanWrapperImpl对象（配置PropertyEditor和ConversionService）
2. 解析缓存在BeanDefinition中的实例化Bean需要的构造方法参数
3. 解析实例化Bean需要的构造方法和构造方法参数（如果在步骤2没有解析出来）



#### ParameterNameDiscoverer

方法或者构造方法的参数名称解析器，在BeanFactory中用于对Bean的方法或构造方法进行名称解析。Spring默认使用了`DefaultParameterNameDiscoverer`。`DefaultParameterNameDiscoverer`解析器继承了`PrioritizedParameterNameDiscoverer`类，用于将多个方法名称解析器组合到一起使用。默认包含了`StandardReflectionParameterNameDiscoverer`和`LocalVariableTableParameterNameDiscoverer`两个解析器。其中`StandardReflectionParameterNameDiscoverer`通过反射的方式获取参数名称，利用JDK 8的`-parameters`编译参数来开启。`LocalVariableTableParameterNameDiscoverer`通过读取Class文件并解析其中的本地变量表获取名称。

#### DependencyDescriptor

封装了构造方法或者普通方法中依赖注入的参数或基于字段注入的字段值。

#### AutowireCandidateResolver

自动注入候选值解析器。一个策略接口，用于解析自动注入的候选值。基于注解的依赖注入场景用到了实现类`ContextAnnotationAutowireCandidateResolver`。

该解析器由`DefaultListableBeanFactory`维护，通过`setAutowireCandidateResolver()`设置，通过`getAutowireCandidateResolver()`获取。

