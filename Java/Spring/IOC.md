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

### Scope

* singleton：单例，从容器中获取的 bean 每次都是同一个对象

* prototype：每次调用创建一个新的 bean 对象

* request：同一个请求创建一个实例

* session：同一个 session 创建一个实例

* appplication：同一个 web 应用程序创建一个实例

> 单例对象内部的多例对象创建时是同一个对象，而不是多例对象，对于单例对象而言，依赖注入只发生了一次，后续没有用到多例的对象，因此始终是第一次依赖注入的内容
>
> 解决：
>
> * 使用`@Lazy`解决
> * 代理对象虽然还是同一个，但当每次使用代理对象的任意方法时，由代理创建新的多例对象