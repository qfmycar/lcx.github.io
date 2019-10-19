### Mybatis源码分析  

###### @author 	Lin.cx		

###### @date		2019.09.28

```java
	mybatis源码可以看做两部分，分别对应两个问题
	1、mybatis是如何与Spring相结合的(不依赖Spring也可运行)?
	2、mybatis是如何运行自身SQL的(自身的逻辑结构)?
```

### 一、第一部分

#### 1、Mybatis是如何与Spring相结合的?

```java
	首先使用 @mapperScan这个注解，会注入一个 MapperScannerRegistrar类，这个类实现的 Spring的工厂后置处理器，将指定包下的 mapper包装成一个 BeanDefinition，并将 Class设置成 MapperFactoryBean，而MapperFactoryBean的 getObject()方法返回的是一个代理类。
	与此同时，SpringBoot启动自动装配时，自动加载了 MybatisAutoConfiguration，而这个配置类自动注入了sqlSessionFactory和 sqlSessionTemple，此时环境中有这两个对象
	那么接下来我们再看一下 MapperFactoryBean这边，在 Spring的 IOC容器中现在已经加载了Mapper类，name就是mapper的名字，类就是 MapperFactoryBean，可我们怎么使用 mapper中的方法呢？在MapperFactoryBean中维护了一个 Mapper接口，在包装成 BeanDefinition时已经设置了这个值，当Service需要XXXMapper依赖注入时，就会去 Spring容器中寻找，于是就会调用 MapperFactoryBean<XXXMapper>的 getObject()方法（不明白的可以去看Spring的 FactoryBean），在它这个方法里面会根据 sqlSession获取 Mapper
		return getSqlSession.getMapper(this.mapperInterface);
	参数就是 Mapper类型的接口，通过 JDK动态代理生成对象调用
```

##### 1.1抛出问题

```java
	是不是对Mybatis架构又多了一层了解呢？这还是远远不够的。根据上述抛出以下几个问题：
	1、MapperFactoryBean的 getSqlSession().getMapper(this.mapperInterface)方法，它的sqlSession是怎么来的？？？
	2、已经知道Spring环境中有sqlSession对象，那它是怎么复制给MapperFactoryBean的？是自动注入还是？
	3、getMapper(接口XXX)里面究竟是怎么返回代理对象的？
	带着这些问题，先继续往下看第二部分
```

### 二、第二部分

#### 1、Mybatis是如何运行自身SQL的？

```java
	getMapper(接口XXX)点进去是 getConfiguration().getMapper(type, this)，问题又多了一个，一看名字Configuration又是哪里来的？继续点进去看到 mapperRegistry.getMapper(type, sqlSession)；再点进去发现里面维护了一个 Map<Class<?>, MapperProxyFactory<?>>，所以 getMapper()最终就是从这个map里根据key获取value，代码返回了 mapperProxyFactory.newInstance(sqlSession)；这里的newInstance是反射的方法，但是Map里的 K-V是啥时候放进去的？
	发现MapperFactoryBean里有一个checkDaoConfig()方法，当中有一行代码							      configuration.addMapper(this.mapperInterface);
这个 checkDaoConfig()是怎么被调用的？查看 MapperScannerBean的父类 sqlSessionDaoSupport的父类 DaoSupport发现它实现了 InitializingBean，而在afterPropertiesSet()方法中调用了 checkDaoConfig()，恍然大悟！
	那么问题依旧存在，sqlSession和Configuration是从哪里来的？？？
	经过寻找，在 MybatisAutoConfiguration中返回了一个 SqlSessionFactory，而这个对象是由SqlSessionFactoryBean生产的，猜想在 SqlSessionFactoryBean的 getObject方法中有相关实例化，于是点进去看，发现调用了 afterPropertiesSet()；继续点入，调用了 buildSqlSessionFactory()，在这里发现了
		configuration = new Configuration(); 一行代码
既然 Configuration有了，那 sqlSession呢？
	我通过翻阅各个地方都没有发现 SqlSession的实例化，最终发现这么一行代码
		definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
给 beanDefinition设置 autoWireMode的时候，竟然是这行代码起了作用


	Spring在实例化Bean的时候会根据自动注入类型去调用方法
	if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
                mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

        // Add property values based on autowire by name if applicable.
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }

        // Add property values based on autowire by type if applicable.
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }
        pvs = newPvs;
    }

	前面的MapperFactoryBean的 BeanDefinition已经设置为 AUTOWIRE_BY_TYPE，所以会调用autowireByType()方法，该方法内部逻辑为获取当前bean的所有PropertyDescriptor，并过滤出包含有WriteMethod的PropertyNames。
	获取一个bean的PropertyDescriptor示例代码如下
	
    public void testPropertyDescripties() throw IntrospectionException{
    	BeanInfo beanInfo = Intorspector.getBeanInfo(IntrospectorTest.class);
    	PropertyDescriptor[] pds = beanInfo.getPropertyDescripties();
        pds.forEach(pd ->{
            if(pd.getName().equals("calss")){
				continue;
            }        
            System.out.println(pd.getName());
            System.out.println(pd.getReadMethod());
            System.out.println(pd.getWriteMethod());
            System.out.println("********");
        });
    }
```
