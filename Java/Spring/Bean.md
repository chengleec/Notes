### BeanFactory 与 ApplicationContext

* BeanFactory 是 ApplicationContext 的父接口
* BeanFactory 是 Spring 的核心容器，ApplicationContext 实现了 BeanFactory，并对其做了一些扩展

### BeanFactory 实现

BeanFactory 实现的功能比较单一（`getBean`），需要借助各种实现类来完成各项功能

* BeanFactory 不会主动调用 BeanFactory 后处理器，需要手动注册
* BeanFactory 不会主动添加 Bean 后处理器，需要手动添加（Bean 后处理器会有排序的逻辑）
* BeanFactory 不会主动初始化单例，会延迟初始化，用到时再创建
* BeanFactory 不会解析 ${ } 与 #{ }

### ApplicationContext 实现

* ClassPathXmlApplicationContext()<了解>
* FileSystemXmlApplicationContext()<了解>
* AnnotationConfigApplicationContext()
* AnnotationConfigServletWebServerApplicationContext()<Web环境>

### Bean 生命周期

* 构造方法
* 依赖注入
* 初始化
* 销毁

### Bean 后处理器

为 Bean 生命周期各个阶段提供扩展，帮助解析各种注解关键字

* AutoWiredAnnotationBeanPostProcessor：`@AutoWired`，`@Value`
* CommonAnnotationBeanPostProcessor：`@Resource`，`@PostConstruct`（初始化前），`@PreDestroy`（销毁前）
* ConfigurationPropertiesBindingPostProcessor：`@ConfigurationProperties`（初始化前）
* ConfigurationClassPostProcessor：`@Bean`，`@ComponentScan`，`@Import`，`@ImportResource`

**TODO**：Bean 后处理器的执行流程

### Bean 的初始化和销毁方法

#### 初始化（优先级从高到低）

* `@PostConstruct`注解
* 实现 InitializingBean 接口
* `@Bean(initMethod = “init3”)`

#### 销毁（优先级从高到低）

* `@PreDestroy` 注解
* 实现 DisposableBean 接口
* `@Bean(destroyMethod = “destroy3”)`