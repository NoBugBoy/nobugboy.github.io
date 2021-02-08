# 从refresh方法开始,了解ioc,bean生命周期

> 该文章内容所使用的spring版本为：spring-core-5.2.4.RELEASE.jar

---

## 1. 这里我们开始从Spring容器的创建开始讲起，了解refresh方法都做了什么，不会非常细致的走每一步，重载和实现太多了，这里主要讲xml的形式

---
***我们先创建spring上下文对象,它继承了AbstractApplicationContext，AbstractApplicationContext有多种子类实现，xml配置方式，注解方式等，用来获取bean的定义信息（BeanDefinition）***
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524164930972.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)

***如果是注解方式创建上下文使用扫描配置类，或扫描包的形式获取BeanDefinition***

> AnnotationConfigApplicationContext

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524170446653.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)

***扫描到之后直接注册到BeanFactory的map里***

> DefaultListableBeanFactory

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524174336775.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)

***如果是Xml方式，调用setConfigLocationsh获取一个或多个xml配置文件***

> ClassPathXmlApplicationContext

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524171313774.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
***解析占位符，替换一些属性，这里和注解方式不一样，并没有直接将xml去直接解析为BeanDefinition，然后注册到BeanFactory的map里***

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524173644791.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)

---
***无论是哪一种实现都要调用refresh刷新容器（所以好多面试都是会问到这个东西），接下来就看一下里面都干了什么***

###    1 refresh进来之后的第一个方法，做了一些初始化时间和设置标志位，实例化Environment并且做了一些校验

 > AbstractApplicationContext <<< 当前位置

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200525184323776.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)


 ### 2 refresh进来后二个方法**关键**，实例化了beanFactory，创建**BeanDefinitionReader加载BeanDefinition**
 Loads the bean definitions via an XmlBeanDefinitionReader.
> AbstractXmlApplicationContext <<< 当前位置

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200525184216350.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)

 走到这里，用IO读取了配置文件，包装成document对象
 Load bean definitions from the specified XML file.

> XmlBeanDefinitionReader <<< 当前位置 
>
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200525192025104.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)

 从document中get出element对象循环去解析
  Register each bean definition within the given root {@code <beans/>} element.

> DefaultBeanDefinitionDocumentReader <<< 当前位置

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200525193111991.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)

  (**重点**) 一路向下，将解析出来的BeanDefinition放入DefaultListableBeanFactory的map中
> DefaultListableBeanFactory  <<< 当前位置
>
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200525212418420.png)
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200525212452763.png)

---
## 2. 看到这里就知道了，ClassPathXmlApplicationContext通过Io读取Xml文件，并且解析为BeanDefinition，然后在Bean在实例化之前，将Bean定义信息存放在DefaultListableBeanFactory的map当中保留，Bean工厂也已经实例化完成，后续的方法会需要传入这个工厂对象


  上面一直在提BeanDefinition，下面就例举一些BeanDefinition接口中的方法

```java
//是否单例
boolean isSingleton();
//是否多例
boolean isPrototype();
//是否懒加载
boolean isLazyInit();
//是否是主要对象
boolean isPrimary();
//初始化方法名称
String getInitMethodName();
//销毁方法名称
String getDestroyMethodName();
```
### 3 refresh进来后 第三个方法就是对刚实例化好的bean工厂设置一些属性，需要注意的是这里设置了一个**beanPostProcessor**(增强器)，实例化了一个ApplicationListenerDetector将当前上下文对象传递进去，之后会判断是否是ApplicationListener的实现，是的话加入上下文中

> AbstractApplicationContext <<< 当前位置

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200525222500309.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200525222823556.png)

### 4  refresh进来后 第四个方法是空方法，需要自定义实现BeanFactoryPostProcessor接口，重写postProcessBeanFactory方法，将实例化完成的BeanFactory传入，并对其进行一些增强处理操作
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200526203121355.png)

### 5 refresh进来后 第五个方法 就是按照一定的优先级，去调用所有注册到容器中的BeanFactoryPostProcessor的postProcessBeanFactory方法


> First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
> Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
> Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
>
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200528211618950.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
### 6 refresh进来后 第六个方法将经过反复加工的bean工厂，和当前上下文对象（applicationContext）传入,注册BeanPostProcessors（和BeanFactoryPostProcessor **不要搞混了，不是一个东西** ，BeanPostProcessor用的和问的比较多，对bean初始化前后增强就是靠它，后面会讲）
（这里提醒下，beanPostProcessor听上去是后置处理器，不要以为是后置处理器就是初始化之后 执行的类，要点进去看里面的方法，有一个befor和after，也就是前后都是这个类使用方法区分前后执行的）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200528211723739.png)
> PostProcessorRegistrationDelegate <<< 当前位置

根据类型在之前注册好存放BeanDefinition的map中去找符合的对象，返回BeanName
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200528213534163.png)
接下来还是按照排序和优先级分类注册到集合中，最后顺序循环集合添加到BeanFactory中的List<BeanPostProcessor> beanPostProcessors = new CopyOnWriteArrayList<>();

> First, register the BeanPostProcessors that implement PriorityOrdered.
> Next, register the BeanPostProcessors that implement Ordered.
> Finally, re-register all internal BeanPostProcessors.

### refresh进来后 第七 ，第8 个方法放一起说吧，
第七个方法注册了解析消息（比如i18n国际化需要MessageSource）
第八个方法注册了多播器，Spring提供的事件机制中的ApplicationListener监听器需要注册在这里
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200528221900475.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
### 9 refresh进来后 第九个方法是一个空方法，Spring预留给子类使用，子类可以实现通过重写onRefresh做一些操作
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200528222736851.png)
### 10 refresh进来后 第十个方法，根据类型在之前注册好存放BeanDefinition的map中去找符合的ApplicationListener对象


> 看着是不是有点眼熟，上面的beanPostProcessor一样，通过类型获取beanName并放入集合中
> （***到这里的bean都未实例化哦，依然是均为BeanDefinition***）
> AbstractApplicationContext <<< 当前位置

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200528223525351.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
### 11 refresh进来后 第十一个方法 finishBeanFactoryInitialization，终于到了关键的实例化bean的方法

> 直接跳到这行，进入实现 beanFactory.preInstantiateSingletons();
> DefaultListableBeanFactory <<< 当前位置
>
>  进入getBean方法中，再进入doGetBean方法中 
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530134206485.png)
> ![在这里插入图片描述](https://img-blog.csdnimg.cn/2020053013422112.png)
> **找到createBean方法，然后进入doCreateBean ，所有do  method才是实际执行的内容**

>   在这个方法里实例化了bean的时候并填充beanPostProcessor
>   Central method of this class: creates a bean instance,
>    populates the bean instance, applies post-processors, etc.
>    AbstractAutowireCapableBeanFactory <<< 当前位置

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530141353710.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530142401444.png)
**实例化bean，并包装为BeanWrapper**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530150537501.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
**通过构造器的反射创建bean的实例**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530150923505.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530150822308.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
**populateBean方法，注入属性（ioc）**

> Populate the bean instance in the given BeanWrapper with the property values

**根据名称和类型装配**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530164632278.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
**解析pvs注入bean**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530170030959.png)

找到initializeBean方法，(初始化bean前后方法的执行顺序就在这里) （生命周期）
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020053014254658.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
1⃣️ 调用beanPostProcessor的postProcessBeforeInitialization方法，前置增强

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530143157313.png)
2⃣️ 反射调用initMethod方法，初始化

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530142935925.png)
3⃣️ 调用beanPostProcessor的postProcessAfterInitialization方法后置增强
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020053015032874.png)

4⃣️ registerDisposableBeanIfNecessary注册销毁器DisposableBeanAdapter

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200530150344244.png)
### 12  最后一个方法finishRefresh完成容器刷新

最后把beanFactory的对于生命周期的注释放在下面
```java
 * <p>Bean factory implementations should support the standard bean lifecycle interfaces
 * as far as possible. The full set of initialization methods and their standard order is:
 * <ol>
 * <li>BeanNameAware's {@code setBeanName}
 * <li>BeanClassLoaderAware's {@code setBeanClassLoader}
 * <li>BeanFactoryAware's {@code setBeanFactory}
 * <li>EnvironmentAware's {@code setEnvironment}
 * <li>EmbeddedValueResolverAware's {@code setEmbeddedValueResolver}
 * <li>ResourceLoaderAware's {@code setResourceLoader}
 * (only applicable when running in an application context)
 * <li>ApplicationEventPublisherAware's {@code setApplicationEventPublisher}
 * (only applicable when running in an application context)
 * <li>MessageSourceAware's {@code setMessageSource}
 * (only applicable when running in an application context)
 * <li>ApplicationContextAware's {@code setApplicationContext}
 * (only applicable when running in an application context)
 * <li>ServletContextAware's {@code setServletContext}
 * (only applicable when running in a web application context)
 * <li>{@code postProcessBeforeInitialization} methods of BeanPostProcessors
 * <li>InitializingBean's {@code afterPropertiesSet}
 * <li>a custom init-method definition
 * <li>{@code postProcessAfterInitialization} methods of BeanPostProcessors
 * </ol>
 *
 * <p>On shutdown of a bean factory, the following lifecycle methods apply:
 * <ol>
 * <li>{@code postProcessBeforeDestruction} methods of DestructionAwareBeanPostProcessors
 * <li>DisposableBean's {@code destroy}
 * <li>a custom destroy-method definition
 * </ol>
```