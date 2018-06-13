---
title: JDBC基础
date: 2017-06-03 16:45:08
tags:
    - 数据库
    - Java
---

### JDBC的设计

JDBC并不是“Java Database Connectivity”的缩写，它是对ODBC的致敬（ODBC是微软创建的标准数据库API，并被合并了SQL标准中）。
JDBC提供“纯Java”的API，并提供一个**驱动管理器**以允许第三方驱动连接到指定的数据库。数据库供应商需要提供一个**驱动**插入到驱动管理器中。
编写的程序通过API与驱动管理器通信，而驱动管理器测通过驱动来与实际的数据库通信。 这样开发者只需要考虑JDBC的API。
<!-- more -->

JDBC标准把驱动器分类与4种类型：
- 第一类驱动把JDBC翻译成ODBC，并依赖ODBC驱动来与数据库通信。早期的Java包含一个这类驱动：J**DBC/ODBC桥**。不过Java8已经不再支持JDBC/ODBC桥，因为有更好的接口。
- 第二类驱动部分由Java部分和native代码写成，与数据库客户端API通信。使用这类驱动，必须安装平台特定的代码。
- 第三类驱动是纯Java客户端类库，它使用数据库无关的协议将数据库请求发给服务器组件，服务器组件将请求翻译成数据库相关的协议。这简化了部署，因为平台相关的代码只需要在服务端。
- 第四类驱动是纯Java库，将JDBC请求直接翻译成数据库相关协议。
如今大部分驱动为第三类或第四类。

### 连接数据库
#### URL
格式为`jdbc:subprotol:other stuff`。 `subprotol`选择了指定的驱动来连接数据库。`other`和`stuff`取决于数据库，具体查看相应数据库的文档。
这是一个mysql的URL示例：`jdbc.url=jdbc:mysql://127.0.0.1:33060/dbInContainer?autoReconnect=true&useSSL=false`
#### 导入驱动
IDE有添加Jar包的功能，Idea在**Project Settings**->**Modules**->**Dependencies**。 如果从命令行执行一个程序，使用`java -classpath driverPath:. ProgramName`。
#### 注册驱动类
大部分JDBC的Jar包会自动注册驱动类，这需要文件中有`META-INF/services/java.sql.Driver`。自动注册是遵循`JDBC4`的驱动所必须的。
如果Jar包不自动注册类，需要手动注册。可以找到JDBC驱动类名（如`org.postgresql.Driver`)然后在Java程序中加载`Class.forName("org.postgresql.Driver");`；或者可以设置`jdbc.drivers`属性，使用`java -Djdbc.drivers-org.postgresql.Driver ProgramName`或者`System.setProperty("jdbc.drivers","org.postgresql.Driver");`。
可以用`:`分隔来注册多个驱动类。
#### 连接数据库
```java
String url = "jdbc:postgresql:COREJAVA";
String username = "dbuser";
String password = "secret";
Connection conn = DriverManager.getConnection(url, username, password);
```
驱动管理器会遍历注册过的驱动来找到能使用url里指定的`subprotocol`的驱动。

### 使用JDBC语句
#### 执行SQL语句
要执行SQL语句，需要先创建`Statement`对象：`Statement stat = conn.createStatement;`，其中`conn`是`DriverManager.getConnection`返回的`Connection`对象。
然后拼接命令字符串，如`String command = "SELECT *  from  Books"`，执行`stat.executeQuery(command)`;
`executeQuery`用于执行SELECT语句，返回一个`ResultSet`类对象，可以遍历它来获取每一行。
还有个`executeUpdate`语句。`executeUpdate`方法返回被SQL影响的行数，或者对不返回行数的语句返回0。它能执行INSERT、UPDATE、DELETE，以及诸如CREATE  TABLE和DROP TABLE的数据定义语句。
```java
ResultSet rs = stat.executeQuery("SELECT * from Books");
while(rs.next()){
    //处理数据集
}
```
如果想获得单独一行的数据，ResultSet类提供了许多有效接口：
```java
// 返回第一列的数据
String isbn = rs.getString(1);
double price = rs.getDouble("Price");
```
**注意：**
与数组不同，数据库的列数从1开始。

#### Prepared Statements预备语句
每当数据库执行一个查询时，它总是先通过计算来确定查询策略，以便高效地执行查询操作。通过事先准备好查询并多次调用它，就可以确保准备步骤只执行一次。
如果两次查询的语句一样，只有部分变量不同，那么可以使用预备语句。
```Java
String str=
    "SELECT Books.Price"+
    "FROM Books,Publisher"+
    "WHERE Books.Publisher_ID=Publishers.Publisher_ID and Publisher.NAME=?";
PreparedStatement stat = conn.prepareStatement(str);
```
要设置参数时，只需要调用`stat.setString(1,publisher);`即可，其中参数1参数表示第一个`?`,第二个参数是要设置的值。
如果要重用已经执行过的预备查询语句，那么除非使用`set`方法或者调用`clearParameters`方法，否则已经设置过的变量不变。
执行方式不变，如`ResultSet rs = stat.executeQuery();`。
**注意：**
手工拼接字符串有潜在危险，你需要考虑特殊字符如`"`,如果使用了用户输入，那么就要防备注入攻击。因此，任何时候需要使用变量时都要使用预备语句。

#### SQL转义

#### 管理Connections、Statements、ResultSets
`Statement`对象可以执行多个不相关的SQL语句。但是一个statement一次只能打开一个`ResultSet`。也就是说，在并发的情况下，需要给一个Connections创建多个Statement实例。
需要注意的是，有一个常用的数据库Microsoft SQL Server有一个JDBC驱动，允许最多同时有一个statement。使用`DatabaseMetaData`接口的的`getMaxStatements`来查看数据库同一时间支持的最大statement数。
这看上去有局限性，但实际上，不应该为并发的ResultSets烦恼。如果结果集是相关联的，那么可以执行组合查询，这样就只需要分析一个结果。**由数据库组合查询远比Java程序遍历多个结果集高效**。
当使用Connections、Statements、ResultSets时，需要尽可能快的执行`close`方法。这个对象使用了大的数据结构，占用了有限的数据库服务器的资源。
`Statement`对象的close方法会关闭关联的结果集对象，同样`Connection`对象的close方法会关闭关联的Statement对象。
**注意：**
应该使用带资源的`try`方法来关闭连接，并使用一个单独的`try/catch`语句块了处理异常。分离`try`程序块可以提高代码的可阅读性和可维护性。


### SQL异常处理

### API
- `java.sql.Connection`
    - `Statement createStatement()` 创建一个`statement`对象用于执行SQL查询语句。
    - `void close()` 关闭当前连接以及它创建的JDBC资源。
- `java.sql.Statement`
    - `ResultSet executeQuery(String sqlQuery)` 执行SQL的`SELECT`语句，并返回一个`ResultSet`对象。
    - `int executeUpdate(String sqlQuery)`、`long executeLargeUpdate(String sqlStatement)` 执行SQL的INSERT，UPDATE或删除语句，或者执行DDL（Data Definition Language数据定义语句）如`CREATE TABLE`。返回被影响的函数或者0。
    - `boolean execute(String sqlStatement)` 执行SQL语句，可能会执行多条语句。
    - `ResultSet getResultSet()` 返回前一条语句的结果集，或者null。
    - `int getUpdateCount()`、`long getLargeUpdateCount()` 返回受前一条更新语句影响的函数。如果前一条语句未影响数据库，返回-1。
    - `void close()`、 `boolean is Closed()`
    - `void closeOnCompletion()` 如果该语句的所有结果集都被关闭，那么它也将被关闭。
- `java.sql.ResultSet`
    - `boolean next()` 遍历（将当前的行数前移1），如果不存在则返回false。注意必须先执行该语句来获得第一行。
    - `XXX getXXX(int columnNumber)`、`XXX getXXX(String columnLabel)`  `XXX`是个类型，如Int,double,String,Date等
    - `<T>  T getObject(int columnIndex, Class<T> type)`、`<T>  T getObject(String columnLabel, Class<T> type)`、`void updateObject(int columnIndex,Object x, SQLType targetSqlType)`、`void updateObject(String columnLabel,Object x, SQLType targetSqlType)`   返回会更新某一列的值，并转换成指定类型。
    - `int findColumn(String columnName)` 获取指定列数的列名
    - `void close()`、`boolean isClosed()`
  