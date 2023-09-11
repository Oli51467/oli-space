#### 1. 为什么ConcurrentHashMap的key不允许为null

因为ConcurrentHashMap是并发安全的，为了避免在多线程下出现歧义。在多线程执行时，如果key允许为null，则调用的线程不知道取到的值为null，还是该key不存在返回为null。这会造成线程安全性问题，而该集合本身线程安全的，所以有了这样的设计。

#### 2. Redis和Mysql如何保证数据一致性

作用：数据缓冲。一份数据存储在两个位置。

1. 先更新数据库，再更新缓存。

2. 先删除缓存，再更新数据库（非原子操作，如果此时有其他线程访问，仍会出现数据不一致的问题）

   可以基于最终一致性，使用RocketMQ，将更新失败的请求写入MQ的事务消息再异步重试。要根据具体的业务场景来选择。

#### 3. Integer和int的区别

关键：从封装类的特性和功能角度回答

1. Integer是int的封装类，Integer需要new关键字创建对象。Integer存储在堆内存，int存储在栈空间。

2. 基本类型int和Integer封装类型混合使用时，Java会自动通过拆箱和装箱实现类型转换
3. Integer作为对象类型，封装了一些方法和属性，比如Integer.parseInt(String.class),可以利用这些方法操作数据。
4. Integer的默认值是null，int的默认值是0

Java是面向对象的语言，对象是基础操作单元，一些集合只能存储对象类型

#### 4. lock和synchronized的区别

1. 从功能的角度来看，lock和synchronized都是用来解决线程安全问题的一个工具。
2. 特性看，synchronized是Java中的同步关键字，lock是J.U.C包中提供的接口，这个接口有很多实现类，其中就包括ReentrantLock重入锁。
3. synchronized可以通过两种方式控制锁的力度，一种是修饰在方法层面，另一种是修饰在代码块，可以通过synchronized加锁对象的生命周期来控制锁的作用范围：如果锁对象是类对象或者是静态变量，那么这个锁就属于全局锁；如果锁对象是普通实例对象，那么锁的范围取决于这个实例的生命周期。
4. lock锁的力度是通过lock.lock和lock.unlock来控制的，中间的代码块能够保证线程安全，锁的作用域取决于lock实例的生命周期。
5. lock比synchronized的灵活性更高，可以自主决定什么时候加锁什么时候解锁。
6. lock提供了非阻塞竞争锁的方法：tryLock，通过返回true/false来告诉线程是否已经有其他线程正在使用锁。
7. synchronized的释放是被动的，被synchronized包含的代码块执行结束以后或者代码出现异常的时候锁才会被释放。
8. lock提供了公平锁和非公平锁的机制，公平锁是指线程在竞争锁资源的时候，如果已经有其他线程正在排队，那么当前竞争锁的进程是无法插队的。非公平锁是不管是否有其他线程在排队，都会尝试去获取锁。synchronized只提供了一种非公平锁的实现。
9. 性能方面，synchronized引入了偏向锁，轻量级锁，重量级锁实现性能优化。lock中使用自旋锁的方式实现性能优化。

#### 5. @Resource和@Autowired的区别

共同点：实现Bean的依赖注入

##### Autowired：

1. 是Spring里提供的注解，是通过类型实现依赖注入，有required属性，默认值为true。在应用启动的时候，如果IOC容器里不存在对应类型的Bean，启动时报错。
2. 如果存在多个相同类型的Bean，由于Autowired是根据类型注入，启动时也会报错，可以使用@Primary(优先使用的Bean)或@Qualifier(根据名字进行目标筛选找到装配目标的Bean)这两个注解来解决。

##### Resource:

1. JDK(JSR 250规范)提供的注解，可以支持name(名字)和type(类型)两种注入方式。如果都没有配置，会先根据定义的属性名字去匹配