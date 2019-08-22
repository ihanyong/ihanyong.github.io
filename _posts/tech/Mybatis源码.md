Mybatis源码.md

# SqlSessionFactory 接口及实现

SqlSessionFactory 是Mybaits的一个重要的概念， 顾名思义，就是用来获取 SqlSession 对象的。  
但怎么获得一个SqlSessionFactory对象呢？ 通过调用SqlSessionFactoryBuilder.build() 方法来解析配置文件生成一个SqlSessionFactory对象。 具体解析过程这里不做深入讨论。 

SqlSessionFactory 定义两类方法
- SqlSession openSession(*); // 接收各种参数来生成SqlSession
- Configuration getConfiguration(); // 获取解析出来的配置对象


SqlSessionFactory 接口有两个内置实现
- DefaultSqlSessionFactory
- SqlSessionManager



### DefaultSqlSessionFactory 的实现
getConfiguration 方法的实现很直白，就是将构造函数传入的 Configuration 对象返回。 

各种重载 openSession 方法的实现最终都是调用下面两个方法返回一个SqlSession对象。 
+ openSessionFromDataSource
+ openSessionFromConnection

而这两个方法的实现基本一样，最终都是返回一人`DefaultSqlSession`对象，
不同点在获取事物的方式不一样， 分别从数据源和数据库连接获取事务。

```java


  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      // 从数据源获取事务
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      final Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }



  private SqlSession openSessionFromConnection(ExecutorType execType, Connection connection) {
    try {
      boolean autoCommit;
      try {
        autoCommit = connection.getAutoCommit();
      } catch (SQLException e) {
        // Failover to true, as most poor drivers
        // or databases won't support transactions
        autoCommit = true;
      }      
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      // 从连接获取事务
      final Transaction tx = transactionFactory.newTransaction(connection);
      final Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }

```

### SqlSessionManager 的实现
SqlSessionManager 类实现了 SqlSessionFactory 和 SqlSession接口。 

SqlSessionFactory的实现部分本质上就是一个对DefaultSqlSessionFactory对象的静态代理。 在newInstance()工厂方法中，调用SqlSessionFactoryBuilder.build()来生成DefaultSqlSessionFactory对象，并使用 DefaultSqlSessionFactory 对象来实例化一个 SqlSessionManager 对象。
```java
  public static SqlSessionManager newInstance(Reader reader, String environment) {
    return new SqlSessionManager(new SqlSessionFactoryBuilder().build(reader, environment, null));
  }
  // ......
```


SqlSession 的实现部分是基于ThreadLocal的Java动态代理模式。
先看下构造函数：
``` java
  private SqlSessionManager(SqlSessionFactory sqlSessionFactory) {
    this.sqlSessionFactory = sqlSessionFactory;

    // 生成一个 SqlSession 的动态代理对象
    this.sqlSessionProxy = (SqlSession) Proxy.newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[]{SqlSession.class},
        new SqlSessionInterceptor());
  }
``` 
SqlSessionInterceptor 是SqlSessionManager的内部成员类：
```java

  private class SqlSessionInterceptor implements InvocationHandler { 
    public SqlSessionInterceptor() {
        // Prevent Synthetic Access
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

      // ThreadLocal<SqlSession> localSqlSession
      // 尝试从当前线程获取sqlSession对象
      final SqlSession sqlSession = SqlSessionManager.this.localSqlSession.get();
      if (sqlSession != null) {
        // 如果当前线程已经存在sqlSession对象，直接在该sqlSession对象上调用对应的方法
        try {
          return method.invoke(sqlSession, args);
        } catch (Throwable t) {
          throw ExceptionUtil.unwrapThrowable(t);
        }
      } else {
        // 如果当前线程没有sqlSession对象，
        // 调用 获取一个新的sqlSession对象，并调用对应的方法
        try (SqlSession autoSqlSession = openSession()) {
          try {
            final Object result = method.invoke(autoSqlSession, args);
            autoSqlSession.commit();
            return result;
          } catch (Throwable t) {
            autoSqlSession.rollback();
            throw ExceptionUtil.unwrapThrowable(t);
          }
        }
      }
    }
  }

```

继续理解SqlSession 之前先看下 executor 包中的四大接口。 

- Executor : 
- StatementHandler : 
- ParameterHandler : 
- ResultSetHandler : 

### Executor 接口及实现

定义的主要方法
- update
- query
- queryCursor


实现类
- BaseExecutor
- SimpleExecutor
- BatchExecutor
- ReuseExecutor
- CachingExecutor


BaseExecutor 类是一个模板方法类。 

定义了本地缓存

update 方法中先调用`clearLocalCache()`方法来清除本地缓存，然后调用子类的 `doUpdate()` 实现

query 方法先尝试从本缓存查找缓存的结果，如果命中，直接返回；没有命中，再从DB中查询（调用子类`doQuery()` 实现）

SimpleExecutor 就是最基本的实现
BatchExecutor 对相同的 update statemet 进行合并batch处理
ReuseExecutor 重用 Statement


### StatementHandler 接口及实现

定义的主要方法
- prepare
- parameterize
- batch
- update
- query
- queryCursor
- getBoundSql
- getParameterHandler


实现类
- BaseStatementHandler
- SimpleStatementHandler
- PreparedStatementHandler
- CallableStatementHandler
- RoutingStatementHandler


### ParameterHandler 接口及实现
### ResultSetHandler 接口及实现




# SqlSession 接口及实现

DefaultSqlSession

# Mapper 与 SqlSession的关系
