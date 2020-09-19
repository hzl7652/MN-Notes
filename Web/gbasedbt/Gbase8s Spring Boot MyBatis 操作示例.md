# GBase8s Spring Boot MyBatis 操作示例

本文将创建一个简单的 Spring Boot 项目结构，并演示如何使用 Mybatis 进行GBase 8s 数据库的数据处理工作(插入，选择，更新和删除)，以及分页显示。
## 1. MyBatis + XML方式
本文我们会使用 mybatis-spring-boot-starter 自动化配置 MyBatis 主要配置。同时，在 XML 中编写相应的 SQL 操作。
### 1.1 引入依赖
在 pom.xml 文件中，引入相关依赖。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.5.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <artifactId>GBase8s-Spring-Boot-MyBatis-Demo</artifactId>

    <name>GBase8s-Spring-Boot-MyBatis-Demo</name>
    <description>GBase8s-Spring-Boot-MyBatis-Demo</description>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>

<!-- GBase8s jdbc驱动 由于没有发布到maven中央仓库，使用 systemPath 方式引用-->
        <dependency>
            <groupId>com.gbasedbt</groupId>
            <artifactId>gbasedbtjdbc</artifactId>
            <version>1.0.0</version>
            <scope>system</scope>
            <systemPath>${project.basedir}/lib/ifxjdbc.jar</systemPath>
        </dependency>

        <!-- 实现对 MyBatis 的自动化配置 -->
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>2.1.1</version>
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
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <includeSystemScope>true</includeSystemScope>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>

```
### 1.2 Application
创建 Application.java 类，配置 @MapperScan 注解，扫描对应 Mapper 接口所在的包路径。代码如下：
```java
// Application.java

@SpringBootApplication
@MapperScan(basePackages = "cn.gbase.gbase8s.demo.mybatis.mapper")
public class Application {
}
```
cn.gbase.8s.demo.mybatis.mapper 包路径下，就是我们 Mapper 接口所在的包路径。
### 1.3 应用配置文件
在 resources 目录下，创建 application.yml 配置文件。配置如下：
```yaml
spring:
  # datasource 数据源配置内容
  datasource:
    url: jdbc:gbasedbt-sqli://192.168.56.102:9088/test:GBASEDBTSERVER=ol_gbasedbt1210;NEWCODESET=utf-8,utf8,57372;DB_LOCALE=zh_cn.utf8;
    driver-class-name: com.gbasedbt.jdbc.IfxDriver
    username: gbasedbt
    password: gbasedbt

# mybatis 配置内容
mybatis:
  config-location: classpath:mybatis-config.xml # 配置 MyBatis 配置文件路径
  mapper-locations: classpath:mapper/*.xml # 配置 Mapper XML 地址
  type-aliases-package: cn.iocoder.springboot.lab12.mybatis.dataobject # 配置数据库实体包路径
```
### 1.4应用配置文件
在 resources 目录下，创建 mybatis-config.xml 配置文件。配置如下：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

    <settings>
        <!-- 使用驼峰命名法转换字段。 -->
        <setting name="mapUnderscoreToCamelCase" value="true"/>
    </settings>

    <typeAliases>
        <typeAlias alias="Integer" type="java.lang.Integer"/>
        <typeAlias alias="Long" type="java.lang.Long"/>
        <typeAlias alias="HashMap" type="java.util.HashMap"/>
        <typeAlias alias="LinkedHashMap" type="java.util.LinkedHashMap"/>
        <typeAlias alias="ArrayList" type="java.util.ArrayList"/>
        <typeAlias alias="LinkedList" type="java.util.LinkedList"/>
    </typeAliases>

    <typeHandlers>
        <!-- 配置以支持lvarchar -->
        <typeHandler handler="org.apache.ibatis.type.StringTypeHandler"
                     jdbcType="LONGVARCHAR" javaType="String" />
    </typeHandlers>



</configuration>
```
因为在数据库中的表的字段，我们是使用下划线风格，而数据库实体的字段使用驼峰风格，所以通过 **mapUnderscoreToCamelCase = true** 来自动转换。
### 1.5 UserDO 数据表对应实体类
在 cn.iocoder.springboot.lab12.mybatis.dataobject 包路径下，创建 UserDO.java 类，用户 DO 。代码如下：
```java
public class UserDO {

    /**
     * 用户编号
     */
    private Integer id;
    /**
     * 账号
     */
    private String username;
    /**
     * 密码
     *
     */
    private String password;
    /**
     * 创建时间
     */
    private Date createTime;

    // ... 省略 setting/getting 方法

}
```
对应的创建表的 SQL 如下：
```sql
create table users
(
  id serial not null,
  username varchar(60),
  password varchar(32),
  create_time  DATETIME YEAR TO FRACTION(5) NOT NULL,
  primary key(id)
);
```
### 1.6 UserMapper
在 cn.iocoder.springboot.lab12.mybatis.mapper 包路径下，创建 UserMapper 接口。代码如下：
```java
@Repository
public interface UserMapper {

    int insert(UserDO user);

    int updateById(UserDO user);

    int deleteById(@Param("id") Integer id);

    int deleteAll();

    UserDO selectById(@Param("id") Integer id);

    UserDO selectByUsername(@Param("username") String username);

    List<UserDO> selectByIds(@Param("ids") Collection<Integer> ids);

    List<UserDO> selectByPage(@Param("pageNum") int pageNum,@Param("pageSize") int pageSize);

}
```
在 resources/mapper 路径下，创建 UserMapper.xml 配置文件。代码如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cn.gbase.gbase8s.demo.mybatis.mapper.UserMapper">

    <sql id="FIELDS">
        id, username, password, create_time
    </sql>

    <insert id="insert" parameterType="UserDO" useGeneratedKeys="true" keyProperty="id">
        INSERT INTO users (
          username, password, create_time
        ) VALUES (
          #{username}, #{password}, #{createTime}
        )
    </insert>

    <update id="updateById" parameterType="UserDO">
        UPDATE users
        <set>
            <if test="username != null">
                , username = #{username}
            </if>
            <if test="password != null">
                , password = #{password}
            </if>
        </set>
        WHERE id = #{id}
    </update>

    <delete id="deleteById" parameterType="Integer">
        DELETE FROM users
        WHERE id = #{id}
    </delete>

    <delete id="deleteAll" >
        truncate table users
    </delete>

    <select id="selectById" parameterType="Integer" resultType="UserDO">
        SELECT
        <include refid="FIELDS" />
        FROM users
        WHERE id = #{id}
    </select>

    <select id="selectByUsername" parameterType="String" resultType="UserDO">
        SELECT
        FIRST 1
        <include refid="FIELDS" />
        FROM users
        WHERE username = #{username}

    </select>

    <select id="selectByIds" resultType="UserDO">
        SELECT
        <include refid="FIELDS" />
        FROM users
        WHERE id IN
        <foreach item="id" collection="ids" separator="," open="(" close=")" index="">
            #{id}
        </foreach>
    </select>

    <!-- 分页查询所有用户  -->
    <select id="selectByPage" resultType="UserDO">
        select
        skip #{pageNum} first #{pageSize}
        <include refid="FIELDS" />
          from users
    </select>

</mapper>
```
### 1.7 简单测试
创建 UserMapperTest 测试类，我们来测试一下简单的 UserMapper 的每个操作。代码如下：
```java
package cn.gbase.gbase8s.demo.mybatis.mapper;

import cn.gbase.gbase8s.demo.Application;
import cn.gbase.gbase8s.demo.mybatis.dataobject.UserDO;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import javax.annotation.PostConstruct;
import java.util.Arrays;
import java.util.Date;
import java.util.List;
import java.util.UUID;

@SpringBootTest(classes = Application.class)
class UserMapperTest {

    @Autowired
    private UserMapper userMapper;

    @PostConstruct
    public void init() {
        userMapper.deleteAll();

        UserDO user = new UserDO().setPassword(UUID.randomUUID().toString())
                .setUsername("test1").setCreateTime(new Date());
        userMapper.insert(user);
        UserDO user2 = new UserDO().setPassword(UUID.randomUUID().toString())
                .setUsername("test2").setCreateTime(new Date());
        userMapper.insert(user2);
        UserDO user3 = new UserDO().setPassword(UUID.randomUUID().toString())
                .setUsername("test3").setCreateTime(new Date());
        userMapper.insert(user3);
        UserDO user4 = new UserDO().setPassword(UUID.randomUUID().toString())
                .setUsername("test4").setCreateTime(new Date());
        userMapper.insert(user4);
        UserDO user5 = new UserDO().setPassword(UUID.randomUUID().toString())
                .setUsername("test5").setCreateTime(new Date());
        userMapper.insert(user5);
    }

    @Test
    void insert() {
        UserDO user = new UserDO().setUsername(UUID.randomUUID().toString())
                .setPassword("nicai").setCreateTime(new Date());
        userMapper.insert(user);
    }

    @Test
    void updateById() {
        UserDO updateUser = userMapper.selectByUsername("test1");
        updateUser.setPassword("wobucai");

        userMapper.updateById(updateUser);
    }

    @Test
    void deleteById() {
        userMapper.deleteById(2);
    }

    @Test
    void selectById() {
        userMapper.selectById(1);
    }

    @Test
    void selectByUsername() {
        userMapper.selectByUsername("test1");
    }

    @Test
    void selectByIds() {

        List<UserDO> users = userMapper.selectByIds(Arrays.asList(1, 3));
        System.out.println("users：" + users.size());
    }

    @Test
    void selectByPage() {
        List<UserDO> users = userMapper.selectByPage(1, 2);
        System.out.println("users：" + users.size());
    }
}
```