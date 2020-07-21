##  Spring-boot 笔记（一）— 整合 mybatis-plus 及代码生成器

### 1. 添加相关依赖

```xml
<!-- 3.1.1的版本存在无法将date类型转换为LocalDate类型的问题，替换方案为使用3.0.7版本 -->
<mybatis-plus-boot-starter.version>3.1.1</mybatis-plus-boot-starter.version>
<druid.version>1.1.17</druid.version>
<mysql-connector-java.version>8.0.16</mysql-connector-java.version>
<mybatis-plus-generator.version>3.1.1</mybatis-plus-generator.version>
<velocity-engine-core.version>2.1</velocity-engine-core.version>

<!-- 数据访问 -->
<dependency>
   <groupId>org.projectlombok</groupId>
   <artifactId>lombok</artifactId>
   <optional>true</optional>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>${mysql-connector-java.version}</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>${mybatis-plus-boot-starter.version}</version>
</dependency>
<!-- 解决无法将date类型转换为LocalDate类型的问题 -->
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis-typehandlers-jsr310</artifactId>
    <version>1.0.1</version>
</dependency>
<!-- 代码生成器 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-generator</artifactId>
    <version>${mybatis-plus-generator.version}</version>
</dependency>
<!-- 代码生成模板 -->
<dependency>
    <groupId>org.apache.velocity</groupId>
    <artifactId>velocity-engine-core</artifactId>
    <version>${velocity-engine-core.version}</version>
</dependency>
```



### 2. 编写生成器代码

```java
/**
 * myBatis-plus代码生成器
 *
 * @author Ding RD
 * @date 2019/6/18
 */
public class MyBatisPlusGenerator {

    public static void main(String[] args) {
        String projectPath = System.getProperty("user.dir") + File.separator + "test";

        // 代码生成器
        AutoGenerator mpg = new AutoGenerator();

        // 全局配置
        GlobalConfig gc = new GlobalConfig();
        gc.setOutputDir(projectPath);
        // 是否覆盖
        gc.setFileOverride(true);
        gc.setActiveRecord(true);
        // XML 二级缓存
        gc.setEnableCache(false);
        // XML ResultMap
        gc.setBaseResultMap(true);
        // XML columList
        gc.setBaseColumnList(false);
        gc.setAuthor("Ding RD");
        gc.setMapperName("%sMapper");
        gc.setXmlName("%sMapper");
        gc.setServiceName("%sService");
        gc.setServiceImplName("%sServiceImpl");
        gc.setControllerName("%sController");
        mpg.setGlobalConfig(gc);

        // 数据源配置
        DataSourceConfig dsc = new DataSourceConfig();
        dsc.setUrl("jdbc:mysql://localhost:3306/demo?useUnicode=true&useSSL=false&characterEncoding=utf8&serverTimezone=Asia/Shanghai");
        dsc.setDriverName("com.mysql.cj.jdbc.Driver");
        dsc.setUsername("root");
        dsc.setPassword("123456");
        mpg.setDataSource(dsc);

        // 策略配置
        StrategyConfig strategy = new StrategyConfig();
        strategy.setTablePrefix("t_base");
        strategy.setNaming(NamingStrategy.underline_to_camel);
        strategy.setColumnNaming(NamingStrategy.underline_to_camel);
        strategy.setEntityLombokModel(true);
        strategy.setInclude("t_base_user");
        // 自动生成实体时生成字段注解
        strategy.setEntityTableFieldAnnotationEnable(true);
        mpg.setStrategy(strategy);

        // 包配置
        PackageConfig pc = new PackageConfig();
        pc.setParent(null);
        pc.setEntity("com.example.springbootdemo.entity");
        pc.setMapper("com.example.springbootdemo.mapper");
        pc.setXml("com.example.springbootdemo.mapper");
        pc.setService("com.example.springbootdemo.service");
        pc.setServiceImpl("com.example.springbootdemo.service.impl");
        mpg.setPackageInfo(pc);

        InjectionConfig injectionConfig = new InjectionConfig() {
            // 自定义属性注入:abc
            // 在.ftl(或者是.vm)模板中，通过${cfg.abc}获取属性
            @Override
            public void initMap() {
                Map<String, Object> map = new HashMap<>();
                map.put("abc", this.getConfig().getGlobalConfig().getAuthor() + "-mp");
                this.setMap(map);
            }
        };
        //  配置自定义属性注入
        mpg.setCfg(injectionConfig);

        // 执行生成
        mpg.execute();
        // 打印注入设置
        System.err.println(mpg.getCfg().getMap().get("abc"));
    }
}
```

- 注意：使用 MySQL 8.0 以上版本（MySQL 连接驱动和版本都是 8.0 以上）的时候，由于数据库和系统时区差异，需要在 MySQL 中执行 set global time_zone='+8:00' 或者访问数据库的 Url 后面加上以下的语句：

  ```xml
  serverTimezone=Asia/Shanghai
  ```

