<font size=4 >

##### 解决的前提
    1. 循环依赖的类型
        - 互相在构造函数中依赖对方
        - 一个在构造函数，另外一个在字段中引入（或者set方法）
        - 互相都在set方法中
    
    2. 可以解决的循环依赖
        - 以上三种情况中，"互相在构造函数中"的是无法解决的
            - 原理见下（实例化的时候就不能成功了）
        
        - 且符合以下三个条件
            1. 只针对scope单例
            2. 没有显式指明不需要解决循环依赖的对象
            3. 该对象没有被代理过
        
        
##### 如何解决
    - Spring实例化一个bean时,分为三步
        1. 实例化
            - createBeanInstance
            
        2. 填充属性
            - populate：populateBean
            
                
        3. 属性处理完后处理
            -  AfterPropertiesSet方法，或spring xml中指定的init方法
    
    - 通过深度优先的方式获取目标bean及其所依赖的bean
    
    - 即：
        1. 先实例化当前对象
            - 此时即使没有完成属性的注入，他也是一个半成品，可以被依赖
            - 基于引用传递，获取到对象的引用时，对象的field或者或属性可以延后设置
            
        2. 递归的获取属性Bean进行注入

###### 三级缓存

	/** Cache of singleton objects: bean name --> bean instance */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);
	    - 单例对象的cache
	    - "一级缓存"

	/** Cache of singleton factories: bean name --> ObjectFactory */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);
	    - 单例对象工厂的cache
	    - "三级缓存"

	/** Cache of early singleton objects: bean name --> bean instance */
	private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);
	    - 提前曝光的单例对象的cache
	    - "二级缓存"
	  
	    
###### 源码分析
    - AbstractBeanFactory
    
    1. 首先从singletonObjects（一级缓存）中尝试获取
        - 如果获取不到并且对象在创建中，则尝试从earlySingletonObjects(二级缓存)中获取
        
        - 具体
            1. doGetBean会调用getSingleton方法，尝试从singletonObjects中获取
                - 获取不到就标记此bean"创建中"
                
            2. doGetBean 会再次调用getSingleton的重载方法
                - 传入一个singletonFactory 对象
                - 并调用createBean
                
            3. createBean（子类的）会进入doCreateBean
                - doCreateBean一开始，就会进行实例化（实例化完没有设置属性）
                    - 并且判断是否可以提前暴露
                    - 可以的话通过addSingletonFactory提前暴露，其实就是在三级缓存，即从singletonFactories中设置
            
            4. 接着会逐个需要注入的属性进行处理，最后再执行初始化
    
    2. 循环依赖，会再次调用getBean，即第二次进来  
        1. 由于是创建中，因此允许尝试从earlySingletonObjects二级缓存获取
    
        2. 如果还是获取不到并且允许从singletonFactories通过getObject获取
            - 则通过singletonFactory.getObject()(三级缓存)获取
        
        3. 如果获取到了则将singletonObject放入到earlySingletonObjects
            - 也就是 将三级缓存提升到二级缓存中
            
    
    - A中有B属性，B中有A属性
        - 先new A，提前暴露，放进singletonFactories
        - 对A的属性进行注入，及populate
            - populate找B对象，会创建B
            - B也是创建后，暴露至singletonFactories。
            - B调用populate获取属性A
                - 此时A在singletonFactories，拿到，B完成
        - 拿到B，A也完成
        
        - earlySingletonObjects的作用
            - typeMatch类型匹配（待研究。。）
            
#####  为何需要三级缓存,而不是两级缓存?
    1. 将刚实例化好的bean放在singletonFactories的好处是可扩展
    
    2. 就在addSingletonFactory提前暴露时，传入的ObjectFactory的一个函数式接口
        - getEarlyBeanReference方法
        - ObjectFactory中只有一个getObject()方法，所以是调用getObject()时处理扩展
        
    3. 该方法会对SmartInstantiationAwareBeanPostProcessor这种后置处理器进行处理
        - 给用户（或spring内部）提供接口扩展的
        
    4. 放到early里面的已经是处理完成的
        



	   