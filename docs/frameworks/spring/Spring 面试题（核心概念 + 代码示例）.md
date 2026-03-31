# Spring 面试题（核心概念 + 代码示例）

## 1. IOC 与 DI

### 理论

- **IOC（控制反转）**：将对象的创建和管理权交给 Spring 容器，你不再需要手动 `new` 对象。容器负责创建、装配、管理对象的整个生命周期。
- **DI（依赖注入）**：IOC 的具体实现方式。容器在创建对象时，自动将对象所依赖的其他对象注入进去。

**注入方式**  
- 字段注入（`@Autowired`）  
- Setter 注入  
- 构造器注入（推荐，便于测试且避免循环依赖）

**注入策略**  
- `byType`：按类型匹配（`@Autowired` 默认）  
- `byName`：按名称匹配  
- `@Resource` 默认按名称，若找不到对应名称的 bean 则按类型

### 代码示例

```java
@Component
public class UserService {
    private final OrderService orderService;

    // 构造器注入（推荐）
    public UserService(OrderService orderService) {
        this.orderService = orderService;
    }

    public void doSomething() {
        orderService.createOrder();
    }
}

@Component
public class OrderService {
    // 字段注入
    @Autowired
    private UserService userService;

    public void createOrder() {
        System.out.println("创建订单");
    }
}
```

---

## 2. Spring Bean 的生命周期

### 理论

1. **实例化**：通过构造器（或工厂方法）创建对象实例（属性为 null）。  
2. **属性赋值**：调用 `populateBean` 完成依赖注入。  
3. **初始化前**：调用 `BeanPostProcessor.postProcessBeforeInitialization`。  
4. **初始化**：执行 `InitializingBean.afterPropertiesSet` 或自定义的 `init-method`。  
5. **初始化后**：调用 `BeanPostProcessor.postProcessAfterInitialization`（AOP 代理在此生成）。  
6. **使用**：从容器获取 bean。  
7. **销毁**：执行 `DisposableBean.destroy` 或自定义的 `destroy-method`。



### 代码示例

```java
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.stereotype.Component;

/**
 * @author wrmeng
 * @create 2026-03-31 -23:44
 **/

@Component
public class LifecycleBean implements InitializingBean, DisposableBean {

    private String dependency = "默认值（字段初始化）"; // 模拟依赖注入

    public LifecycleBean() {
        System.out.println("1. 实例化（构造函数）");
    }

    /**
     * 模拟依赖注入阶段
     * Spring 会在属性赋值时调用 setter，但 String 类型需要手动配置或使用@Value
     */
    public void setDependency(String dependency) {
        this.dependency = dependency;
        System.out.println("2. 依赖注入（setter 方法）- 实际值：" + dependency);
    }

    @PostConstruct
    public void init() {
        System.out.println("3. @PostConstruct 初始化（JSR-250 注解）");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("4. afterPropertiesSet（InitializingBean 接口）");
    }

    /**
     * 自定义初始化方法
     * 需要在配置类中使用 @Bean(initMethod = "customInit") 来启用
     */
    public void customInit() {
        System.out.println("5. 自定义 init-method（@Bean 配置）");
    }

    @PreDestroy
    public void destroyMethod() {
        System.out.println("6. @PreDestroy 销毁注解）");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("7. DisposableBean destroy（DisposableBean 接口）");
    }

    /**
     * 自定义销毁方法
     * 需要在配置类中使用 @Bean(destroyMethod = "customDestroy") 来启用
     */
    public void customDestroy() {
        System.out.println("8. 自定义 destroy-method（@Bean 配置）");
    }
}
```

配置类指定自定义方法（可选）：
```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * Bean 生命周期演示配置
 * @author wrmeng
 * @create 2026-03-31 -23:50
 **/
@Configuration
public class LifecycleConfig {

    /**
     * 配置 LifecycleBean，启用自定义初始化和销毁方法
     * 并使用@Value 注入 String 值
     */
    @Bean(initMethod = "customInit", destroyMethod = "customDestroy")
    public LifecycleBean lifecycleBean(@Value("测试依赖值") String dependency) {
        LifecycleBean bean = new LifecycleBean();
        bean.setDependency(dependency);
        return bean;
    }
}

```

---

## 3. BeanFactory vs FactoryBean

### 理论

- **BeanFactory**：Spring 容器的根接口，定义了获取 bean 的基本规范，是“大管家”。  
- **FactoryBean**：一种特殊的 bean，用于创建其他 bean，适合复杂对象的构建（如代理、连接池）。通过实现 `getObject()` 方法自定义创建逻辑。

**获取 FactoryBean 本身**：在 bean 名称前加 `&` 前缀。

### 代码示例

```java
@Component
public class MyFactoryBean implements FactoryBean<ComplexObject> {

    @Override
    public ComplexObject getObject() throws Exception {
        ComplexObject obj = new ComplexObject();
        obj.setName("created by factory");
        return obj;
    }

    @Override
    public Class<?> getObjectType() {
        return ComplexObject.class;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
}

public class ComplexObject {
    private String name;
    // getter/setter
}
```

使用：
```java
@Autowired
private ApplicationContext context;

public void demo() {
    // 获取 FactoryBean 本身
    FactoryBean<?> factory = (FactoryBean<?>) context.getBean("&myFactoryBean");
    // 获取 FactoryBean 创建的对象
    ComplexObject obj = context.getBean(ComplexObject.class);
}
```

---

## 4. 循环依赖与三级缓存

### 理论

**三级缓存**  
- **一级缓存 (`singletonObjects`)**：存放完全初始化好的成品 bean。  
- **二级缓存 (`earlySingletonObjects`)**：存放半成品 bean（实例化但未填充属性）。  
- **三级缓存 (`singletonFactories`)**：存放 `ObjectFactory`，可延迟决定返回原始对象还是代理对象。

**为什么需要三级缓存？**  
当 A 需要 AOP 增强时，B 在填充属性时通过三级缓存获取 A 的工厂，工厂可根据需要返回代理对象，避免 B 持有原始对象而失效。三级缓存实现了“延迟决定返回哪种对象”。

### 代码示例

```java
@Component
public class A {
    private B b;

    @Autowired
    public void setB(B b) {
        this.b = b;
    }
}

@Component
public class B {
    private A a;

    @Autowired
    public void setA(A a) {
        this.a = a;
    }
}
```

Spring Boot 3.x 默认禁止循环依赖，可通过配置临时允许：
```properties
spring.main.allow-circular-references=true
```

---

## 5. AOP 底层原理

### 理论

- **实现方式**：有接口 → JDK 动态代理；无接口 → CGLIB 字节码增强。  
- **流程**：容器通过 `BeanPostProcessor` 识别需要增强的 bean，生成代理对象；方法调用时，代理对象将通知（Advice）组装成拦截器链，依次执行，最终调用目标方法。

### 代码示例（记录方法执行时间）

```java
@Aspect
@Component
public class TimeAspect {

    @Around("@annotation(TimeMonitor)")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = joinPoint.proceed();
        long cost = System.currentTimeMillis() - start;
        System.out.println(joinPoint.getSignature() + " 耗时: " + cost + "ms");
        return result;
    }
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TimeMonitor {
}

@Service
public class SomeService {
    @TimeMonitor
    public void slowMethod() throws InterruptedException {
        Thread.sleep(100);
    }
}
```

---

## 6. Spring 事务如何回滚

### 理论

事务回滚本质由 `TransactionInterceptor` 实现：  
1. 根据事务属性开启事务，获取数据库连接，关闭自动提交。  
2. 执行业务代码。  
3. 若抛出符合回滚规则的异常（默认 `RuntimeException` 和 `Error`），则调用 `doRollback()` 回滚；否则提交。  
4. 清理事务上下文。

### 代码示例

```java
@Service
@Transactional
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;

    public void createOrder(Order order) {
        orderRepository.save(order);
        if (order.getAmount() < 0) {
            throw new RuntimeException("金额不能为负"); // 触发回滚
        }
    }
}
```

测试回滚：
```java
@SpringBootTest
public class OrderServiceTest {
    @Autowired
    private OrderService orderService;

    @Test
    void testRollback() {
        Order order = new Order();
        order.setAmount(-10);
        assertThrows(RuntimeException.class, () -> orderService.createOrder(order));
        // 验证数据库中没有该订单
    }
}
```

---

## 7. 事务传播特性

### 理论

Spring 在 `TransactionDefinition` 中定义了 7 种传播行为：  

| 传播属性           | 行为描述                                                     |
| ------------------ | ------------------------------------------------------------ |
| `REQUIRED`（默认） | 当前有事务则加入，无则新建                                   |
| `SUPPORTS`         | 当前有事务则加入，无则以非事务方式运行                       |
| `MANDATORY`        | 当前必须有事务，否则抛异常                                   |
| `REQUIRES_NEW`     | 挂起当前事务，新建独立事务                                   |
| `NOT_SUPPORTED`    | 挂起当前事务，以非事务方式运行                               |
| `NEVER`            | 当前有事务则抛异常，否则以非事务方式运行                     |
| `NESTED`           | 当前有事务则创建嵌套事务（Savepoint），内层回滚可回滚到 Savepoint，外层回滚则整个回滚 |

### 代码示例（REQUIRED + REQUIRES_NEW）

```java
@Service
public class OuterService {

    @Autowired
    private InnerService innerService;

    @Transactional(propagation = Propagation.REQUIRED)
    public void outerMethod() {
        // 插入数据...
        innerService.innerMethod();  // 新事务
        throw new RuntimeException("外层异常"); // 外层回滚
    }
}

@Service
public class InnerService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void innerMethod() {
        // 插入数据，独立事务，不受外层影响
    }
}
```

**结果**：`innerMethod` 提交的数据保留，`outerMethod` 插入的数据回滚。

---

