## jdbc preparedstatement 驱动预编译分页sql报转换错误问题分析
## 调查结论
1. 目前jdbc驱动不支持 SQL  order by    SKIP ? FIRST ? 这种方式方式设置分页参数，执行SQL报错
2. 支持 SQL  order by    SKIP XX FIRST XX XX为具体数字
3. 现场问题反馈反应 GBase Data Studio 支持分页应为第二种形式 
4. 目前无 JDBC 源码，无法进一步分析 

## 支持分页SQL语句
1. SELECT SKIP 0 FIRST 1 字段 FROM 分页硬编码
2. SELECT SKIP ? FIRST ? 字段 FROM 分页参数绑定变量方式
3. SELECT  字段 FROM WHERE ORDER BY SKIP 0 FIRST 1 分页硬编码
## 代码测试用例  
 ``` java
    //测试通过
	@Test
    void notPreparedStatementSetPageInfo(){
        log.info("Querying for customer records where first_name = 'Josh':");
        String sql = "SELECT id, first_name, last_name FROM customers where first_name = ?  order by id desc SKIP 0 FIRST 1 ";
        Object[] params = {"Josh"};
        querySql(sql, params);
    }
	//测试通过
    @Test
    void notPreparedStatementSetPageInfoFirst(){
        log.info("Querying for customer records where first_name = 'Josh':");
        String sql = "SELECT SKIP 0 FIRST 1 id, first_name, last_name FROM customers where first_name = ?  order by id desc  ";
        Object[] params = {"Josh"};
        querySql(sql, params);
    }
	//测试异常
    @Test
    void skipFirstAtLastQuery() {

        log.info("Querying for customer records where first_name = 'Josh':");
        String sql = "SELECT id, first_name, last_name FROM customers where first_name = ?  order by id desc SKIP ? FIRST ? ";
        Object[] params = {"Josh", 0, 1};
        try{
            querySql(sql, params);
        }catch (Exception e){
            log.error("",e);
        }
        //assertThrows(UncategorizedSQLException.class,() ->  querySql(sql, params));
        //querySql(sql, params);
    }
	//测试通过
    @Test
    void skipFirstAtStartQuery() {

        log.info("Querying for customer records where first_name = 'Josh':");
        String sql = "SELECT SKIP ? FIRST ?  id, first_name, last_name FROM customers where first_name = ?  order by id desc  ";
        Object[] params = {0, 1,"Josh" };
        querySql(sql, params);
    }

    private void querySql(String sql, Object[] params) {
        jdbcTemplate.query(
                sql, params,
                rowMapper
        ).forEach(customer -> log.info(customer.toString()));
    }
 ```
## 程序异常信息
``` java
org.springframework.jdbc.UncategorizedSQLException: PreparedStatementCallback; uncategorized SQLException for SQL [SELECT id, first_name, last_name FROM customers where first_name = ?  order by id desc SKIP ? FIRST ? ]; SQL state [IX000]; error code [-3420]; null; nested exception is java.sql.SQLException
	at org.springframework.jdbc.support.AbstractFallbackSQLExceptionTranslator.translate(AbstractFallbackSQLExceptionTranslator.java:89) ~[spring-jdbc-5.2.4.RELEASE.jar:5.2.4.RELEASE]
	at org.springframework.jdbc.support.AbstractFallbackSQLExceptionTranslator.translate(AbstractFallbackSQLExceptionTranslator.java:81) ~[spring-jdbc-5.2.4.RELEASE.jar:5.2.4.RELEASE]
	at org.springframework.jdbc.support.AbstractFallbackSQLExceptionTranslator.translate(AbstractFallbackSQLExceptionTranslator.java:81) ~[spring-jdbc-5.2.4.RELEASE.jar:5.2.4.RELEASE]
	at org.springframework.jdbc.core.JdbcTemplate.translateException(JdbcTemplate.java:1443) ~[spring-jdbc-5.2.4.RELEASE.jar:5.2.4.RELEASE]
	at org.springframework.jdbc.core.JdbcTemplate.execute(JdbcTemplate.java:633) ~[spring-jdbc-5.2.4.RELEASE.jar:5.2.4.RELEASE]
	at org.springframework.jdbc.core.JdbcTemplate.query(JdbcTemplate.java:669) ~[spring-jdbc-5.2.4.RELEASE.jar:5.2.4.RELEASE]
	at org.springframework.jdbc.core.JdbcTemplate.query(JdbcTemplate.java:700) ~[spring-jdbc-5.2.4.RELEASE.jar:5.2.4.RELEASE]
	at org.springframework.jdbc.core.JdbcTemplate.query(JdbcTemplate.java:712) ~[spring-jdbc-5.2.4.RELEASE.jar:5.2.4.RELEASE]
	at org.springframework.jdbc.core.JdbcTemplate.query(JdbcTemplate.java:763) ~[spring-jdbc-5.2.4.RELEASE.jar:5.2.4.RELEASE]
	at com.example.demo.gbasedbt.Issue2210Test.querySql(Issue2210Test.java:66) ~[test-classes/:na]
	at com.example.demo.gbasedbt.Issue2210Test.skipFirstAtLastQuery(Issue2210Test.java:48) ~[test-classes/:na]
	
Caused by: java.sql.SQLException: null
	at com.gbasedbt.util.IfxErrMsg.getSQLException(IfxErrMsg.java:408) ~[ifxjdbc.jar:na]
	at com.gbasedbt.jdbc.IfxSqli.a(IfxSqli.java:3140) ~[ifxjdbc.jar:na]
	at com.gbasedbt.jdbc.IfxSqli.D(IfxSqli.java:3420) ~[ifxjdbc.jar:na]
	at com.gbasedbt.jdbc.IfxSqli.dispatchMsg(IfxSqli.java:2333) ~[ifxjdbc.jar:na]
	at com.gbasedbt.jdbc.IfxSqli.receiveMessage(IfxSqli.java:2258) ~[ifxjdbc.jar:na]
	at com.gbasedbt.jdbc.IfxSqli.a(IfxSqli.java:1557) ~[ifxjdbc.jar:na]
	at com.gbasedbt.jdbc.IfxSqli.executeStatementQuery(IfxSqli.java:1520) ~[ifxjdbc.jar:na]
	at com.gbasedbt.jdbc.IfxSqli.executeStatementQuery(IfxSqli.java:1468) ~[ifxjdbc.jar:na]
	at com.gbasedbt.jdbc.IfxResultSet.a(IfxResultSet.java:214) ~[ifxjdbc.jar:na]
	at com.gbasedbt.jdbc.IfxStatement.executeQueryImpl(IfxStatement.java:1064) ~[ifxjdbc.jar:na]
	at com.gbasedbt.jdbc.IfxPreparedStatement.executeQuery(IfxPreparedStatement.java:353) ~[ifxjdbc.jar:na]
	at com.zaxxer.hikari.pool.ProxyPreparedStatement.executeQuery(ProxyPreparedStatement.java:52) ~[HikariCP-3.4.2.jar:na]
	at com.zaxxer.hikari.pool.HikariProxyPreparedStatement.executeQuery(HikariProxyPreparedStatement.java) ~[HikariCP-3.4.2.jar:na]
	at org.springframework.jdbc.core.JdbcTemplate$1.doInPreparedStatement(JdbcTemplate.java:678) ~[spring-jdbc-5.2.4.RELEASE.jar:5.2.4.RELEASE]
	at org.springframework.jdbc.core.JdbcTemplate.execute(JdbcTemplate.java:617) ~[spring-jdbc-5.2.4.RELEASE.jar:5.2.4.RELEASE]
	... 69 common frames omitted
Caused by: java.sql.SQLException: null
	at com.gbasedbt.util.IfxErrMsg.getSQLException(IfxErrMsg.java:408) ~[ifxjdbc.jar:na]
	at com.gbasedbt.jdbc.IfxSqli.D(IfxSqli.java:3425) ~[ifxjdbc.jar:na]
	... 81 common frames omitted
```