# 合家云项目基础搭建

### 1、根据需求文档分析整个项目，进行相关表的设计

通过需求文档分析各个模块需要的表以及相应的表之间的关系，此阶段主要是想让大家熟悉这个过程，后续实际开发的时候我会将表结构直接给出，大家做对比即可，记住一句话，

**表设计没有对错之分，只有合适与不合适**

### 2、搭建前端项目

此项目的开发我们使用的是阿里开源的前端框架Ant Design,此框架是使用vue来完成具体功能的，因为在现在的公司的开发中，几乎都是前后端分离，前端工程师完成前端功能，后端工程师完成后端逻辑的编写，我们教学的侧重点在于后端，因此前端直接使用给大家提供好的模板，大家只需要下载即可。

操作步骤：

1、下载项目并完成解压的功能

2、在当前项目的根路径下打开cmd，然后运行npm install

3、在整个安装过程中一般不会出现任何错误，如果出现错误

4、安装完成之后，可以使用 npm run serve命令来进行 运行

### 3、搭建后端项目

后端我们抛弃使用ssm的这种老的项目搭建模式，使用现在应用最多的springboot来进行实现。

操作步骤：

1、创建springboot项目

2、导入需要的依赖

这个位置，我们可以先引入：Spring Web、MySql Driver、MyBatis、DevTools、Mybatis-Plus

pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.mashibing</groupId>
    <artifactId>family_service_platform</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>family_service_platform</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>2.1.2</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.9</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <!--mybatis代码快速生成-->
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-generator</artifactId>
            <version>3.3.1</version>
        </dependency>
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.3.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.velocity</groupId>
            <artifactId>velocity-engine-core</artifactId>
            <version>2.2</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/log4j/log4j -->
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.17</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
        <resources>
            <resource>
                <directory>src/main/java</directory>
                <includes>
                    <include>**/*.xml</include>
                </includes>
            </resource>
        </resources>
    </build>
</project>
```

3、修改配置文件

application.yaml

```yaml
#启动端口
server:
  port: 8080
#数据源配置（数据库连接池使用默认的SpringBoot HikariCP）
spring:
  datasource:
    username: root
    password: root
    url: jdbc:mysql://localhost:3306/family_service_platform?useSSL=false&serverTimezone=UTC
    driver-class-name: com.mysql.cj.jdbc.Driver
#配置MyBatis
mybatis:
  mapper-locations: classpath:com/mashibing/mapper/*.xml
  configuration:
    map-underscore-to-camel-case: true

#SQL语句日志打印
logging:
  level:
    com:
      mashibing:
        mapper: debug
```

log4j.properties

```properties
# 全局日志配置%n
log4j.rootLogger=info, stdout
# MyBatis 日志配置
log4j.logger.com.mashibing.mapper=TRACE
# 控制台输出
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%5p [%t] - %m

log4j.logger.java.sql=DEBUG
log4j.logger.java.sql.Connection = DEBUG  
log4j.logger.java.sql.Statement = DEBUG  
log4j.logger.java.sql.PreparedStatement = DEBUG  
log4j.logger.java.sql.ResultSet = DEBUG
```

4、通过mybatis-plus反向生成对应的实体类

```java
package com.mashibing;

import com.baomidou.mybatisplus.annotation.IdType;
import com.baomidou.mybatisplus.generator.AutoGenerator;
import com.baomidou.mybatisplus.generator.config.DataSourceConfig;
import com.baomidou.mybatisplus.generator.config.GlobalConfig;
import com.baomidou.mybatisplus.generator.config.PackageConfig;
import com.baomidou.mybatisplus.generator.config.StrategyConfig;
import com.baomidou.mybatisplus.generator.config.rules.NamingStrategy;
import org.junit.jupiter.api.Test;

public class MyTest {


    @Test
    public void testGenerator() {
        AutoGenerator autoGenerator = new AutoGenerator();

        //全局配置
        GlobalConfig globalConfig = new GlobalConfig();
        globalConfig.setAuthor("lian")
                .setOutputDir("E:\\self_project\\family_service_platform\\src\\main\\java")//设置输出路径
                .setFileOverride(true)//设置文件覆盖
                .setIdType(IdType.AUTO)//设置主键生成策略
                .setServiceName("%sService")//service接口的名称
                .setBaseResultMap(true)//基本结果集合
                .setBaseColumnList(true)//设置基本的列
                .setControllerName("%sController");

        //配置数据源
        DataSourceConfig dataSourceConfig = new DataSourceConfig();
        dataSourceConfig.setDriverName("com.mysql.cj.jdbc.Driver").setUrl("jdbc:mysql://localhost:3306/family_service_platform?serverTimezone=UTC")
                .setUsername("root").setPassword("123456");

        //策略配置
        StrategyConfig strategyConfig = new StrategyConfig();
        strategyConfig.setCapitalMode(true)//设置全局大写命名
                .setNaming(NamingStrategy.underline_to_camel)//数据库表映射到实体的命名策略
                //.setTablePrefix("tbl_")//设置表名前缀
                .setInclude();

        //包名配置
        PackageConfig packageConfig = new PackageConfig();
        packageConfig.setParent("com.mashibing").setMapper("mapper")
                .setService("service").setController("controller")
                .setEntity("bean").setXml("mapper");

        autoGenerator.setGlobalConfig(globalConfig).setDataSource(dataSourceConfig)
                .setStrategy(strategyConfig).setPackageInfo(packageConfig);

        autoGenerator.execute();
    }
}
```

5、运行springboot项目，保证项目能够运行起来。

6、运行之后发现项目报错，报错原因是因为mapper对象无法自动装配，因此需要在启动的application类上添加@MapperScan注解或者在每一个mapper的接口上添加@Mapper注解，当然不可否认的是我们当前项目的表比较多，因此在每一个Mapper上添加@Mapper注解比较麻烦。

---

总结，截止到此处为止，我们需要环境准备工作已经完成，下面开始进行下一步的开发。
