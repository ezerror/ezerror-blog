---
external: false
title: Mybatis动态数据源切换
date: 2020-08-23 01:05:23
tags:
- 后端
- Java
- mybatis
---

# 1. 结构设计

首先看一下这个功能的架构设计

![Dynamic DataSource Config](/images/Dynamic%20DataSource%20Config.png)

- 我们默认有一个缺省的数据源 `Deault DataSource` ,他是从配置文件中获取的，在应用刚开始启动时就注入，而在某些情况下，我们需要在一次操作中短时或长时间的对其它的数据库进行操作，这就是所谓的数据源切换
- 为了保证新添加的数据源不会对其它线程的操作有英影响，我们使用ThreadLocal来存储当前使用的数据源的相关信-息，创建上下文 `DataSourceContextHolder` 类，来保留当前线程的数据源
- 在Mybatis读取配置创建 `Session` 时，会注入 `DynamicDataSource` ，而 `DynamicDataSource` 通过读取当前线程变量可以获取自己设置的数据源，如果没有设置会注入默认的数据源，这个数据源来自于配置文件，调用流程如下图

![Dynamic DataSource Call Process](/images/Dynamic%20DataSource%20Call%20Process.png)

# 2. 实现

```java
public class DataSourceContextHolder {
    // 当前线程使用的数据源，为null表示默认数据源
    private static final ThreadLocal<DataSource> contextHolder = new InheritableThreadLocal<DataSource>();
    // 当前线程使用过的数据源,方便事务
    private static final List<DataSource> dataSources = new ArrayList<>();
    // 全局外部数据源缓存
    private static final HashMap<String, DataSource> map = new HashMap<>();

    // 设置当前线程的数据源
    public static void setDataSource(DruidDataSource datasource) {
        if (!map.containsKey(datasource.getUrl())) {
            contextHolder.set(datasource);
            map.put(datasource.getUrl(), datasource);
        }
        else {
            contextHolder.set(map.get(datasource.getUrl()));
        }
        dataSources.add(contextHolder.get());
    }

    // 获取数据源
    public static DataSource getDataSource() {
        return contextHolder.get();
    }

    // 获取数据源
    public static List<DataSource> getThreadDataSources() {
        return dataSources;
    }

    public static void clearCache() {
        map.clear();
    }

    public static void clearDataSource() {
        contextHolder.remove();
    }

}
```

```java
@Configuration
public class DataSourceConfig {
    
    
    @Bean(name = "defaultDataSource")
    @ConfigurationProperties(prefix = "jdbc")
    public DataSource defaultDataSource() {
        return new DruidDataSource();
    }

    @Bean(name = "dynamicDataSource")
    public DynamicDataSource dynamicDataSource() {
        return new DynamicDataSource();
    }
}
```

```java
/**
 * 动态数据源管理类
 */
public class DynamicDataSource extends AbstractDataSource {
	
    // 注入默认数据源
    @Resource(name = "defaultDataSource")
    private DataSource defaultDs;

    protected DataSource determineTargetDataSource() {
        // 获取当前线程的数据源
        DataSource dataSource = DataSourceContextHolder.getDataSource();
        // 如果没有设置动态数据源，则返回默认数据源
        if (dataSource == null) {
            return defaultDs;
        }
        return dataSource;
    }

    @Override
    public Connection getConnection() throws SQLException {
        return determineTargetDataSource().getConnection();
    }

    @Override
    public Connection getConnection(String username, String password) throws SQLException {
        return determineTargetDataSource().getConnection(username, password);
    }
}
```

```java
@Configuration
public class MyBatisConfig {

    @Resource(name = "dynamicDataSource")
    private DynamicDataSource dynamicDataSource;

    @Bean
    public SqlSessionFactory sqlSessionFactory() throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setObjectWrapperFactory(new MapWrapperFactory());
        // 设置数据源为DynamicDataSource
        sqlSessionFactoryBean.setDataSource(dynamicDataSource);
        sqlSessionFactoryBean.setTypeAliasesPackage("me.ezerror.pojo");
        return sqlSessionFactoryBean.getObject();
    }

    @Bean
    public SqlSessionTemplate financialMasterSqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }

}
```

# 3. 使用方式

```java
// 新建数据源
DruidDataSource dataSource = new DruidDataSource();
dataSource.setUrl(url);
dataSource.setPassword(pwd);
dataSource.setUsername(username);
// 设置数据源
DataSourceContextHolder.setDataSource(dataSource);
// 数据库操作
List<Moment> moments = recordService.findMoment();
// 清除数据源，还原到默认数据源
DataSourceContextHolder.clearDataSource();
```

