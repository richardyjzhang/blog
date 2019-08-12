# 背景介绍
目前项目组后端使用Spring Cloud全家桶，数据存储使用MySQL和MongoDB，其中MongoDB用于存储传感器上传的数据。  
之前听说过时序数据库这个东西，不过一直没有抽开时间研究。现在发现海量监测数据的统计确实有性能瓶颈了，就比较了几种时序数据库。  
具体的比较放在其他文章说吧，反正最后选用了基于PostgreSQL的TimeScaleDB，准备淘汰MongoDB。这样一来，程序中就需要同时连接两个结构化数据库，而且我准备继续使用Druid连接池，以及MyBatis框架。  
经过一下午的谷歌和试错，终于搞定了。网上这方面的资料很多，但完整但配置记录好像就很少了，这里记录一下。  
# Maven依赖
操作数据库这部分，显式用到了以下5个依赖：
```xml
<dependency>
  <goroupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
  <groupId>org.mybatis.spring.boot</groupId>
  <artifactId>mybatis-spring-boot-starter</artifactId>
  <version>1.3.2</version>
</dependency>
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>druid-spring-boot-starter</artifactedId>
  <version>1.1.10</version>
</dependency>
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <scope>runtime</scope>
</dependency>
<dependency>
  <groupId>org.postgresql</groupId>
  <artifactId>postgresql</artifactId>
  <version>42.2.6</version>
</dependency>
```
# 配置文件
我们有开发、内测、演示、生产等多套环境，且好多个服务同时在跑，因此使用Spring Cloud Config Server配置中心来集中管理配置。放到每一个Spring Boot中也是一样用的。这里给出yml的配置文件（部分，这里是内网开发数据库，用户名密码就随便放出来了）  
```yml
spring:
  datasource:
    name: mysql-dev
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      filters: stat
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://192.168.0.101:3308/
      username: root
      password: 123456
      ...
# 以下为新增PostgreSQL配置，名字随便起
postgres:
  datasource:
    name: postgres-dev
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      filters: stat
      driver-class-name: org.postgresql.Driver
      url: jdbc:postgresql://192.168.0.101:5432/monitor_data
      username: postgres
      password: 123456
```
要说明的一点是，这里我故意将PostgreSQL的配置放在了一个不标准的位置，这样的好处是，只有需要连接PostgreSQL的服务才需要加载相关的依赖，对已经运行起来的其他服务没有影响，适用于我们项目组只有几个服务需要连接PostgreSQL，其他还都使用MySQL的这种情况。  
# Mapper接口与xml文件的包
我们需要将使用不同数据库的Mapper接口与对应的xml文件放置在不同的包中，以便能够分别使用不同的数据库（在下面章节配置）。  
我将两种接口分别放置在`dao/mydao`和`dao/pgdao`两个包中，xml文件也放置在mapper下的my和pg两个文件夹中。  
这里要注意的是，配置文件（bootstrap.yml或application.yml)中，要给出正确的mybatis扫描路径配置，否则在运行时会提示找不到相应的Mapper  
```yml
mybatis:
  mapper-locations: classpath*:mapper/my*.yml,classpath*:mapper/pg/*.xml
```
此外要注意的是，xml中的namespace一定要和接口中的路径保持一致，在新工程中可能不容易犯错误，但是改动已有项目时，这里比较容易犯错，我下午在这里卡了半个小时才发现。  
# 数据库配置类
写数据库配置类的作用，就在于为不同的Mapper设定不同的数据源。这里主要参考了[这篇文章](https://netfilx.github.io/spring-boot/5.mybatis%E4%BD%BF%E7%94%A8%E5%A4%9A%E4%B8%AA%E6%95%B0%E6%8D%AE%E6%BA%90/muilt-data-source)(哈哈哈这个哥们，我一开始以为是Netflix的官方博客，结果发现是netfilx……)。  
我们有两个数据源，因此要写两个配置类，区别主要在于，为MySQL的配置类设置了@Primary  
```Java
@Configuration
@MapperScan(basePackages = MySQLConfig.PACKAGE, sqlSessionFactoryRef = "mySqlSessionFactory")
public class MySQLConfig {
  static final String PACKAGE = "com.zhangrichard.xxx.dao.mydao";
  static final String MAPPER_LOCATION = "classpath:mapper/my/*.xml";

  @Bean(value = "dataSourceMy")
  @Primary // 这里设置Primary
  @ConfigurationProperties("spring.datasource.druid") // 和配置文件区段对应
  public DataSource dataSourceMY() {
    return DruidDataSourceBuilder.create().build();
  }

  @Bean(name = "mySqlSessionFactory")
  @Primary // 这里设置Primary
  public SqlSessionFactory mySqlSessionFactory(@Qualifier("dataSourceMy") DataSource myDataSource)
    throws Exception {
    final SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
    sessionFactory.setDataSource(myDataSource);
    sessionFactory.setMapperLocations(new PathMachingResourcePatternResolver()
      .getResources(MySQLConfig.MAPPER_LOCATION));
    return sessionFactory.getObject();
  }
}
```
```Java
@Configuration
@MapperScan(basePackages = PostgresConfig.PACKAGE, sqlSessionFactoryRef = "postgresSessionFactory")
public class PostgresConfig {
  static final String PACKAGE = "com.zhangrichard.xxx.dao.pgdao";
  static final String MAPPER_LOCATION = "classpath:mapper/pg/*.xml";

  @Bean(value = "dataSourcePG")
  @ConfigurationProperties("postgres.datasource.druid") // 和配置文件区段对应
  public DataSource dataSourcePG() {
    return DruidDataSourceBuilder.create().build();
  }

  @Bean(name = "postgresSessionFactory")
  public SqlSessionFactory mySqlSessionFactory(@Qualifier("dataSourcePG") DataSource pgDataSource)
    throws Exception {
    final SqlSessionFactoryBean sessionFactory = new SqlSessionFactoryBean();
    sessionFactory.setDataSource(pgDataSource);
    sessionFactory.setMapperLocations(new PathMachingResourcePatternResolver()
      .getResources(PostgresConfig.MAPPER_LOCATION));
    return sessionFactory.getObject();
  }
}
```
可以看到，两个配置类分别为不同的Mapper包（及其对应的xml文件）指定了不同的数据源。这样程序跑起来后，发现运行符合我们的预期，该走MySQL的走MySQL，该查PostgreSQL的查PostgreSQL，很好。  
# 一点想法
发现网络上资料很多，但每个项目都有其与众不同的特点，不能完全根据网上的照抄而来，一定要理解主要精神，再结合自己的工程特点，不怕犯错，不断调试。另外，还要注意总结。  
我以上的代码都是在自己笔记本上对着公司台式机敲出来的，可能会有一些笔误，就当是坑一下那些拿来主义的吧✌️  