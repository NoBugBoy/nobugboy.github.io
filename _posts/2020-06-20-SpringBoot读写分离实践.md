# SpringBoot读写分离实战
文章共6000字，纯文字的文章写的比较少，但是不多写一点很难让读者深入了解，所以请耐心看完，后面会有源码，基本上看明白原理了，复制粘贴即可

> 实际环境建议使用mysql5.7版本，8.0版本坑会比较多，如果使用云环境的mysql读写分离版本一般也基于5.7版本，但是搭建教程基本一致
> [基于mysql8+docker搭建主从集群文章地址](https://blog.csdn.net/Day_Day_No_Bug/article/details/103987741)

##  1 读写分离适用的场景

 1. 读多写少
 3. 并发量小
 4. 非强一致性场景

 当并发量大时，应使用缓存架构，而非加强数据库层吞吐能力，当大量并发进入数据库层，cpu直接会彪满，造成数据库卡死的现象，读写分离解决读的性能，水平扩展多台机器提升了整体读的能力。

## 2 读写分离缺点

 1. 数据冗余
 2. 一致性问题

 实现高可用的方式多以数据冗余的方式出现，这样当一台故障就可以迁移到另一台机器，而读写分离架构通过数据冗余的方式并未达到高可用，当主库故障时，仅能提供读的可用性，当读库故障时又需要重新手动配置同步，主从之间通过binlog进行同步数据，数据同步会有一定的延迟，导致读不出已写事物的数据现象，而从库不可以反向同步，当发现主从数据不一致时难以恢复一致（**从集群搭建测试完成后，就应该将读写的权限分开，任何人不可以手动对从库进行写操作，否则后期会引发一系列bug问题**），导致无法同步。

## 3 SpringBoot实现读写分离（基于MyBatis）
 核心是通过Spring提供的abstractRoutingDataSource实现切换数据源的功能

 ##### 1 基于@Aspect声明式实现
 实现原理，首先定义两个数据源（**或多个，这里讲两个，多个读数据源需要做自定义负载均衡策略决定使用哪个**），定义自己的RoutingDataSource并继承abstractRoutingDataSource，并配置自定义的routingDataSource，将读写数据源放入，配置好默认数据源等之后，配置切面，通过配置切入点表达式来进行读写切换，这里又可以自己定义想使用的方式，如果service层命名的规则定义的比较好可以通过名称来切换，也可以定义注解在方法上，进行切换，原理都是扫描到切入点，在service执行之前，对数据源进行切换（**所以切面的执行优先级一定要大于切换数据源**），否则会无效


---

 1 先定义一个routingDataSource,实现AbstractRoutingDataSource，并重写determineCurrentLookupKey方法即可，determineCurrentLookupKey方法是实际切换数据源的方法（**感兴趣可以看下源码，很简单**），下面的DbRwEnum就是一个简单枚举，定义了master和salve两种状态
 ```java
 @Slf4j
public class RoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        log.info("当前数据源: {}", Objects.isNull(DBContextHolder.get())? DbRwEnum.MASTER.getType():DbRwEnum.SLAVE.getType());
        return Objects.isNull(DBContextHolder.get())? DbRwEnum.MASTER.getType():DbRwEnum.SLAVE.getType();
    }

}
 ```
 2 定义读写的数据源，我这里用的hikaricp其余差别也不大
 ```java
 @Configuration
@Order(1)
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties("spring.datasource.master")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.slave")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().build();
    }
    @Primary
    @Bean(name = "routingDataSource")
    public DataSource routingDataSource(@Qualifier("masterDataSource") DataSource masterDataSource,
                                        @Qualifier("slaveDataSource") DataSource slaveDataSource) {
        Map<Object, Object> targetDataSources = new HashMap<>(2);
        targetDataSources.put(DbRwEnum.MASTER.getType(), masterDataSource);
        targetDataSources.put(DbRwEnum.SLAVE.getType(), slaveDataSource);
        RoutingDataSource routingDataSource = new RoutingDataSource();
        //默认数据源，当切面没有切到对应的方法或者其他情况会默认使用主数据源
        routingDataSource.setDefaultTargetDataSource(masterDataSource);
        routingDataSource.setTargetDataSources(targetDataSources);
        DBContextHolder.set(DbRwEnum.MASTER.getType());
        return routingDataSource;
    }}
 ```
 3 配置mybatis的sqlSessionFactory和事物
 ```java
     @Bean
    public MybatisSqlSessionFactoryBean sqlSessionFactory() throws Exception {
        MybatisSqlSessionFactoryBean sqlSessionFactoryBean = new MybatisSqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(routingDataSource);
        return sqlSessionFactoryBean;
    }
    @Bean
    @Primary
    public PlatformTransactionManager platformTransactionManager() {
        return new DataSourceTransactionManager(routingDataSource);
    }
 ```
 4 配置切换的上下文
 ```java
 public class DBContextHolder {
    //记录当前请求线程所持有的数据源信息
    private static final ThreadLocal<String> contextHolder = new ThreadLocal<>();

    public static void set(String dbType) {
        contextHolder.set(dbType);
    }

    public static String get() {
        return contextHolder.get();
    }
    //防止ThreadLocal内存泄漏，垃圾回收时弱引用只回收了key对应那块内存value的那块内存依然占有且不会被回收
    public static void clear() {
         contextHolder.remove();
    }

    public static void switchMaster() {
        set(DbRwEnum.MASTER.getType());
        log.info("数据源切换到master");
    }

    public static void switchSlave() {
        set(DbRwEnum.SLAVE.getType());
        log.info("数据源切换到slave");
    }
}
 ```
 5 配置切面以及表达式
 ```java
 @Aspect
@Order(-999)
public class SwitchDataSourceAspect {

    @Pointcut("@annotation(com.*.*.annotations.UseMaster)")
    public void masterPointcut() {}
	//提示一下这里注解不要加在接口上要加在实现类上
    @Pointcut("execution(public * com.*.*.service.impl.*.*(..)))")
    public void point(){}

    @Before("!masterPointcut() && point()")
    public void read() {
        DBContextHolder.switchSlave();
    }

    @Before("masterPointcut() && point()")
    public void write() {
        DBContextHolder.switchMaster();
    }
    @After("point() || masterPointcut()")
    public void after(JoinPoint p) {
        DBContextHolder.clear();
        log.info("清理 ");
    }
}
 ```
 到这里第一种方式就完成了，但是由于我是对旧工程进行改造，发现了一些问题


 1. 最严重的问题，无法通过order排序，某些方法执行顺序正确，某些方法永远都是先切换数据源再进行进入切面导致determineCurrentLookupKey方法永远返回null使用默认数据源，而且执行一个方法会切换多次，暂时不清楚具体原因可能由于shiro等过滤器导致。
 2. 由于最开始没有把权限分开导致了在测试期间某些方法未加注解，其实走了从库但是依然写了进去造成数据脏数据过多，无法恢复主从同步（看过开头就知道了不能反向同步，而且脏数据过多很难完全清理和主数据一致），所以开始就配置好操作从库的账号只有只读权限即可解决。

 ## 4 上面的问题可以通过编程式aop解决
 我怀疑是否是因为由于项目使用了MyBatisPlus框架导致有一些功能无法运作，最后官网提供了一种读写分离方案，我带着怀疑的态度看了一下源码，本质上和上面的实现方式并无过多差别，但是我用了一下发现居然好用？，见鬼了，所以我就借鉴了其中一部分源码，在对原读写分离架构的源码不过多修改的情况下，进行改造，最终可以达到想要的效果。

  ##### 2 基于编程式aop实现

  1 实现MethodInterceptor，实现代理切面（MethodInterceptor？如果玩过动态代理一看就知道这是cglib的实现方式，当然这里的MethodInterceptor是aopalliance提供的）
 ```java
 public class DataSourceInterceptor implements MethodInterceptor {
    @Override
    public Object invoke(MethodInvocation methodInvocation) throws Throwable {
        Method method = methodInvocation.getMethod();
        UseMaster annotation = method.getAnnotation(UseMaster.class);
        try {
            if(DbRwEnum.MASTER.equals(annotation.value())){
                DBContextHolder.switchMaster();
                return methodInvocation.proceed();
            }else{
                DBContextHolder.switchSlave();
                return methodInvocation.proceed();
            }
        }finally {
            DBContextHolder.clear();
        }
    }
}
 ```
 2 有了切面肯定要有切入点，既然我们已经使用了注解，那么就按照注解的方式继续实现,通过继承AbstractPointcutAdvisor来实现aop，这里定义了注解切入点AnnotationMatchingPointcut（平时我们用注解方式比较多很少使用这种方式，正好可以了解下这种方式，打开脑洞可以实现很多好玩的功能）
 ```java
 public class DataSourceAdvisor extends AbstractPointcutAdvisor {
    Advice advice;
    Pointcut pointcut;
    public DataSourceAdvisor(Advice advice){
        this.advice = advice;
        this.pointcut = buildPointcut();
    }
    @Override
    public Pointcut getPointcut() {
        return pointcut;
    }

    @Override
    public Advice getAdvice() {
        return advice;
    }

    private Pointcut buildPointcut() {
        Pointcut cpc = new AnnotationMatchingPointcut(UseMaster.class, true);
        Pointcut mpc = AnnotationMatchingPointcut.forMethodAnnotation(UseMaster.class);
        //组合切入点
        return new ComposablePointcut(cpc).union(mpc);
    }
}
 ```
 3 最后交给spring去管理就好了,设置一下执行优先级
 ```java
  @Bean
    public DataSourceAdvisor dynamicDatasourceAnnotationAdvisor() {
        DataSourceInterceptor interceptor = new DataSourceInterceptor();
        DataSourceAdvisor advisor = new DataSourceAdvisor(interceptor);
        advisor.setOrder(Ordered.HIGHEST_PRECEDENCE + 1);
        return advisor;
    }
 ```
最终通过了第二种方式解决了，执行顺序不对的问题，通过配置读账号权限解决了脏数据的问题，然后就可以正常的使用了。

---

***都看到这了喜欢的话点个关注，后续会有更多好玩的技术文章分享，感谢！***