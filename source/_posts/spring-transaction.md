---
title: Spring事务详解
date: 2018-05-11 16:01:48
toc: true
comments: true
tags:
- spring
- spring事务
---

# 概念

对于事务(Transaction)的概念，网上有各种版本，大同小异，

事务就是是由一系列对系统中数据进行读写的操作组成的一个程序执行单元，狭义上的事务特指数据库事务。

事务是一系列的动作，它们综合在一起才是一个完整的工作单元，这些动作必须全部完成，如果有一个失败的话，那么事务就会回滚到最开始的状态，仿佛什么都没发生过一样。

在企业级应用程序开发中，事务管理必不可少的技术，用来确保数据的完整性和一致性。

比如你去ATM机取1000块钱，大体有两个步骤：首先输入密码金额，银行卡扣掉1000元钱；然后ATM出1000元钱。这两个步骤必须是要么都执行要么都不执行。如果银行卡扣除了1000块但是ATM出钱失败的话，你将会损失1000元；如果银行卡扣钱失败但是ATM却出了1000块，那么银行将损失1000元。所以，如果一个步骤成功另一个步骤失败对双方都不是好事，如果不管哪一个步骤失败了以后，整个取钱过程都能回滚，也就是完全取消所有操作的话，这对双方都是极好的。

<!-- more -->

# 事务的特性

大名鼎鼎的ACID

1. 原子性（Atomicity），事务必须是一个原子的操作序列单元，一次事务只允许存在两种状态，全部成功或全部失败，任何一个操作失败都将导致整个事务失败
2. 一致性（Consistency），事务的执行不能破坏系统数据的完整性和一致性，如果未完成的事务对系统数据的修改有一部分已经写入物理数据库，这时系统数据就处于不一致状态
3. 隔离性（Isolation），在并发环境中，不同的事务操作相同的数据时，虚相互隔离不能相互干扰
4. 持久性（Durability），事务一旦提交，对系统数据的变更就应该是永久的，必须被永久保存下来，即使服务器宕机了，只要数据库能够重新启动，就一定能够恢复到事务成功结束时的状态

# 事务并发处理问题

如果没有锁定且多个用户同时访问一个数据库，则当他们的事务同时使用相同的数据时可能会发生问题。由于并发操作带来的数据不一致性包括：丢失数据修改、读”脏”数据（脏读）、不可重复读、产生幽灵数据：

假设数据库中有如下一张表：

![测试结果](/image/spring-transcation/1.png)

## 第一类丢失更新(lost update)

### **回滚丢失**

在完全未隔离事务的情况下，两个事物更新同一条数据资源，某一事物异常终止，回滚造成第一个完成的更新也同时丢失。

![image](/image/spring-transcation/2.png)

在T1时刻开启了事务1，T2时刻开启了事务2，

在T3时刻事务1从数据库中取出了id=”402881e535194b8f0135194b91310001”的数据，

T4时刻事务2取出了同一条数据，

T5时刻事务1将age字段值更新为30，

T6时刻事务2更新age为35并提交了数据，

但是T7事务1回滚了事务age最后的值依然为20，事务2的更新丢失了，

这种情况就叫做”第一类丢失更新(lost update)”。

## 脏读(dirty read)

### **事务没提交，提前读取**

如果第二个事务查询到第一个事务还未提交的更新数据，形成脏读

![image](/image/spring-transcation/3.png)

在T1时刻开启了事务1，T2时刻开启了事务2，

在T3时刻事务1从数据库中取出了id=”402881e535194b8f0135194b91310001”的数据，

在T5时刻事务1将age的值更新为30，但是事务还未提交，

T6时刻事务2读取同一条记录，获得age的值为30，但是事务1还未提交，

若在T7时刻事务1回滚了事务2的数据就是错误的数据(脏数据)，

这种情况叫做” 脏读(dirty read)”。

## 虚读(phantom read)

一个事务执行两次查询，第二次结果集包含第一次中没有或者某些行已被删除，造成两次结果不一致，只是另一个事务在这两次查询中间插入或者删除了数据造成的

![image](/image/spring-transcation/phantomread.png)

在T1时刻开启了事务1，T2时刻开启了事务2，

T3时刻事务1从数据库中查询所有记录，记录总共有一条，

T4时刻事务2向数据库中插入一条记录，T6时刻事务2提交事务。

T7事务1再次查询数据数据时，记录变成两条了。

这种情况是”虚读(phantom read)”。

### 不可重复读(unrepeated read)

一个事务两次读取同一行数据，结果得到不同状态结果，如中间正好另一个事务更新了该数据，两次结果相异，不可信任
![image](/image/spring-transcation/unrepeatedread.png)

在T1时刻开启了事务1，T2时刻开启了事务2，

在T3时刻事务1从数据库中取出了id=”402881e535194b8f0135194b91310001”的数据，此时age=20，

T4时刻事务2查询同一条数据，

T5事务2更新数据age=30，T6时刻事务2提交事务，

T7事务1查询同一条数据，发现数据与第一次不一致。

这种情况就是”不可重复读(unrepeated read)”

### 第二类丢失更新(second lost updates)

**覆盖丢失**

不可重复读的特殊情况，如果两个事务都读取同一行，然后两个都进行写操作，并提交，第一个事务所做的改变就会丢失。

![image](/image/spring-transcation/secondlostupdates.png)

在T1时刻开启了事务1，T2时刻开启了事务2，

T3时刻事务1更新数据age=25，

T5时刻事务2更新数据age=30，

T6时刻提交事务，

T7时刻事务2提交事务，把事务1的更新覆盖了。

这种情况就是”第二类丢失更新(second lost updates)”。

## 并发问题总结

不可重复读的重点是修改 :

同样的条件 , 你读取过的数据 , 再次读取出来发现值不一样了

幻读的重点在于新增或者删除

同样的条件 , 第 1 次和第 2 次读出来的记录数不一样

第一类更新丢失(回滚丢失)

第二类更新丢失(覆盖丢失)

## 隔离级别

解决并发问题的途径是什么?答案是：采取有效的隔离机制。怎样实现事务的隔离呢？隔离机制的实现必须使用锁

一般在编程的时候只需要设置隔离等级

数据库系统提供四种事务隔离级别：

​	1.未提交读（READ UNCOMMITTED ）

最低隔离级别，一个事务能读取到别的事务未提交的更新数据，很不安全，可能出现丢失更新、脏读、不可重复读、幻读；

​	2.提交读（READ COMMITTED）

一个事务能读取到别的事务提交的更新数据，不能看到未提交的更新数据，不会出现丢失更新、脏读，但可能出现不可重复读、幻读；

​	3.可重复读（REPEATABLE READ）

保证同一事务中先后执行的多次查询将返回同一结果，不受其他事务影响，不可能出现丢失更新、脏读、不可重复读，但可能出现幻读；

​	4.序列化（SERIALIZABLE）

最高隔离级别，不允许事务并发执行，而必须串行化执行，最安全，不可能出现更新、脏读、不可重复读、幻读，但是效率最低。

![image](/image/spring-transcation/isolation.jpg)

隔离级别越高，数据库事务并发执行性能越差，能处理的操作越少。
所以一般地，推荐使用REPEATABLE READ级别保证数据的读一致性。
对于幻读的问题，可以通过加锁来防止

![image](/image/spring-transcation/isolation-phr.png)

MySQL支持这四种事务等级，默认事务隔离级别是REPEATABLE READ。

Oracle数据库支持READ COMMITTED 和 SERIALIZABLE这两种事务隔离级别，
所以Oracle数据库不支持脏读

Oracle数据库默认的事务隔离级别是READ COMMITTED

不可重复读和幻读的区别是，不可重复读对应的表的操作是更改(UPDATE)，而幻读对应的表的操作是插入(INSERT)，两种的应对策略不一样。对于不可重复读，只需要采用行级锁防止该记录被更新即可，而对于幻读必须加个表级锁，防止在表中插入数据

### 锁

#### 乐观锁(Optimistic Lock)和悲观锁(Pessimistic Lock)

最重要的分类就是乐观锁(Optimistic Lock)和悲观锁(Pessimistic Lock)，这实际上是两种锁策略

**乐观锁**，顾名思义就是非常乐观，非常相信真善美，每次去读数据都认为其它事务没有在写数据，所以就不上锁，快乐的读取数据，而只在提交数据的时候判断其它事务是否搞过这个数据了，如果搞过就rollback。乐观锁相当于一种检测冲突的手段，可通过为记录添加版本或添加时间戳来实现。

**悲观锁**，对其它事务抱有保守的态度，每次去读数据都认为其它事务想要作祟，所以每次读数据的时候都会上锁，直到取出数据。悲观锁大多数情况下依靠数据库的锁机制实现，以保证操作最大程度的独占性，但随之而来的是各种开销。悲观锁相当于一种避免冲突的手段。

*选择标准*：如果并发量不大，或数据冲突的后果不严重，则可以使用乐观锁；而如果并发量大或数据冲突后果比较严重（对用户不友好），那么就使用悲观锁。

#### 分共享锁（S锁，Shared Lock）和排他锁（X锁，Exclusive Lock）

从读写角度，分共享锁（S锁，Shared Lock）和排他锁（X锁，Exclusive Lock），也叫读锁（Read Lock）和写锁（Write Lock）。
理解：

持有S锁的事务只读不可写。

如果事务A对数据D加上S锁后，其它事务只能对D加上S锁而不能加X锁。

持有X锁的事务可读可写。

如果事务A对数据D加上X锁后，其它事务不能再对D加锁，直到A对D的锁解除。

#### 表级锁（Table Lock）和行级锁（Row Lock）

从锁的粒度角度，主要分为表级锁（Table Lock）和行级锁（Row Lock）。

表级锁将整个表加锁，性能开销最小

用户可以同时进行读操作。当一个用户对表进行写操作时，用户可以获得一个写锁，写锁禁止其他的用户读写操作。写锁比读锁的优先级更高，即使有读操作已排在队列中，一个被申请的写锁仍可以排在所队列的前列。

行级锁仅对指定的记录进行加锁

这样其它进程可以对同一个表中的其它记录进行读写操作。行级锁粒度最小，开销大，能够支持高并发，可能会出现死锁。

MySQL的MyISAM引擎使用表级锁，而InnoDB支持表级锁和行级锁，默认是行级锁。
还有BDB引擎使用页级锁，即一次锁定一组记录，并发性介于行级锁和表级锁之间。

#### 三级锁协议

三级加锁协议是为了保证正确的事务并发操作，事务在读、写数据库对象是需要遵循的加锁规则。

一级封锁协议：事务T在修改数据R之前必须对它加X锁，直到事务结束方可释放。而若事务T只是读数据，不进行修改，则不需加锁，因此一级加锁协议下可能会出现脏读和不可重复读。

二级加锁协议：在一级加锁协议的基础上，加上这样一条规则——事务T在读取数据R之前必须对它加S锁，直到读取完毕以后释放。二级加锁协议下可能会出现不可重复读。

三级加锁协议：在一级加锁协议的基础上，加上这样一条规则——事务T在读取数据R之前必须对它加S锁，直到事务结束方可释放。三级加锁协议避免了脏读和不可重复读的问题

# Spring事务

Spring事务管理的实现有许多细节，如果对整个接口框架有个大体了解会非常有利于我们理解事务

![image](/image/spring-transcation/transaction.jpg)

![image](/image/spring-transcation/transaction_1.jpg)

## Spring事务管理器

Spring事务管理涉及的接口的联系如下：
![image](/image/spring-transcation/spring-interface.png)

Spring并不直接管理事务，而是提供了多种事务管理器，他们将事务管理的职责委托给Hibernate或者JTA等持久化机制所提供的相关平台框架的事务来实现。

Spring事务管理器的接口是org.springframework.transaction.PlatformTransactionManager，
通过这个接口，Spring为各个平台如JDBC、Hibernate等都提供了对应的事务管理器，

但是具体的实现就是各个平台自己的事情了

```java
/**
 * This is the central interface in Spring's transaction infrastructure.
 * Applications can use this directly, but it is not primarily meant as API:
 * Typically, applications will work with either TransactionTemplate or
 * declarative transaction demarcation through AOP.
 */
public interface PlatformTransactionManager {
    TransactionStatus getTransaction(TransactionDefinition definition)
        throws TransactionException;
    void commit(TransactionStatus status) throws TransactionException;
    void rollback(TransactionStatus status) throws TransactionException;
}
```

标准的jdbc处理事务代码

```java
Connection conn = DataSourceUtils.getConnection();
 //开启事务
conn.setAutoCommit(false);
try {
    Object retVal = callback.doInConnection(conn);
    conn.commit(); //提交事务
    return retVal;
}catch (Exception e) {
    conn.rollback();//回滚事务
    throw e;
}finally {
    conn.close();
}
```

spring对应的TranstactionTemplate处理

```java
public class TransactionTemplate extends DefaultTransactionDefinition
		implements TransactionOperations, InitializingBean {
		
    @Override
	public <T> T execute(TransactionCallback<T> action) throws TransactionException {
		if (this.transactionManager instanceof CallbackPreferringPlatformTransactionManager) {
			return ((CallbackPreferringPlatformTransactionManager) this.transactionManager).execute(this, action);
		}
		else {
			TransactionStatus status = this.transactionManager.getTransaction(this);
			T result;
			try {
				result = action.doInTransaction(status);
			}
			catch (RuntimeException ex) {
				// Transactional code threw application exception -> rollback
				rollbackOnException(status, ex);
				throw ex;
			}
			catch (Error err) {
				// Transactional code threw error -> rollback
				rollbackOnException(status, err);
				throw err;
			}
			catch (Exception ex) {
				// Transactional code threw unexpected exception -> rollback
				rollbackOnException(status, ex);
				throw new UndeclaredThrowableException(ex, "TransactionCallback threw undeclared checked exception");
			}
			this.transactionManager.commit(status);
			return result;
		}
	}
	
}
```

具体的事务管理机制对Spring来说是透明的，它并不关心那些，那些是对应各个平台需要关心的，所以Spring事务管理的一个优点就是为不同的事务API提供一致的编程模型，如JTA、JDBC、Hibernate、JPA。下面分别介绍各个平台框架实现事务管理的机制。

![image](/image/spring-transcation/spring-txmanager.jpg)

### JDBC事务

如果应用程序中直接使用JDBC来进行持久化，DataSourceTransactionManager会为你处理事务边界。为了使用DataSourceTransactionManager，你需要使用如下的XML将其装配到应用程序的上下文定义中：

```java
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource" />
</bean>
```

实际上，DataSourceTransactionManager是通过调用java.sql.Connection来管理事务，而后者是通过DataSource获取到的。通过调用连接的commit()方法来提交事务，同样，事务失败则通过调用rollback()方法进行回滚。

### Hibernate事务

如果应用程序的持久化是通过Hibernate实习的，那么你需要使用HibernateTransactionManager。对于Hibernate3，需要在Spring上下文定义中添加如下的声明：

```java
<bean id="transactionManager" class="org.springframework.orm.hibernate3.HibernateTransactionManager">
    <property name="sessionFactory" ref="sessionFactory" />
</bean>
```

sessionFactory属性需要装配一个Hibernate的session工厂，HibernateTransactionManager的实现细节是它将事务管理的职责委托给org.hibernate.Transaction对象，而后者是从Hibernate Session中获取到的。当事务成功完成时，HibernateTransactionManager将会调用Transaction对象的commit()方法，反之，将会调用rollback()方法。

## 事务属性

事务管理器接口PlatformTransactionManager通过getTransaction(TransactionDefinition definition)方法来得到事务，这个方法里面的参数是TransactionDefinition类，这个类就定义了一些基本的事务属性。

```java
public interface TransactionDefinition {
    int getPropagationBehavior(); // 返回事务的传播行为
    int getIsolationLevel(); // 返回事务的隔离级别，事务管理器根据它来控制另外一个事务可以看到本事务内的哪些数据
    int getTimeout();  // 返回事务必须在多少秒内完成
    boolean isReadOnly(); // 事务是否只读，事务管理器能够根据这个返回值进行优化，确保事务是只读的
}
```

那么什么是事务属性呢？事务属性可以理解成事务的一些基本配置，描述了事务策略如何应用到方法上。事务属性包含了5个方面

![image](/image/spring-transcation/spring-attribute.png)

### 传播行为

事务的第一个方面是传播行为（propagation behavior）。
当事务方法被另一个事务方法调用时，必须指定事务应该如何传播。

为什么需要定义传播？

> 在我们用SSH开发项目的时候，我们一般都是将事务设置在Service层 那么当我们调用Service层的一个方法的时候它能够保证我们的这个方法中执行的所有的对数据库的更新操作保持在一个事务中，在事务层里面调用的这些方法要么全部成功，要么全部失败。那么事务的传播特性也是从这里说起的。
> 如果你在你的Service层的这个方法中，除了调用了Dao层的方法之外，还调用了本类的其他的Service方法，那么在调用其他的Service方法的时候，这个事务是怎么规定的呢，我必须保证我在我方法里掉用的这个方法与我本身的方法处在同一个事务中，否则如果保证事物的一致性。事务的传播特性就是解决这个问题的，“事务是会传播的”在Spring中有针对传播特性的多种配置我们大多数情况下只用其中的一种:PROPGATION_REQUIRED：这个配置项的意思是说当我调用service层的方法的时候开启一个事务(具体调用那一层的方法开始创建事务，要看你的aop的配置),那么在调用这个service层里面的其他的方法的时候,如果当前方法产生了事务就用当前方法产生的事务，否则就创建一个新的事务。这个工作使由Spring来帮助我们完成的。
> 以前没有Spring帮助我们完成事务的时候我们必须自己手动的控制事务，例如当我们项目中仅仅使用hibernate，而没有集成进spring的时候，我们在一个service层中调用其他的业务逻辑方法，为了保证事物必须也要把当前的hibernate session传递到下一个方法中，或者采用ThreadLocal的方法，将session传递给下一个方法，其实都是一个目的。现在这个工作由spring来帮助我们完成，就可以让我们更加的专注于我们的业务逻辑。而不用去关心事务的问题。

Spring定义了七种传播行为：

- PROPAGATION_REQUIRED

如果当前没有事务，就新建一个事务，如果已经存在一个事务中，加入到这个事务中。这是最常见的选择。

* PROPAGATION_SUPPORTS

支持当前事务，如果当前没有事务，就以非事务方式执行。

* PROPAGATION_MANDATORY

使用当前的事务，如果当前没有事务，就抛出异常。

* PROPAGATION_REQUIRES_NEW

新建事务，如果当前存在事务，把当前事务挂起。

* PROPAGATION_NOT_SUPPORTED

以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。

* PROPAGATION_NEVER

以非事务方式执行，如果当前存在事务，则抛出异常。

* PROPAGATION_NESTED

如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作。

#### 传播行为详细

通过实例尝试一下各个传播属性

```java
ServiceA {
       
     void methodA() {
         ServiceB.methodB();
     }
}
  
ServiceB {
       
     void methodB() {
     }
}
```

##### PROPAGATION_REQUIRED

如果存在一个事务，则支持当前事务。如果没有事务则开启一个新的事务

```java
//事务属性 PROPAGATION_REQUIRED
methodA{
    ……
    methodB();
    ……
}
//事务属性 PROPAGATION_REQUIRED
methodB{
   ……
}
```

单独调用methodB方法：

```java
main{ 
    metodB(); 
}
```

相当于

```java
Main{ 
    Connection con=null; 
    try{ 
        con = getConnection(); 
        con.setAutoCommit(false); 

        //方法调用
        methodB(); 

        //提交事务
        con.commit(); 
    } Catch(RuntimeException ex) { 
        //回滚事务
        con.rollback();   
    } finally { 
        //释放资源
        closeCon(); 
    } 
}
```

Spring保证在methodB方法中所有的调用都获得到一个相同的连接。在调用methodB时，没有一个存在的事务，所以获得一个新的连接，开启了一个新的事务。

**只是在ServiceB.methodB内的任何地方出现异常，ServiceB.methodB将会被回滚，不会引起ServiceA.methodA的回滚**

单独调用MethodA时，在MethodA内又会调用MethodB.

执行效果相当于：

```java
main{ 
    Connection con = null; 
    try{ 
        con = getConnection(); 
        methodA(); 
        con.commit(); 
    } catch(RuntimeException ex) { 
        con.rollback(); 
    } finally {    
        closeCon(); 
    }  
}
```

调用MethodA时，环境中没有事务，所以开启一个新的事务.

当在MethodA中调用MethodB时，环境中已经有了一个事务，所以methodB就加入当前事务

**ServiceA.methodA或者ServiceB.methodB无论哪个发生异常methodA和methodB作为一个整体都将一起回滚**

##### PROPAGATION_SUPPORTS

如果存在一个事务，支持当前事务。如果没有事务，则非事务的执行。但是对于事务同步的事务管理器，PROPAGATION_SUPPORTS与不使用事务有少许不同

```java
//事务属性 PROPAGATION_REQUIRED
methodA(){
  methodB();
}

//事务属性 PROPAGATION_SUPPORTS
methodB(){
  ……
}
```

单纯的调用methodB时，methodB方法是非事务的执行的。当调用methdA时,methodB则加入了methodA的事务中,事务地执行

##### PROPAGATION_MANDATORY

如果已经存在一个事务，支持当前事务。如果没有一个活动的事务，则抛出异常

```java
//事务属性 PROPAGATION_REQUIRED
methodA(){
    methodB();
}

//事务属性 PROPAGATION_MANDATORY
    methodB(){
    ……
}
```

当单独调用methodB时，因为当前没有一个活动的事务，则会抛出异常throw new IllegalTransactionStateException(“Transaction propagation ‘mandatory’ but no existing transaction found”);

当调用methodA时，methodB则加入到methodA的事务中，事务地执行

##### PROPAGATION_REQUIRES_NEW

总是开启一个新的事务。如果一个事务已经存在，则将这个存在的事务挂起。

```java
//事务属性 PROPAGATION_REQUIRED
methodA(){
    doSomeThingA();
    methodB();
    doSomeThingB();
}

//事务属性 PROPAGATION_REQUIRES_NEW
methodB(){
    ……
}
```

调用A方法：

```java
main(){
    methodA();
}
```

相当于

```java
main(){
    TransactionManager tm = null;
    try{
        //获得一个JTA事务管理器
        tm = getTransactionManager();
        tm.begin();//开启一个新的事务
        Transaction ts1 = tm.getTransaction();
        doSomeThing();
        tm.suspend();//挂起当前事务
        try{
            tm.begin();//重新开启第二个事务
            Transaction ts2 = tm.getTransaction();
            methodB();
            ts2.commit();//提交第二个事务
        } Catch(RunTimeException ex) {
            ts2.rollback();//回滚第二个事务
        } finally {
            //释放资源
        }
        //methodB执行完后，恢复第一个事务
        tm.resume(ts1);
        doSomeThingB();
        ts1.commit();//提交第一个事务
    } catch(RunTimeException ex) {
        ts1.rollback();//回滚第一个事务
    } finally {
        //释放资源
    }
}
```

在这里，我把ts1称为外层事务，ts2称为内层事务。从上面的代码可以看出，ts2与ts1是两个独立的事务，互不相干。

Ts2是否成功并不依赖于ts1

如果methodA方法在调用methodB方法后的doSomeThingB方法失败了，而methodB方法所做的结果依然被提交。

而除了 methodB之外的其它代码导致的结果却被回滚了

##### PROPAGATION_NOT_SUPPORTED

以非事务方式执行操作，如果当前存在事务，就把当前事务挂起,

```java
//事务属性 PROPAGATION_REQUIRED
methodA(){
    doSomeThingA();
    methodB();
    doSomeThingB();
}

//事务属性 PROPAGATION_NOT_SUPPORTED
methodB(){
    ……
}
```

当前不支持事务。比如ServiceA.methodA的事务级别是PROPAGATION_REQUIRED ，
而ServiceB.methodB的事务级别是PROPAGATION_NOT_SUPPORTED ，
那么当执行到ServiceB.methodB时，ServiceA.methodA的事务挂起，而他以非事务的状态运行完，再继续ServiceA.methodA的事务

##### PROPAGATION_NEVER

不能在事务中运行。假设ServiceA.methodA的事务级别是PROPAGATION_REQUIRED，
而ServiceB.methodB的事务级别是PROPAGATION_NEVER ，
那么ServiceB.methodB就要抛出异常了。

##### PROPAGATION_NESTED

开始一个 “嵌套的” 事务, 它是已经存在事务的一个真正的子事务. 潜套事务开始执行时, 它将取得一个 savepoint. 如果这个嵌套事务失败, 我们将回滚到此 savepoint. 潜套事务是外部事务的一部分, 只有外部事务结束后它才会被提交.

比如我们设计ServiceA.methodA的事务级别为PROPAGATION_REQUIRED，ServiceB.methodB的事务级别为PROPAGATION_NESTED，那么当执行到ServiceB.methodB的时候，ServiceA.methodA所在的事务就会挂起，ServiceB.methodB会起一个新的子事务并设置savepoint，等待ServiceB.methodB的事务完成以后，他才继续执行

因为ServiceB.methodB是外部事务的子事务，那么

1. 如果ServiceB.methodB已经提交，那么ServiceA.methodA失败回滚，ServiceB.methodB也将回滚。
2. 如果ServiceB.methodB失败回滚，如果他抛出的异常被ServiceA.methodA的try..catch捕获并处理，ServiceA.methodA事务仍然可能提交；如果他抛出的异常未被ServiceA.methodA捕获处理，ServiceA.methodA事务将回滚。

理解Nested的关键是savepoint。

**与PROPAGATION_REQUIRES_NEW的区别**：

1. RequiresNew每次都创建新的独立的物理事务，而Nested只有一个物理事务；
2. Nested嵌套事务回滚或提交不会导致外部事务回滚或提交，但外部事务回滚将导致嵌套事务回滚，而 RequiresNew由于都是全新的事务，所以之间是无关联的；
3. Nested使用JDBC 3的保存点实现，即如果使用低版本驱动将导致不支持嵌套事务。
   使用嵌套事务，必须确保具体事务管理器实现的nestedTransactionAllowed属性为true，否则不支持嵌套事务，如DataSourceTransactionManager默认支持，而HibernateTransactionManager默认不支持，需要我们来开启。

在 spring 中使用 PROPAGATION_NESTED的前提：

1. 我们要设置 transactionManager 的 nestedTransactionAllowed 属性为 true, 注意, 此属性默认为 false!!!
2. java.sql.Savepoint 必须存在, 即 jdk 版本要 1.4+
3. Connection.getMetaData().supportsSavepoints() 必须为 true, 即 jdbc drive 必须支持 JDBC 3.0

![image](/image/spring-transcation/propagation.png)

### 隔离规则

用来解决并发事务时出现的问题，其使用TransactionDefinition中的静态变量来指定

1. ISOLATION_DEFAULT 使用后端数据库默认的隔离级别
2. ISOLATION_READ_UNCOMMITTED 最低的隔离级别，允许读取尚未提交的数据变更，可能会导致脏读、幻读或不可重复读
3. ISOLATION_READ_COMMITTED 允许读取并发事务已经提交的数据，可以阻止脏读，但是幻读或不可重复读仍有可能发生
4. ISOLATION_REPEATABLE_READ 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，可以阻止脏读和不可重复读，但幻读仍有可能发生
5. ISOLATION_SERIALIZABLE 最高的隔离级别，完全服从ACID的隔离级别，确保阻止脏读、不可重复读以及幻读，也是最慢的事务隔离级别，因为它通常是通过完全锁定事务相关的数据库表来实现的

可以使用DefaultTransactionDefinition类的setIsolationLevel(TransactionDefinition. ISOLATION_READ_COMMITTED)来指定隔离级别，其中此处表示隔离级别为提交读

也可以使用或setIsolationLevelName(“ISOLATION_READ_COMMITTED”)方式指定，其中参数就是隔离级别静态变量的名字，但不推荐这种方式

### 事务只读

将事务标识为只读，只读事务不修改任何数据；

对于JDBC只是简单的将连接设置为只读模式，对于更新将抛出异常；

对于一些其他ORM框架有一些优化作用，如在Hibernate中，Spring事务管理器将执行“session.setFlushMode(FlushMode.MANUAL)”
即指定Hibernate会话在只读事务模式下不用尝试检测和同步持久对象的状态的更新。

如果使用设置具体事务管理的validateExistingTransaction属性为true（默认false），将确保整个事务传播链都是只读或都不是只读
![image](/image/spring-transcation/readonly.JPG)

第二个addressService.save()不能设置成false

对于错误的事务只读设置将抛出IllegalTransactionStateException异常，并伴随“Participating transaction with definition [……] is not marked as read-only……”信息，表示参与的事务只读属性设置错误

### 事务超时

设置事务的超时时间，单位为秒，默认为-1表示使用底层事务的超时时间

使用如setTimeout(100)来设置超时时间，如果事务超时将抛出org.springframework.transaction.TransactionTimedOutException异常并将当前事务标记为应该回滚，即超时后事务被自动回滚

可以使用具体事务管理器实现的defaultTimeout属性设置默认的事务超时时间，如DataSourceTransactionManager. setDefaultTimeout(10)

### 回滚规则

spring事务管理器会捕捉任何未处理的异常，然后依据规则决定是否回滚抛出异常的事务

默认配置下，Spring只有在抛出的异常为运行时unchecked异常时才回滚该事务，也就是抛出的异常为RuntimeException的子类(Errors也会导致事务回滚)，而抛出checked异常则不会导致事务回滚。可以明确的配置在抛出那些异常时回滚事务，包括checked异常。也可以明确定义那些异常抛出时不回滚事务

**如何改变默认规则**：

1. 让checked例外也回滚：在整个方法前加上 @Transactional(rollbackFor=Exception.class)
2. 让unchecked例外不回滚： @Transactional(notRollbackFor=RunTimeException.class)
3. 不需要事务管理的(只查询的)方法：@Transactional(propagation=Propagation.NOT_SUPPORTED)

## 事务状态

上面讲到的调用PlatformTransactionManager接口的getTransaction()的方法得到的是TransactionStatus接口的一个实现，这个接口的内容如下：

```java
public interface TransactionStatus{
    boolean isNewTransaction(); // 是否是新的事物
    boolean hasSavepoint(); // 是否有恢复点
    void setRollbackOnly();  // 设置为只回滚
    boolean isRollbackOnly(); // 是否为只回滚
    boolean isCompleted; // 是否已完成
}
```

可以发现这个接口描述的是一些处理事务提供简单的控制事务执行和查询事务状态的方法，在回滚或提交的时候需要应用对应的事务状态

## 编程式和声明式事务

Spring提供了对编程式事务和声明式事务的支持，编程式事务允许用户在代码中精确定义事务的边界

而声明式事务（基于AOP）有助于用户将操作与事务规则进行解耦。

简单地说，编程式事务侵入到了业务代码里面，但是提供了更加详细的事务管理；而声明式事务由于基于AOP，所以既能起到事务管理的作用，又可以不影响业务代码的具体实现。

### 编程式

Spring提供两种方式的编程式事务管理，分别是：使用TransactionTemplate和直接使用PlatformTransactionManager

#### 使用TransactionTemplate

采用TransactionTemplate和采用其他Spring模板，如JdbcTempalte和HibernateTemplate是一样的方法。它使用回调方法，把应用程序从处理取得和释放资源中解脱出来。如同其他模板，TransactionTemplate是线程安全的。代码片段：

```java
TransactionTemplate tt = new TransactionTemplate(); // 新建一个TransactionTemplate
Object result = tt.execute(
    new TransactionCallback(){  
        public Object doTransaction(TransactionStatus status){  
            updateOperation();  
            return resultOfUpdateOperation();  
        }  
}); // 执行execute方法进行事务管理
```

使用TransactionCallback()可以返回一个值。如果使用TransactionCallbackWithoutResult则没有返回值

#### 使用PlatformTransactionManager

示例代码如下：

```java
DataSourceTransactionManager dataSourceTransactionManager = new DataSourceTransactionManager();
//定义一个某个框架平台的TransactionManager，如JDBC、Hibernate
dataSourceTransactionManager.setDataSource(this.getJdbcTemplate().getDataSource()); // 设置数据源
    DefaultTransactionDefinition transDef = new DefaultTransactionDefinition(); // 定义事务属性
    transDef.setPropagationBehavior(DefaultTransactionDefinition.PROPAGATION_REQUIRED); // 设置传播行为属性
    TransactionStatus status = dataSourceTransactionManager.getTransaction(transDef); // 获得事务状态
    try {
        // 数据库操作
        dataSourceTransactionManager.commit(status);// 提交
    } catch (Exception e) {
        dataSourceTransactionManager.rollback(status);// 回滚
    }
```

### 声明式

有几种实现方式，不一一罗列了

#### 使用tx拦截器

```java
<!-- 定义事务管理器（声明式的事务） --> 
    <bean id="transactionManager"
        class="org.springframework.orm.hibernate3.HibernateTransactionManager">
        <property name="sessionFactory" ref="sessionFactory" />
    </bean>

    <tx:advice id="txAdvice" transaction-manager="transactionManager">
        <tx:attributes>
            <tx:method name="*" propagation="REQUIRED" />
        </tx:attributes>
    </tx:advice>

    <aop:config>
        <aop:pointcut id="interceptorPointCuts"
            expression="execution(* com.bluesky.spring.dao.*.*(..))" />
        <aop:advisor advice-ref="txAdvice"
            pointcut-ref="interceptorPointCuts" />       
    </aop:config>
```

#### 全注解

```java
<tx:annotation-driven transaction-manager="transactionManager" />
    <bean id="transactionManager"
          class="org.springframework.jdbc.datasource.DataSourceTransactionManager ">
        <property name="dataSource">
            <ref bean="basicDataSource" />
        </property>
</bean>
```

## Spring源码片段

在《BeanPostProcessor学习》中提到了AOP的实现方式，声明式事务实现是基于AOP

首先得解析xml配置，TxNamespaceHandler

```java
@Override
	public void init() {
		registerBeanDefinitionParser("advice", new TxAdviceBeanDefinitionParser());
		registerBeanDefinitionParser("annotation-driven", new AnnotationDrivenBeanDefinitionParser());
		registerBeanDefinitionParser("jta-transaction-manager", new JtaTransactionManagerBeanDefinitionParser());
	}
```

主要是TransactionInterceptor类

```java
public Object invoke(final MethodInvocation invocation) throws Throwable {
		// Work out the target class: may be {@code null}.
		// The TransactionAttributeSource should be passed the target class
		// as well as the method, which may be from an interface.
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

		// Adapt to TransactionAspectSupport's invokeWithinTransaction...
		//主要逻辑在父类
		return invokeWithinTransaction(invocation.getMethod(), targetClass, new InvocationCallback() {
			@Override
			public Object proceedWithInvocation() throws Throwable {
				return invocation.proceed();
			}
		});
	}
```

核心逻辑，还得看父类TransactionAspectSupport#invokeWithinTransaction

逻辑主干很清晰

```java
if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
			// 判断创建Transaction
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
				//执行业务逻辑
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
				// 出现异常，回滚
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				//清除当前事务状态
				cleanupTransactionInfo(txInfo);
			}
			//提交事务
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}
```

### 创建事务createTransactionIfNecessary

主要逻辑在PlatformTransactionManager#getTransaction()

```java
public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
		//得到各个不同数据源的事务对象，spring尽然没有把transaction对象抽象出来，很是奇怪
		Object transaction = doGetTransaction();

		// Cache debug flag to avoid repeated checks.
		boolean debugEnabled = logger.isDebugEnabled();

		if (definition == null) {
			// Use defaults if no transaction definition given.
			definition = new DefaultTransactionDefinition();
		}

		//此事务是否已经存在
		if (isExistingTransaction(transaction)) {
			// Existing transaction found -> check propagation behavior to find out how to behave.
			return handleExistingTransaction(definition, transaction, debugEnabled);
		}

		// Check definition settings for new transaction.
		if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
			throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
		}

		// No existing transaction found -> check propagation behavior to find out how to proceed.
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
			throw new IllegalTransactionStateException(
					"No existing transaction found for transaction marked with propagation 'mandatory'");
		}
		//这三种都是新建事务
		else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
			SuspendedResourcesHolder suspendedResources = suspend(null);
			if (debugEnabled) {
				logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
			}
			try {
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
				//开始获取链接，开启事务，绑定资源到当前线程
				doBegin(transaction, definition);
				prepareSynchronization(status, definition);
				return status;
			}
			catch (RuntimeException | Error ex) {
				resume(null, suspendedResources);
				throw ex;
			}
		}
		else {
			// Create "empty" transaction: no actual transaction, but potentially synchronization.
			if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
				logger.warn("Custom isolation level specified but no actual transaction initiated; " +
						"isolation level will effectively be ignored: " + definition);
			}
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
			return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
		}
	}
```

#### TransactionStatus

这儿返回的是**TransactionStatus**

```java
public interface TransactionStatus extends SavepointManager, Flushable {

	boolean isNewTransaction();

	boolean hasSavepoint();

	void setRollbackOnly();

	boolean isRollbackOnly();

	void flush();

	boolean isCompleted();
```

#### TransactionInfo

事务信息

```java
protected final class TransactionInfo {

		private final PlatformTransactionManager transactionManager;

		private final TransactionAttribute transactionAttribute;

		private final String joinpointIdentification;

		private TransactionStatus transactionStatus;

		private TransactionInfo oldTransactionInfo;
	}
```

### commitTransactionAfterReturning提交事务

逻辑到了AbstractPlatformTransactionManager#processRollback

```java
private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
		try {
			boolean unexpectedRollback = unexpected;

			try {
				triggerBeforeCompletion(status);
				//有savepoint，
				if (status.hasSavepoint()) {
					if (status.isDebug()) {
						logger.debug("Rolling back transaction to savepoint");
					}
					status.rollbackToHeldSavepoint();
				}
				else if (status.isNewTransaction()) {
					if (status.isDebug()) {
						logger.debug("Initiating transaction rollback");
					}
					//回滚事务
					doRollback(status);
				}
				else {
					// Participating in larger transaction
					//在一个事务中，就先设置回滚标识,等父事务一起回滚
					if (status.hasTransaction()) {
						if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
							if (status.isDebug()) {
								logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
							}
							doSetRollbackOnly(status);
						}
						else {
							if (status.isDebug()) {
								logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
							}
						}
					}
					else {
						logger.debug("Should roll back transaction but cannot - no transaction available");
					}
					// Unexpected rollback only matters here if we're asked to fail early
					if (!isFailEarlyOnGlobalRollbackOnly()) {
						unexpectedRollback = false;
					}
				}
			}
			catch (RuntimeException | Error ex) {
				triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
				throw ex;
			}

			triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);

			// Raise UnexpectedRollbackException if we had a global rollback-only marker
			if (unexpectedRollback) {
				throw new UnexpectedRollbackException(
						"Transaction rolled back because it has been marked as rollback-only");
			}
		}
		finally {
			cleanupAfterCompletion(status);
		}
```

### completeTransactionAfterThrowing回滚事务

```java
protected void completeTransactionAfterThrowing(TransactionInfo txInfo, Throwable ex) {
		//有事务才能回滚
		if (txInfo != null && txInfo.hasTransaction()) {
			if (logger.isTraceEnabled()) {
				logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
						"] after exception: " + ex);
			}
			//回滚在 (ex instanceof RuntimeException || ex instanceof Error)
			if (txInfo.transactionAttribute.rollbackOn(ex)) {
				try {
					txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
				}
				catch (TransactionSystemException ex2) {
					logger.error("Application exception overridden by rollback exception", ex);
					ex2.initApplicationException(ex);
					throw ex2;
				}
				catch (RuntimeException ex2) {
					logger.error("Application exception overridden by rollback exception", ex);
					throw ex2;
				}
				catch (Error err) {
					logger.error("Application exception overridden by rollback error", ex);
					throw err;
				}
			}
			else {
				// We don't roll back on this exception.
				// Will still roll back if TransactionStatus.isRollbackOnly() is true.
				try {
					txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
				}
				catch (TransactionSystemException ex2) {
					logger.error("Application exception overridden by commit exception", ex);
					ex2.initApplicationException(ex);
					throw ex2;
				}
				catch (RuntimeException ex2) {
					logger.error("Application exception overridden by commit exception", ex);
					throw ex2;
				}
				catch (Error err) {
					logger.error("Application exception overridden by commit error", ex);
					throw err;
				}
			}
		}
	}
```

# 总结

一个完整的事务介绍结束了。框架就是封装一切，透明一切，简化一切。本质的流程不会变

# 参考资料

<http://blog.sina.com.cn/s/blog_7ed8b1e90101mgas.html>

<https://my.oschina.net/wanyuxiang000/blog/277568>

---

原文出处：https://zhuxingsheng.github.io/2017/10/10/spring-transaction/