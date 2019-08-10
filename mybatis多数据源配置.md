---
title: springboot+mybatis多数据源配置
date: <2019-08-10 六>
tags: [springboot, mybatis, datasource, mysql]
---

mybatis多数据源
===============

构建主从数据库
--------------

1.  主库写入
2.  从库读取

springboot多数据源配置
----------------------

1.  修改application.yaml

    ``` {.yaml}
    # 配置主从数据源
    spring:
      datasource:
        master:
          jdbcUrl: jdbc:mysql://localhost:3306/mybatismaster?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
          username: root
          password: root
        slaver:
          jdbcUrl: jdbc:mysql://localhost:3306/mybatisslaver?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8&useSSL=true
          username: root
          password: root  
    ```

2.  修改configuration, 分别指定不同的mapper

    -   master

    ``` {.java}
    package com.lx.demo.config;

    import org.apache.ibatis.session.SqlSessionFactory;
    import org.apache.ibatis.session.SqlSessionFactoryBuilder;
    import org.mybatis.spring.SqlSessionFactoryBean;
    import org.mybatis.spring.SqlSessionTemplate;
    import org.mybatis.spring.annotation.MapperScan;
    import org.springframework.beans.factory.annotation.Qualifier;
    import org.springframework.boot.context.properties.ConfigurationProperties;
    import org.springframework.boot.jdbc.DataSourceBuilder;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.context.annotation.Primary;
    import org.springframework.jdbc.datasource.DataSourceTransactionManager;

    import javax.sql.DataSource;

    /**
    * 主域配置
    *  包括datasource session等, 都搞一套自己的
    */
    @Configuration
    @MapperScan(basePackages = "com.lx.demo.mapper.master", sqlSessionTemplateRef = "masterSqlSessionTemplate")
    public class DataSourceConfigMaster {

        /**
        * 创建主datasource, 这里多个数据源时必须指定一个为primary
        * @return
        */
        @ConfigurationProperties("spring.datasource.master")
        @Bean(name = "masterDataSource")
        @Primary
        public DataSource dataSource(){
            return DataSourceBuilder.create().build();
        }

        /**
        *
        * @param dataSource
        * @return
        * @throws Exception
        */
        @Bean(name = "masterSqlSessionFactory")
        @Primary
        public SqlSessionFactory sqlSessionFactory(@Qualifier("masterDataSource") DataSource dataSource) throws Exception {
            final SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
            sqlSessionFactoryBean.setDataSource(dataSource);
            //如果使用xml的方式, 需要设置mybatis配置文件路径
    //        sqlSessionFactoryBean.setConfigLocation("");
            return sqlSessionFactoryBean.getObject();
        }

        /**
        * 主域 事务管理器
        * @param dataSource
        * @return
        */
        @Bean(name = "masterDataSourceTransactionManager")
        @Primary
        public DataSourceTransactionManager dataSourceTransactionManager(@Qualifier("masterDataSource") DataSource dataSource){
            return new DataSourceTransactionManager(dataSource);
        }

        /**
        * 主 sqlsessiontemplet
        * @param sqlSessionFactory
        * @return
        */
        @Bean(name = "masterSqlSessionTemplate")
        @Primary
        public SqlSessionTemplate sqlSessionTemplate(@Qualifier("masterSqlSessionFactory") SqlSessionFactory sqlSessionFactory){
            return new SqlSessionTemplate(sqlSessionFactory);
        }
    }

    ```

    -   slaver

    ``` {.java}
    package com.lx.demo.config;

    import org.apache.ibatis.session.SqlSessionFactory;
    import org.mybatis.spring.SqlSessionFactoryBean;
    import org.mybatis.spring.SqlSessionTemplate;
    import org.mybatis.spring.annotation.MapperScan;
    import org.springframework.beans.factory.annotation.Qualifier;
    import org.springframework.boot.context.properties.ConfigurationProperties;
    import org.springframework.boot.jdbc.DataSourceBuilder;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.context.annotation.Primary;
    import org.springframework.jdbc.datasource.DataSourceTransactionManager;

    import javax.sql.DataSource;

    /**
    * 从域配置
    *  包括datasource session等, 都搞一套自己的
    */
    @Configuration
    @MapperScan(basePackages = "com.lx.demo.mapper.slaver", sqlSessionTemplateRef = "slaverSqlSessionTemplate")
    public class DataSourceConfigSlaver {

      /**
      * 创建从datasource, 这里多个数据源时必须指定一个为primary
      * @return
      */
      @ConfigurationProperties("spring.datasource.slaver")
      @Bean(name = "slaverDataSource")
      public DataSource dataSource(){
          return DataSourceBuilder.create().build();
      }

      /**
      *
      * @param dataSource
      * @return
      * @throws Exception
      */
      @Bean(name = "slaverSqlSessionFactory")
      public SqlSessionFactory sqlSessionFactory(@Qualifier("slaverDataSource") DataSource dataSource) throws Exception {
          final SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
          sqlSessionFactoryBean.setDataSource(dataSource);
          //如果使用xml的方式, 需要设置mybatis配置文件路径
    //        sqlSessionFactoryBean.setConfigLocation("");
          return sqlSessionFactoryBean.getObject();
      }

      /**
      * 从域 事务管理器
      * @param dataSource
      * @return
      */
      @Bean(name = "slaverDataSourceTransactionManager")
      public DataSourceTransactionManager dataSourceTransactionManager(@Qualifier("slaverDataSource") DataSource dataSource){
          return new DataSourceTransactionManager(dataSource);
      }

      /**
      * 从 sqlsessiontemplet
      * @param sqlSessionFactory
      * @return
      */
      @Bean(name = "slaverSqlSessionTemplate")
      public SqlSessionTemplate sqlSessionTemplate(@Qualifier("slaverSqlSessionFactory") SqlSessionFactory sqlSessionFactory){
          return new SqlSessionTemplate(sqlSessionFactory);
      }
    }

    ```

3.  分别测试写入读取

    ``` {.java}
    public class UserController {

        /**
        * 同时注入主从库, 代码控制多数据源使用
        * 该方式只是多来源，其实没有动态切换
        */
        @Autowired
        private UserMapperMaster userMapperMaster;

        @Autowired
        private UserMapperSlaver userMapperSlaver;

        /**
        * 获取数据走从域
        * @param id
        * @return
        */
        @GetMapping("/user/{id}")
        public User selectOne(@PathVariable Long id){
            final User user = userMapperSlaver.getOne(id);
            return user;
        }

        @GetMapping("/users")
        public List<User> users(){
            return userMapperSlaver.getAll();
        }

        /**
        * 写入数据走主域
        */
        @PostMapping("/users")
        public void save(@RequestBody User user){
            userMapperMaster.insert(user);
        }
    }

    ```

通过注解切换主从数据源(个数固定)
--------------------------------

与上述方式类似，只不过现在依赖一个bean, 然后通过注解方式切换数据源

1.  主从数据源初始化

    ``` {.java}
    /**
    * 配置多数据源
    */
    @Configuration
    @MapperScan(basePackages = "com.lx.demo.mapper.dynamic", sqlSessionTemplateRef = "dynamicSqlSessionTemplate")
    public class DataSourceConfigDynamic {

        /**
        * 创建主datasource, 这里多个数据源时必须指定一个为primary
        * @return
        */
        @ConfigurationProperties("spring.datasource.master")
        @Bean
        @Primary
        public DataSource dataSource1(){
            return DataSourceBuilder.create().build();
        }

        /**
        * 创建从datasource, 这里多个数据源时必须指定一个为primary
        * @return
        */
        @ConfigurationProperties("spring.datasource.slaver")
        @Bean
        public DataSource dataSource2(){
            return DataSourceBuilder.create().build();
        }

        /**
        * 动态数据源: 通过AOP在不同数据源之间动态切换
        * @return
        */
        @Bean(name = "dynamicDataSource")
        public DataSource dataSource() {
            DynamicDataSource dynamicDataSource = new DynamicDataSource();
            // 默认数据源
            dynamicDataSource.setDefaultTargetDataSource(dataSource1());

            // 配置多数据源
            Map<Object, Object> dsMap = new HashMap(5);
            dsMap.put("master", dataSource1());
            dsMap.put("slaver", dataSource2());
            dynamicDataSource.setTargetDataSources(dsMap);

            return dynamicDataSource;
        }

        /**
        *
        * @param dataSource
        * @return
        * @throws Exception
        */
        @Bean(name = "dynamicSqlSessionFactory")
        public SqlSessionFactory sqlSessionFactory(@Qualifier("dynamicDataSource") DataSource dataSource) throws Exception {
            final SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
            sqlSessionFactoryBean.setDataSource(dataSource);
            //如果使用xml的方式, 需要设置mybatis配置文件路径
    //        sqlSessionFactoryBean.setConfigLocation("");
            return sqlSessionFactoryBean.getObject();
        }

        /**
        * 主域 事务管理器
        * @param dataSource
        * @return
        */
        @Bean(name = "dynamicDataSourceTransactionManager")
        public DataSourceTransactionManager dataSourceTransactionManager(@Qualifier("dynamicDataSource") DataSource dataSource){
            return new DataSourceTransactionManager(dataSource);
        }

        /**
        * 主 sqlsessiontemplet
        * @param sqlSessionFactory
        * @return
        */
        @Bean(name = "dynamicSqlSessionTemplate")
        public SqlSessionTemplate sqlSessionTemplate(@Qualifier("dynamicSqlSessionFactory") SqlSessionFactory sqlSessionFactory){
            return new SqlSessionTemplate(sqlSessionFactory);
        }
    }

    ```

    -   **注意,
        这里不要使用注入bean的方式初始化动态数据源，否则会产生循环依赖，直接用方法datasource1()这种即可**

2.  主从数据源管理

    ``` {.java}
    /**
    * 主从数据源切换, 保存到threadLocal动态切换
    */
    @Slf4j
    public class DynamicDataSource extends AbstractRoutingDataSource {

        /**
        * 默认数据源
        */
        public static final String DEFAULT_DS = "master";

        private static final ThreadLocal<String> contextHolder = new ThreadLocal<>();

        /**
        * 设置数据源
        * @param dbType
        */
        public static void setDB(String dbType) {
            log.debug("切换到{}数据源", dbType);
            contextHolder.set(dbType);
        }

        /**
        *
        * 获取数据源名
        * @return
        */
        public static String getDB() {
            return (contextHolder.get());
        }

        /**
        *
        * 清除数据源名
        */
        public static void clearDB() {
            contextHolder.remove();
        }

        @Override
        protected Object determineCurrentLookupKey() {
            log.debug("数据源为{}", DynamicDataSource.getDB());

            return DynamicDataSource.getDB();
        }

    }

    ```

3.  利用反射执行方法时候动态切换数据源

    ``` {.java}
    /**
    * https://blog.csdn.net/neosmith/article/details/61202084
    */
    @Aspect
    @Component
    public class DynamicDataSourceAspect {

        @Before("@annotation(com.lx.demo.annotation.DS)")
        public void beforeSwitchDS(JoinPoint point) {

            //获得当前访问的class
            Class<?> className = point.getTarget().getClass();

            //获得访问的方法名
            String methodName = point.getSignature().getName();
            //得到方法的参数的类型
            Class[] argClass = ((MethodSignature) point.getSignature()).getParameterTypes();
            String dataSource = DynamicDataSource.DEFAULT_DS;
            try {
                // 得到访问的方法对象
                Method method = className.getMethod(methodName, argClass);

                // 判断是否存在@DS注解
                if (method.isAnnotationPresent(DS.class)) {
                    DS annotation = method.getAnnotation(DS.class);
                    // 取出注解中的数据源名
                    dataSource = annotation.value();
                }
            } catch (Exception e) {
                e.printStackTrace();
            }

            // 切换数据源
            DynamicDataSource.setDB(dataSource);
        }

        /**
        * Caused by: java.lang.IllegalArgumentException: error Type referred to is not an annotation type: DS
        * @param point
        */
        @After("@annotation(com.lx.demo.annotation.DS)")
        public void afterSwitchDS(JoinPoint point) {
            DynamicDataSource.clearDB();
        }
    }

    ```

4.  测试

    ``` {.java}
    /**
    * 增加动态主从数据源注解，　配置后自动切换数据源
    * @param user
    */
    @DS("slaver")
    @PostMapping("/users-dy")
    public void saveByDynamic(@RequestBody User user){
        userMapperDynamic.insert(user);
    }

    @DS("slaver")
    @GetMapping("/users-dy")
    public List<User> usersByDynamic(){
        return userMapperDynamic.getAll();
    }


    ```

动态数据源(可以任意添加, 暂未实现)
----------------------------------

问题列表
--------

1.  使用mybatisplus实现多数据源目前有问题,会报错
    org.apache.ibatis.binding.BindingException: Invalid bound statement
    (not found): com.yk.yearmeet.modular.service.UserService.list
2.  aspect @annotation(类的全限定名), 否则报错
    <https://blog.csdn.net/yangshangwei/article/details/77619875> Caused
    by: java.lang.IllegalArgumentException: error Type referred to is
    not an annotation type: DS 这种方式才可以,
    @Before(\"@annotation(com.lx.demo.annotation.DS)\")
