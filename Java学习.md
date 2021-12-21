# Java 学习

高昊宇 

## 1 数据库 (MySQL)

### 1.1 数据库引擎

|              | MYSIAM                 | InnoDB             |
| ------------ | ---------------------- | ------------------ |
| 事务支持     | 不支持                 | 支持               |
| 锁机制       | 表级锁                 | 表、行级锁         |
| 主键         | 可以没有（非聚集索引） | 必须有（聚集索引） |
| 外键         | 不支持                 | 支持               |
| 全文索引     | 支持                   | 不支持 (5.7后支持) |
| 表空间的大小 | 较小                   | 较大               |

MYSIAM早期使用，InnoDB默认使用

常规使用：

- MYSIAM：节约空间，速度快
- InnoDB：安全性高，支持事务和多表操作

存储文件：

- MYSIAM：表定义文件frm，数据文件ibd
- InnoDB：表定义文件frm，数据文件myd，索引文件myi

CRUD操作 (增检更删)：

- MYSIAM：执行SELECT更快；保存了行号，没有条件的select count(*) from table可以直接输出
- InnoDB：有大量更新、增加操作时，因为支持行锁，可以大大增加并发度

### 1.2 JDBC

- SUN公司为简化开发人员对数据库的操作，提供了一个Java操作数据库的规范，规范的实现由数据库厂商实现。
- 即开发人员调用规范接口即可操作各类数据库。

#### 1.2.1 操作框架

```java
        // 1. 加载驱动
        Class.forName("com.mysql.jdbc.Driver");  // 加载驱动

        // 2. 用户信息和url
        String url = "jdbc:mysql://localhost:3306/jdbcTest?
            useUnicode=true&characterEncoding=utf-8&useSSL=true";
        String username = "root";
        String password = "001118";

        // 3. 建立连接
        Connection conn = DriverManager.getConnection(url, username, password);

        // 4. 创建SQL对象
        Statement statement = conn.createStatement();

        // 5. 执行SQL语句
        String sql = "SELECT * FROM users";
        ResultSet resultSet = statement.executeQuery(sql);

        while(resultSet.next()){
            System.out.println("id= " + resultSet.getObject("id"));
            System.out.println("name= " + resultSet.getObject("NAME"));
            System.out.println("pw= " + resultSet.getObject("PASSWORD"));
            System.out.println("Email= " + resultSet.getObject("EMAIL"));
            System.out.println("birthday= " + resultSet.getObject("BIRTHDAY"));
            System.out.println("=====================");
        }

        // 6. 断开数据库连接
        resultSet.close();
        statement.close();
        conn.close();
```



> DriverManager

```java
Class.forName("com.mysql.jdbc.Driver");  // 加载驱动
```



> URL

```java
String url = "jdbc:mysql://localhost:3306/jdbcTest?useUnicode=true&characterEncoding=utf-8&useSSL=true";

// url = "协议://主机:端口号/数据库名?参数1&参数2…";

// mqsql 端口号 3306
```



> 建立连接

```java
Connection conn = DriverManager.getConnection(url, username, password);
 
// connection对象代表数据库，执行所有数据库层面的操作
// 事务回滚: conn.rollback()
// 事务提交: conn.commit()
// 设置自动提交: conn.setAutoCommit()   // 默认开启，关闭后算是开启了事务（需要提交）
```



> 执行SQL

```java
Statement statement = conn.createStatement();
String sql = "SELECT * FROM users";
ResultSet resultSet = statement.executeQuery(sql);

statement.execute();        // 执行所有SQL
statement.executeQuery();   // 执行查询
statement.executeUpdate();  // 执行更新、插入、删除，返回受影响行数
```



> 获取数据

```java
System.out.println("id= " + resultSet.getObject("id"));

// 直到类型就使用指定的类型
resultSet.getString();
resultSet.getInt();
resultSet.getFloat();
resultSet.getDate();
resultSet.getObject();

// 光标移动
resultSet.next();         // 下一个
resultSet.previous();     // 上一个
resultSet.beforeFirst();  // 第一个
resultSet.afterNext();    // 最后一个
resultSet.row();          // 指定行
```



> 释放资源

```java
resultSet.close();
statement.close();
conn.close();
```



#### 1.2.2 工具类封装

```java
public class jdbcUtils {

    private static String driver = null;
    private static String url = null;
    private static String username = null;
    private static String password = null;

    static{
    
        try{
            // 从文件中读取驱动、url、用户名和密码
            InputStream in = jdbcUtils.class.getClassLoader().getResourceAsStream("db.properties");
            Properties properties = new Properties();
            properties.load(in);

            driver = properties.getProperty("driver");
            url = properties.getProperty("url");
            username = properties.getProperty("username");
            password = properties.getProperty("password");

            // 加载驱动
            Class.forName(driver);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // 获取连接
    public static Connection getConnection() throws SQLException {
        return DriverManager.getConnection(url, username, password);
    }

    // 释放连接
    public static void release(Connection conn, Statement st, ResultSet rt) throws SQLException {

        if(rt != null){
            rt.close();
        }

        if(st != null){
            st.close();
        }

        if(conn != null) {
            conn.close();
        }
    }
}
```



实际操作时，只需调用静态方法即可

```java
import testDB.utils.jdbcUtils;
import javax.xml.transform.Result;
import java.sql.*;

public class test {
    public static void main(String[] args) throws SQLException {

        // 1. 建立连接
        Connection conn  = jdbcUtils.getConnection();
        Statement st = conn.createStatement();

        // 2. 执行SQL语句
        String sql = "INSERT INTO users(id, NAME, PASSWORD, EMAIL, BIRTHDAY)" +
                "VALUES(4, 'gaohaoyu', '123654', '1363127651@qq.com', '2020-01-01')";
        int ans = st.executeUpdate(sql);
        if(ans > 0){
            System.out.println("插入成功！");
        }

        // 3. 断开数据库连接
        jdbcUtils.release(conn, st);
    }
}
```



#### 1.2.3 sql注入问题

sql语句存在漏洞，可能导致数据泄露。

本质：web应用对用户输入的合法性过滤不严，使得攻击者在事先定义好的sql语句结尾添加了额外的sql语句，实现了非法操作。

​	     （拼接字符串）

> 例

![image-20211119175801031](C:\Users\Salieri\Desktop\image-20211119175801031.png)

![image-20211119175836871](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211119175836871.png)

由于添加了恒成立的条件，所以username的搜索不生效，密码为123456的都会被查询到



#### 1.2.4 PreparedStatement对象

相比于Statement可以防止sql注入，效率也更高。（是Statement的子类）

```java
        // 1. 建立连接
        Connection conn  = jdbcUtils.getConnection();

        // 2. 执行SQL语句
        String sql = "INSERT INTO users(id, NAME, PASSWORD, EMAIL, BIRTHDAY)" +
                "VALUES(?, ?, ?, ?, ?)";      // 使用？占位符代替参数

        PreparedStatement st = conn.prepareStatement(sql); // 预编译sql，不执行

        // 手动赋值(下标从1开始)
        st.setInt(1,4);
        st.setString(2,"gaohaoyu");
        st.setString(3,"123654");
        st.setString(4,"1363127651@qq.com");
        st.setDate(5,new java.sql.Date(new Date().getTime()));

        int ans = st.executeUpdate();
        if(ans > 0){
            System.out.println("插入成功！");
        }

        // 3. 断开数据库连接
        jdbcUtils.release(conn, st);
```

防止sql注入的本质：将手动赋值输入的内容视为字符，再自动包一层‘’，而且当输入内容包含转义字符（如 ‘ ）时，直接忽略。



#### 1.2.5 事务

```java
try{
		// 1. 建立连接
		conn.setAutoCommit(false);  // 关闭自动提交，开启事务
		
	    // 2. 执行SQL语句
        String sql = "INSERT INTO users(id, NAME, PASSWORD, EMAIL, BIRTHDAY)" +
                "VALUES(?, ?, ?, ?, ?)";      // 使用？占位符代替参数

        PreparedStatement st = conn.prepareStatement(sql); // 预编译sql，不执行

        // 手动赋值(下标从1开始)
        st.setInt(1,4);
        st.setString(2,"gaohaoyu");
        st.setString(3,"123654");
        st.setString(4,"1363127651@qq.com");
        st.setDate(5,new java.sql.Date(new Date().getTime()));

        int ans = st.executeUpdate();
        
        // 3-1、提交事务
        conn.commit();  
        
} catch(SQLException e){
		try{
			conn.rollback();      // 3-2、提交事务
		} catch(SQLException e1){
			e1.printStackTrace();
		}
} finally{
	jdbcUtils.release(conn, st);  // 4、释放连接
}
```



#### 1.2.6 数据库连接池

频繁的连接、释放数据库十分浪费系统资源。

池化技术：准备一些预先的资源，过来就连接预先准备好的（先连接一些，完全不需要再释放）

最小连接数：满足需求必须要连多少

最大连接数：业务最高承载上限

业务超过最大连接数时，排队等待，若等待超时则直接取消业务

> 开源数据源

DBCP

C3P0

Druid（阿里）

（实现DataSource接口）



> DBCP

需要lib：commons-dbcp2-2.9.0.jar、commons-pool2-2.11.1.jar、commons-logging-1.2.jar

1、配置文件：dbcpconfig.properties

```
driverClassName=com.mysql.cj.jdbc.Driver
url=jdbc:mysql://localhost:3306/jdbcTest?useUnicode=true&characterEncoding=utf-8&useSSL=true
username=root
password=001118

#!-- 初始化连接 --
initialSize=10

#最大连接数量
maxActive=50

#!-- 最大空闲连接 --
maxIdle=20

#!-- 最小空闲连接 --
minIdle=5

#!-- 超时等待时间以毫秒为单位 6000毫秒/1000等于60秒 --
maxWait=60000
#JDBC驱动建立连接时附带的连接属性属性的格式必须为这样：【属性名=property;】
#注意：user 与 password 两个属性会被明确地传递，因此这里不需要包含他们。
connectionProperties=useUnicode=true;characterEncoding=UTF8

#指定由连接池所创建的连接的自动提交（auto-commit）状态。
defaultAutoCommit=true

#driver default 指定由连接池所创建的连接的只读（read-only）状态。
#如果没有设置该值，则“setReadOnly”方法将不被调用。（某些驱动并不支持只读模式，如：Informix）
defaultReadOnly=

#driver default 指定由连接池所创建的连接的事务级别（TransactionIsolation）。
#可用值为下列之一：（详情可见javadoc。）NONE,READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE
defaultTransactionIsolation=READ_UNCOMMITTED
```

2、工具类

```java
import org.apache.commons.dbcp2.BasicDataSourceFactory;
import javax.sql.DataSource;
import java.io.InputStream;
import java.sql.*;
import java.util.Properties;

public class jdbcUtils_dbcp {

    private static DataSource dataSource = null;
    static{

        try{
            InputStream in = jdbcUtils.class.getClassLoader().getResourceAsStream("dbcpconfig.properties");
            Properties properties = new Properties();
            properties.load(in);

            // 创建数据源 (工厂模式)
            dataSource = BasicDataSourceFactory.createDataSource(properties);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // 获取连接
    public static Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }

    // 释放连接
    public static void release(Connection conn, Statement st, ResultSet rt) throws SQLException {

        if(rt != null){
            rt.close();
        }

        if(st != null){
            st.close();
        }

        if(conn != null) {
            conn.close();
        }
    }
}
```

3、实际使用

```java
        // 1. 建立连接
        Connection conn  = jdbcUtils_dbcp.getConnection();

        // 2. 执行SQL语句
        String sql = "INSERT INTO users(id, NAME, PASSWORD, EMAIL, BIRTHDAY)" +
                "VALUES(?, ?, ?, ?, ?)";      // 使用？占位符代替参数

        PreparedStatement st = conn.prepareStatement(sql); // 预编译sql，不执行

        // 手动赋值(下标从1开始)
        st.setInt(1,4);
        st.setString(2,"gaohaoyu");
        st.setString(3,"123654");
        st.setString(4,"1363127651@qq.com");
        st.setDate(5,new java.sql.Date(new Date().getTime()));

        int ans = st.executeUpdate();
        if(ans > 0){
            System.out.println("插入成功！");
        }

        // 3. 断开数据库连接
        jdbcUtils_dbcp.release(conn, st);
```

实质：将连接管理交给了DBCP，使用时只进行连接的get和release



> C3P0

需要lib：c3p0-0.9.5.5.jar、mchange-commons-java-0.2.19.jar

1、配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<c3p0-config>
    <!--
    c3p0的缺省（默认）配置,
    如果在代码中ComboPooledDataSource ds=new ComboPooledDataSource();这样写就表示使用的是c3p0的缺省（默认）-->
    <default-config>
        <property name="driverClass">com.mysql.cj.jdbc.Driver</property>
        <property name="jdbcUrl">jdbc:mysql://localhost:3306/jdbctest?useUnicode=TRUE&amp;characterEncoding=utf-8&amp;useSSL=TRUE</property>
        <property name="user">root</property>
        <property name="password">001118</property>

        <property name="acquireIncrement">5</property>
        <property name="initialPoolSize">10</property>
        <property name="minPoolSize">5</property>
        <property name="maxPoolSize">20</property>
    </default-config>

    <!--
    自定义的MySQL配置
    如果在代码中ComboPooledDataSource ds=new ComboPooledDataSource("MySQL");这样写就表示使用该设置-->
    <named-config name="MySQL">
        <property name="driverClass">com.mysql.cj.jdbc.Driver</property>
        <property name="jdbcUrl">jdbc:mysql://localhost:3306/jdbctest?useUnicode=TRUE&amp;characterEncoding=utf-8&amp;useSSL=TRUE</property>
        <property name="user">root</property>
        <property name="password">001118</property>

        <property name="acquireIncrement">5</property>
        <property name="initialPoolSize">10</property>
        <property name="minPoolSize">5</property>
        <property name="maxPoolSize">20</property>
    </named-config>
</c3p0-config>

<!-- 可配多套数据源-->
<!-- xml创建后工具类会自动读取，无需指定该文件-->
```

2、工具类

```java
import com.mchange.v2.c3p0.ComboPooledDataSource;
import java.sql.*;

public class jdbcUtils_c3p0 {

    private static ComboPooledDataSource dataSource = null;
    static{

        try{
            // 创建数据源（两种写法）

            // 配置文件写法（自动读取xml文件）
            dataSource = new ComboPooledDataSource("MySQL");  // 通过参数指定数据源

            // 代码写法
//            dataSource = new ComboPooledDataSource();
//            dataSource.setDriverClass();
//            dataSource.setUser();
//            dataSource.setPassword();
//            dataSource.setJdbcUrl();
//
//            dataSource.setMaxPoolSize();
//            dataSource.setMinPoolSize();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    // 获取连接
    public static Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }

    // 释放连接
    public static void release(Connection conn, Statement st, ResultSet rt) throws SQLException {

        if(rt != null){
            rt.close();
        }

        if(st != null){
            st.close();
        }

        if(conn != null) {
            conn.close();
        }
    }
}
```

3、实际使用

同DBCP，只是使用不同的工具类



## 2 反射

java的反射机制：动态地获取对象信息，动态地调用对象方法

Java作为动态语言的关键，允许程序在执行时借助反射API获得任何类的内部信息，并能直接操作人以对象的内部属性和方法

（自我理解：拿到一个对象，可以反方向的获取其对应类名，构造方法，属性和其他方法，并进行调用）



### 2.1 主要功能

- 在运行时判断任意一个对象所属的类。
- 在运行时构造任意一个类的对象。
- 在运行时判断任意一个类所具有的成员变量和方法。
- 在运行时调用任意一个对象的方法。
- 生成动态代理（见 5.8.2 Spring动态代理）。



### 2.2 实现反射的类

java.lang.reflect包中

- Class类：代表一个类；
- Constructor类：代表类的构造方法；
- Field类：代表类的成员变量（也称为类的属性）；
- Method类：代表类的方法；
- Array类：提供了动态创建数组，以及访问数组的元素的静态方法。



### 2.3 相关函数

自定义copy方法，用来创建一个和参数objcet同样类型的对象，然后把object对象中的所有属性拷贝到新建的对象中，并将其返回。

```java
import java.lang.reflect.Field;
import java.lang.reflect.Method;

public class ReflectTester {

    public Object copy(Object object) throws Exception{
        
        //获得对象的类型
        Class classType=object.getClass();
        System.out.println("Class:"+classType.getName());

        //通过默认构造方法创建一个新的对象
        Object objectCopy=classType.getConstructor(new Class[]{}).newInstance(new Object[]{});

        //获得对象的所有属性
        Field fields[]=classType.getDeclaredFields();

        for(int i=0; i<fields.length;i++){
              Field field=fields[i];

              String fieldName=field.getName();
              String firstLetter=fieldName.substring(0,1).toUpperCase();
            
              //获得和属性对应的getXXX()和setXXX()方法的名字
              String getMethodName="get"+firstLetter+fieldName.substring(1);
              String setMethodName="set"+firstLetter+fieldName.substring(1);

              //获得和属性对应的getXXX()和setXXX()方法
              Method getMethod=classType.getMethod(getMethodName,new Class[]{});
              Method setMethod=classType.getMethod(setMethodName,new Class[]{field.getType()});

              //调用原对象的getXXX()方法
              Object value=getMethod.invoke(object,new Object[]{});
              System.out.println(fieldName+":"+value);
            
              //调用拷贝对象的setXXX()方法
             setMethod.invoke(objectCopy,new Object[]{value});
        }
        return objectCopy;
     }
}
```



- getClass()：方法获取对象的类。
- getName()：获得完整名字（包括其所在的包名；类，属性，方法都可用本方法得到名字）。 
- getSimpleName()：获得名字（类，属性，方法都可用本方法得到名字）。 
- Class.forName(String name)：获得参数name指定的类
- getClassLoader()：获取实体的类加载器ClassLoader
- getInterfaces()：获得类实现的所有接口



- getConstrutors()：获得类的public类型的构造方法。
- getConstrutor(Class[] parameterTypes)：获得类的特定构造方法，parameterTypes参数指定构造方法的参数类型。
- newInstance()：通过类的无参构造方法创建这个类的一个对象。
  - 类的class.newInstance()调用无参构造函数
  - 构造器的construtor.newInstance()调用该构造器对应的构造函数，可以有参



- getFields()：获得类的public类型的属性。
- getDeclaredFields()：获得类的所有属性。
- field.getType()：获得属性的数据类型
- field.set(Object object, Object parameter)：设置object对象的field属性值为parameter
  - 设置private属性时，需要先field.setAccessible(true)关闭程序的安全检测



- getMethods()：获得类的public类型的方法。

- getDeclaredMethods()：获得类的所有方法。

- getMethod(String name, Class[] parameterTypes)：获得类的特定方法，name指定方法名，parameterTypes指定方法参数。

- method.invoke(Object object, Object[] parameters)：执行object的method方法，参数为parameters

  

### 2.4 类加载器 ClassLoader

类加载器将class字节码加载到堆内存，将之由静态数据转为方法区的运行时数据结构，堆的Class对象是方法区该类数据的访问入口

<img src="https://api2.mubu.com/v3/document_image/1f91395c-1148-4065-8e5e-07653cd6898d-1933247.jpg" alt="img" style="zoom: 67%;" />

类加载器一般有三层：

- 最底层为引导类加载器Bootstap ClassLoader，负责Java核心库

- 中间层为扩展类加载器Extension ClassLoader，负责jre指定目录下的包

- 上层为系统类加载器System ClassLoader，负责其他包

- 之上还可以自定义类加载器

加载类时从底向上调用类加载器，检查类是否被加载时从上向下依次检查



类缓存：类被加载后会在堆内存维持一段时间，后由JVM垃圾回收机制回收这些Class对象



## 3 JavaWeb

在java中，动态web资源开发的技术统称为JavaWeb

技术栈：Servlet/JSP, A



### 3.1基本知识

**静态Web**

- 页面无法动态更新，只能用JavaScript、VBScript实现伪动态
- 无法和数据库交互



**动态Web**

- ASP
  - 微软的，在HTML中嵌入VB脚本
  - 维护成本高
- C#
- llS
- PHP
  - 开发速度快，功能强大，跨平台，代码简单
  - 无法承载大访问量（局限）
- JSP/Servlet
  - B/S和C/S
  - 基于Java
  - 可以承载三高问题（高负载，高并发，高访问量）



**Web服务器**

服务器是一种被动的操作，用来处理用户的请求，给用户相应信息

- llS：微软，ASP，windows自带
- Tomcat：基于Java



**请求方式**

- get：可携带的参数少，大小有限制，会在浏览器地址栏显示数据内容，不安全，高效
- post：可携带的参数、大小无限制，不会在浏览器地址栏显示数据内容，安全，不高效



响应状态码

200：请求响应成功

3xx：请求重定向，302重定向，307请求转发

4xx：资源 不存在，如404

5xx：服务器代码错误，502网关错误



### 3.2 Tomcat

打开、关闭：bin文件夹下startup.bat、shutdown.bat

服务器核心配置：conf文件夹下server.xml

> 主机端口号

```xml
<Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443" />
```

tomcat：8080

mysql：3306

http：80

https：443



> 主机名称

```xml
<Host name="localhost"  appBase="webapps"
        unpackWARs="true" autoDeploy="true">
```

webapps下一个文件夹为一个web应用



> **问题：网站是如何被访问的？**

1、浏览器输入网址

2、在本机host配置文件中有没有该域名映射

- 有，直接访问该ip地址
- 没有，查找DNS

3、查找DNS

- 先找浏览器DNS缓存、系统DNS缓存
- 没有则再向DNS服务器查询

4、访问该ip的服务器，服务器返回网站内容



> **发布一个Web网站**

将自己写的网站，放到服务器（Tomcat）中web应用指定的文件夹（webapps）下

打开服务器，输入网址：http://主机:端口号/web应用名



> **在浏览器中配置Tomcat**

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211120210206338.png" alt="image-20211120210206338" style="zoom: 67%;" />

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211120210508424.png" alt="image-20211120210508424" style="zoom:67%;" />

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211120210630911.png" alt="image-20211120210630911" style="zoom:67%;" />

Application context为虚拟路径映射，是网址的后缀，localhost:8080/后缀



<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211120211505722.png" alt="image-20211120211505722" style="zoom: 67%;" />

maven的命令行操作

### 3.3 Maven

自动导入和配置jar包--Maven项目架构管理工具

**核心思想：约定大于配置**

按Maven的规定编写java代码

#### 3.3.1 安装与配置

> 环境变量

MAVEN_HOME: D:\coding\apache-maven-3.8.3

M2_HOME: %MAVEN_HOME%\bin

Path中加上和M2_HOME一样的内容



> 阿里云镜像（settings.xml）

```xml
<mirror>
    <id>nexus-aliyun</id>
    <mirrorOf>*,!jeecg,!jeecg-snapshots</mirrorOf>
    <name>Nexus aliyun</name>
    <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
</mirror>
```



> 本地仓库

```xml
<localRepository>D:\coding\apache-maven-3.8.3\maven-repo</localRepository>
```



#### 3.3.2 建工程

![image-20211120203319366](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211120203319366.png)

GroupId: 组id，为公司域名

Artifact: 项目名



<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211120204743586.png" alt="image-20211120204743586" style="zoom:80%;" />

main目录下的java保存源码，resources放置配置文件；test目录下的java用于测试

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211120205039143.png" alt="image-20211120205039143" style="zoom:80%;" />

webapp模版的工程，index.js为网页，web.xml为网页的配置。（需要手动创建非模板工程下的内容）



> POM文件

pom.xml时Maven的核心配置文件



<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211120213009396.png" alt="image-20211120213009396" style="zoom:67%;" />

maven的高级之处在于能够自动导入项目依赖中的jar包所依赖的其他jar包



> 建议

1、将web.xml中webapp的版本调成和tomcat一致，可以直接复制tomcat的

![image-20211120215457155](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211120215457155.png)



#### 3.3.3 maven仓库

https://mvnrepository.com/



#### 3.3.4 资源导出问题

```xml
<!--    在Maven项目中，资源配置文件默认是放在resources目录下的。-->
<!--    但有时我们在编写项目时，配置文件可能会被我们放置的别的目录，-->
<!--    Maven由于它的约定大于配置，所以默认的maven项目在构建编译时不会把我们其他目录下的配置文件导出到target目录中，从而导致配置文件无法导出或者生效的问题。-->
<!--    解决方案：在项目的pom.xml文件中手动配置资源过滤，让它把src/main/java目录下的.properties和.xml文件也能够被导出。-->
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <includes>
                <include>**/*.properties</include>
                <include>**/*.xml</include>
            </includes>
            <filtering>true</filtering>
        </resource>
        <resource>
            <directory>src/main/java</directory>
            <includes>
                <include>**/*.properties</include>
                <include>**/*.xml</include>
            </includes>
            <filtering>true</filtering>
        </resource>
    </resources>
</build>
```



### 3.4 Servlet

SUN公司开发动态web的一门技术，其提供了一个Servlet接口，有两个实现类：HttpServlet

开发Servlet程序：

- 编写类实现Servlet接口
- 吧开发好的java类部署到web服务器

实现了Servlet接口的java程序叫做Servlet



#### 3.4.1 HelloServlet

1、创建一个普通的Maven项目，删除src目录，获得Maven的主工程

2、从https://mvnrepository.com/导入依赖（依赖报红可以先手动要求安装，安装成功后刷新）

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211121104441283.png" alt="image-20211121104441283" style="zoom: 67%;" />

3、在该项目中新建一个module（子Maven项目）

4、Maven环境优化：将web.xml版本换成最新的，补全子项目的结构（java，resources）

​	父项目中有：

```xml
<modules>
    <module>servlet-01</module>
</modules>
```

​	子项目中有：

```xml
<parent>
    <artifactId>servlet01</artifactId>
    <groupId>com.ghy</groupId>
    <version>1.0-SNAPSHOT</version>
</parent>
```

​	父项目中的java子项目可以直接用

5、编写servlet程序

在java目录下创建一个普通类，继承HttpServlet，实现Servlet接口

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211121111620331.png" alt="image-20211121111620331" style="zoom: 80%;" />

```java
public class HelloServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        // ServletOutputStream outputStream = resp.getOutputStream();
        PrintWriter writer = resp.getWriter();  // 响应流
        writer.print("Hello, Servlet");
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        doGet(req, resp);
    }
}
```

6、编写Servlet的映射

我们写的是Java程序，但如果要通过浏览器访问，因为浏览器要连接web服务器，所以我们需要在web服务中注册我们写的Servlet，并给他一个浏览器能访问的路径。

```xml
web.xml
<!--注册Servlet-->
<servlet>
    <servlet-name>hello</servlet-name>
    <servlet-class>com.ghy.servlet.HelloServlet</servlet-class>
</servlet>

<!--Servlet请求路径-->
<servlet-mapping>
    <servlet-name>hello</servlet-name>
    <url-pattern>/hello</url-pattern>
</servlet-mapping>
```



7、配置Tomcat

注意要配置项目发布的路径



#### 3.4.2 Servelet原理

本质：Servlet是一个java应用程序，运行在服务器端，用来处理客户端请求（http请求）并作出响应的程序

工作步骤：

1. Web Client 向Servlet容器（Tomcat）发出Http请求
2. Servlet容器接收Web Client的请求
3. Servlet容器创建一个HttpRequest对象，将Web Client请求的信息封装到这个对象中。
4. Servlet容器创建一个HttpResponse对象
5. Servlet容器调用HttpServlet对象的service方法，把HttpRequest对象与HttpResponse对象作为参数传给HttpServlet 对象。
6. HttpServlet调用HttpRequest对象的有关方法，获取Http请求信息。
7. HttpServlet调用HttpResponse对象的有关方法，生成响应数据。
8. Servlet容器把HttpServlet的响应结果传给Web Client。



#### 3.4.3 寻址问题（Mappiing）

1、一个Servlet请求对应一个映射路径

```xml
<servlet-mapping>
    <servlet-name>hello</servlet-name>
    <url-pattern>/hello</url-pattern>
</servlet-mapping>
```

2、一个Servlet请求对应多个映射路径

只要<servlet-name>一样，不同的映射路径就会对应同一个Servlet请求

```xml
<servlet-mapping>
    <servlet-name>hello</servlet-name>
    <url-pattern>/hello1</url-pattern>
</servlet-mapping>
<servlet-mapping>
    <servlet-name>hello</servlet-name>
    <url-pattern>/hello2</url-pattern>    <!--单纯重复-->
</servlet-mapping>
<servlet-mapping>
    <servlet-name>hello</servlet-name>
    <url-pattern>/hello/*</url-pattern>   <!--对应所有符合要求的-->
</servlet-mapping>
<servlet-mapping>
    <servlet-name>hello</servlet-name>
    <url-pattern>*.ghy</url-pattern>  <!--自定义后缀，*前不能有任何字符-->
</servlet-mapping>
```

匹配时优先匹配固定的，匹配不到再匹配带*的

#### 3.4.4 Servlet Context

web容器启动的时候，为每个web程序都创建一个对应的Servlet Context对象，代表了当前的web应用。 

##### 1、共享数据

+ 一个web应用只有一个Servlet Context，其不管有几个Servlet都对应它一个对象，因此各Servlet实现类可以用它实现数据共享

+ 保存数据

  ```java
  public class HelloServlet extends HttpServlet {
      @Override
      protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
          ServletContext context = this.getServletContext();
          String username = "ghy";
          context.setAttribute("username", username);  // 保存为一个键值对
      }
  }
  ```

+ 读取数据

  ```java
  public class GetServlet extends HttpServlet {
      @Override
      protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
          ServletContext context = this.getServletContext();
          String username = (String) context.getAttribute("username");
          
          resp.setContentType("text/html");
          resp.setCharacterEncoding("utf-8");
          resp.getWriter().print("用户名：" + username);
      }
  }
  ```

+ web.xml

  ```xml
  <servlet>
      <servlet-name>hello</servlet-name>
      <servlet-class>com.ghy.servlet.HelloServlet</servlet-class>   <!--保存-->
  </servlet>
  <servlet-mapping>
      <servlet-name>hello</servlet-name>
      <url-pattern>/hello</url-pattern>
  </servlet-mapping>
  
  <servlet>
      <servlet-name>getc</servlet-name>
      <servlet-class>com.ghy.servlet.GetServlet</servlet-class>     <!--读取-->
  </servlet>
  <servlet-mapping>
      <servlet-name>getc</servlet-name>
      <url-pattern>/getc</url-pattern>
  </servlet-mapping>
  ```

##### 2、初始化参数

+ 在web.xml中配置web应用的初始化参数

  ```xml
  <context-param>
      <param-name>url</param-name>
      <param-value>jdbc:mysql://localhost:3306/mybatis</param-value>
  </context-param>
  ```

+ 之后可在Servlet实现类中读取

  ```java
  public class GetServlet extends HttpServlet {
      @Override
      protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
          ServletContext context = this.getServletContext();
          String url = context.getInitParameter("url");
          resp.getWriter().print(url);
      }
  }
  ```

##### 3、请求转发

```java
public class ServletDemo01 extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        ServletContext context = this.getServletContext();
        context.getRequestDispatcher("/hello").forward(req, resp);  // 简化写法
    }
}
```

```xml
<servlet>
    <servlet-name>hello</servlet-name>
    <servlet-class>com.ghy.servlet.HelloServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>hello</servlet-name>
    <url-pattern>/hello</url-pattern>
</servlet-mapping>

<servlet>
    <servlet-name>sd1</servlet-name>
    <servlet-class>com.ghy.servlet.ServletDemo01</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>sd1</servlet-name>
    <url-pattern>/sd1</url-pattern>
</servlet-mapping>
```

访问sd1的地址时，会调用名为“ServletDemo01”的Servlet，根据其java代码进行请求转发。

转去调用“/hello”地址对应的Servlet。

注：地址栏地址不变



##### 4、读取资源文件

模块中，src下的java、resources中的资源文件会被打包到同一路径下：

target/servlet-版本号-SNAPSHOT/WEB-INF/classes，将之称为classpath

方法：用文件流读取导出的资源文件（相对地址）

```java
public class ServletDemo extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        InputStream in = this.getServletContext().getResourceAsStream("/WEB-INF/classes/db.properties");// 资源文件路径
        Properties prop = new Properties();
        prop.load(in);
        String user = prop.getProperty("username");   // 获取指定资源
        String password = prop.getProperty("password");
        resp.getWriter().print(user + ":" + password);
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        doGet(req, resp);
    }
}
```



##### 补充

1、资源不在resource文件夹下可能无法导出

解决：在模块的pom.xml指定配置

![image-20211122175840859](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211122175840859.png)

#### 3.4.5 HttpServletResponse

web服务器接收到用户http请求后，创建对应的:

HttpServletResponse对象：负责给用户响应

HttpServletRequest对象：负责获取用户http请求的信息



HttpServletResponse定义了各种浏览器状态(Status)：

200：成功返回

3XX：重定向

4XX：浏览器错误

5XX：服务器错误

##### 1、HttpServletResponse方法分类

> 向浏览器发送数据

```java
ServletOutputStream getOutputStream() throws IOException;
PrintWriter getWriter() throws IOException;
```

> 向浏览器发出响应头

```java
ServletResponse.java
void setCharacterEncoding(String var1);
void setContentLength(int var1);
void setContentLengthLong(long var1);
void setContentType(String var1);

HttpServletResponse.java
void setDateHeader(String var1, long var2);
void addDateHeader(String var1, long var2);
void setHeader(String var1, String var2);
void addHeader(String var1, String var2);
void setIntHeader(String var1, int var2);
void addIntHeader(String var1, int var2);
```

##### 2、常见应用

- 向浏览器输出消息
- 下载文件
  - 获取下载文件的路径
  - 下载文件的文件名
  - 让浏览器支持下载该文件
  - 获取下载文件的输入流
  - 创建缓冲区
  - 获取OutputStream对象
  - 将FileOutputStream流写入buffer缓冲区
  - 使用OutputStream将缓冲区的数据输出到客户端

```java
// 获取下载文件的路径
String realPath = this.getServletContext().getRealPath("/1.jpg");
System.out.print("下载文件的路径：" + realPath);
// 下载文件的文件名
String fileName = realPath.substring(realPath.lastIndexOf("\\") + 1);
// 让浏览器支持下载该文件
resp.setHeader("Content-Disposition","attachment:filename=" + fileName);
// 获取下载文件的输入流
FileInputStream in = new FileInputStream(realPath);
// 创建缓冲区
int len = 0;
byte[] buffer = new byte[1024];
// 获取OutputStream对象
ServletOutputStream out = resp.getOutputStream();
// 将FileOutputStream流写入buffer缓冲区,使用OutputStream将缓冲区的数据输出到客户端
while((len=in.read(buffer)) != -1){
    out.write(buffer,0, len);
}

in.close();
out.close();
```

- 验证码（图一乐）

```java
public class ImageServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

        // 让浏览器每3秒刷洗一次
        resp.setHeader("refresh","3");

        // 在内存中创建一个图片
        BufferedImage image = new BufferedImage(80, 20, BufferedImage.TYPE_INT_RGB);

        // 得到图片
        Graphics2D g = (Graphics2D)image.getGraphics();

        // 设置图片背景颜色
        g.setColor(Color.white);
        g.fillRect(0 ,0, 80, 20);

        // 给图片写数据
        g.setColor(Color.BLUE);
        g.setFont(new Font(null,Font.BOLD,20));
        g.drawString(makeNum(),0,20);

        // 告诉浏览器，此请求用图片方式打开
        resp.setContentType("image/jpeg");

        // 不让浏览器缓存
        resp.setDateHeader("expires", -1);
        resp.setHeader("Cache-Control","no-cache"); // 不缓存，不占浏览器资源
        resp.setHeader("Pragma","no-cache");

        // 把图片写给浏览器
        ImageIO.write(image,"jpg", resp.getOutputStream());
    }

    // 生成验证码
    private String makeNum(){
        Random random = new Random();
        String num = random.nextInt(9999999) + "";  // 7位
        StringBuffer sb = new StringBuffer();
        for(int i = 0; i < 7 - num.length(); i++){
            sb.append("0");
        }
        return sb.toString() + num;
    }
}
```



- **实现重定向（重要）**

一个web资源收到客户端请求后，会通知客户端去访问另一个web资源（如用户登录）

```java
public class RedirectServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
//        resp.setHeader("Location", "/r/img");
//        resp.setStatus(HttpServletResponse.SC_MOVED_TEMPORARILY);  // 等价于常数302
        resp.sendRedirect("/response1_war/img");  // 要有项目名（Tomcat里设置）
    }
}
```

地址栏中的地址会变成重定向后的地址

问题：重定向和转发

- 相同点：页面都会跳转
- 不同点：重定向后地址栏会变成重定向后的地址，转发则不变

> 应用：模拟用户登录界面

index.jsp (登录页)

```jsp
<html>
<body>

<%--action为提交的路径，需要寻找该项目的路径--%>
<form action="${pageContext.request.contextPath}/login" method="get">
    用户名：<input type="text" name="username" ><br>
    密码：<input type="password" name="password" ><br>
    <input type="submit">
</form>

</body>
</html>
```

success.jsp（登录后页面）

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<body>
<h2>Login Success</h2>
</body>
</html>
```

ResponseTest.java

```java
public class ResponseTest extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        String username = req.getParameter("username");
        String password = req.getParameter("username");
        System.out.println(username + ":" + password);

        resp.sendRedirect("/response1_war/success.jsp");

    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        doGet(req, resp);
    }
}
```

模块web.xml

```xml
<!--用户登录-->
<servlet>
  <servlet-name>login</servlet-name>
  <servlet-class>com.ghy.servlet.ResponseTest</servlet-class>
</servlet>
<servlet-mapping>
  <servlet-name>login</servlet-name>
  <url-pattern>/login</url-pattern>
</servlet-mapping>
```

不手动输入login地址，而且登录默认的index主页；

submit主页的form后，根据该form的action，提交到该项目的/login;

根据web.xml的设置调用名为“ResponseTest”的Servlet；

ResponseTest进行重定向，跳转到success.jsp页面。

#### 3.4.6 HttpServletRequest

HttpServletRequest代表客户端的请求，用户通过Http协议访问服务器，Http请求中的所有信息会被封装到其中，各种用户请求信息都可以用其get方法获取

```java
req.getContextPath();  // 返回Tomcat中设置的，该模块的地址
```

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211124202211048.png" alt="image-20211124202211048" style="zoom:67%;" />

1、获取前端传递的参数，请求转发

![image-20211124200230360](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211124200230360.png)

```java
req.getRequestDispatcher("/index.jsp").forward(req, resp);
// 这里的“/”代表当前应用，包括Tomcat中设置的地址段，从之后开始即可
```

解决接受参数，响应信息的中文乱码

```java
req.setCharacterEncoding("utf-8");
resp.setCharacterEncoding("utf-8");
```



### 3.5 Cookie、Session

会话：用户打开浏览器，访问一些web资源，关闭浏览器

![image-20211124212952980](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211124212952980.png)

服务端怎么知道客户端来没来过（保存会话）

- 服务端给客户端一个信件，客户端下次访问时带上：cookie
- 服务器登记该客户端已来过，下次有客户端访问时进行匹配：session

应用：首次登录网站后，之后自动登录



cookie：客户端技术（响应、请求）

session：服务器技术，将信息和数据放入session中 



#### 3.5.1 Cookie

服务器从客户端的请求中获取其Cookie，并通过响应给客户端新的Cookie

```java
Cookie[] cookies = req.getCookies();  // 获取Cookie
cookie.getName(); // 获得一个Cookie的Key
cookie.getValue();  // 获得一个Cookie的Value

Cookie cookie = new Cookie("lastLoginTime", System.currentTimeMillis() + ""); // 新建Cookie
resp.addCookie(cookie);  // 响应给客户端Cookie
```

```java
// 一般可解决中文乱码
req.setCharacterEncoding("utf-8");
resp.setCharacterEncoding("utf-8");  

// 或在赋值时转码，
Cookie cookie = new Cookie("name", URLEncoder.encode("高昊宇","utf-8"); 
URLDecoder.decode(cookie.getValue(), "utf-8");
```

Cookie一般保存在本地的用户目录下的appdata中；

细节

- 一个Cookie一般只保存一个信息；
- 1个web站点可以给浏览器发送多个Cookie，最多存放20个；
- Cookie大小有限制4kb；
- 浏览器有Cookie上限，300个
- 删除Cookie

  - 不设置有效期，关闭浏览器则自动失效
  - 手动设置有效期为0




#### 3.5.2 Session (重点）

一个服务器会给每个浏览器创建一个Session对象，一个Session对应一个浏览器直至该浏览器关闭。

应用：用户登录后，保存一个用户的信息，该网站的所有网页都不用再登录；购物车；保存网站中的常用数据，避免频繁取数据

```java
String getId();  // 获得session的id
ServletContext getServletContext();
Object getAttribute(String var1); // 获得session属性
void setAttribute(String var1, Object var2); // 设置session属性
void removeAttribute(String var1); // 删除session属性
void invalidate(); // 注销session，注销后会马上自动生成新的session
boolean isNew();  // 判断得session是不是新创建的
```

- 服务器创建session时，创建了内容为sessionId的Cookie，并响应给了客户端。客户端访问时，服务器通过获取名为“sessionId”的Cookie得到该客户端Session的id，进而找到服务器中对应的Session（可以说，Session基于Cookie）
- 因为一个session一个浏览器通用，所以可用setAttribute、getAttribute实现不同servlet的数据共享

在模块的web.xml可以对Session进行设置

```xml
<session-config>
    <!--Session失效时间，单位为分钟-->
    <session-timeout>15</session-timeout>
</session-config>
```



> Session和Cookie

Cookie把用户的数据写给用户的浏览器，浏览器保存

Session把用户的数据写到用户对应的Session中，服务器端报错

Session对象由服务器创建



> Session和ServletContext

一个Session对应一个客户端，一个ServletContext对应一个服务器。

Session实现一个客户端访问不同Servlet时的数据共享；

ServletContext可以实现不同客户端访问同一服务器时的数据共享



### 3.6 JSP

Java Server Pages：Java服务器端页面，和Servlet一样，用于开发动态web

特点：写JSP就像在写HTML（不同于HTML，可以嵌入Java代码，给用户提供动态数据）

```
// IDEA中tomcat的工作空间
C:\Users\Salieri\AppData\Local\JetBrains\IntelliJIdea2021.2\tomcat

// JSP位置
C:\Users\Salieri\AppData\Local\JetBrains\IntelliJIdea2021.2\tomcat\b095a9ce-9fd8-44be-bac4-39c24a6ae224\work\Catalina\localhost\ROOT\org\apache\jsp
可在次页面看到jsp页面的java程序
```



#### 3.6.1 JSP 工作原理

浏览器向服务器发送请求，不论申请什么资源，都是在访问Servlet

**JSP的本质就是一个Servlet**

jsp文件生成的java文件中，实现的org.apache.jasper.runtime.HttpJspBase类：

1、判断请求

```java
public void _jspInit() {
}

public void _jspDestroy() {
}

public void _jspService(final javax.servlet.http.HttpServletRequest request, final javax.servlet.http.HttpServletResponse response)
```

2、内置一些对象

```java
final javax.servlet.jsp.PageContext pageContext; // 页面上下文
javax.servlet.http.HttpSession session = null;  // session
final javax.servlet.ServletContext application; // applicationContext
final javax.servlet.ServletConfig config; // config
javax.servlet.jsp.JspWriter out = null; // out
final java.lang.Object page = this; // page:当前
javax.servlet.jsp.JspWriter _jspx_out = null;  // 请求
javax.servlet.jsp.PageContext _jspx_page_context = null;  // 响应
```

3、输出页面前增加的代码

```JAVA
response.setContentType("text/html");  // 设置响应的页面类型
pageContext = _jspxFactory.getPageContext(this, request, response,
                                          null, true, 8192, true);
_jspx_page_context = pageContext;
application = pageContext.getServletContext();
config = pageContext.getServletConfig();
session = pageContext.getSession();
out = pageContext.getOut();
_jspx_out = out;
```

实现了：

- 在jsp文件中直接使用内置的对象，${变量名}
- 在jsp中可在<%%>中写java代码，<%=var%>使用java代码中定义的名为var的变量

4、输出页面

```java
out.write("<html>\n");
out.write("<body>\n");
out.write("<h2>Hello World!</h2>\n");
out.write("</body>\n");
out.write("</html>\n");
```

不同的jsp文件生成的java文件，只有输出的页面不一样

jsp中的java代码不变，html代码被放入out.write()中，<%=var%>将会转为out.print(var)进行输出

**实际的访问过程**

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211125182615456.png" alt="image-20211125182615456" style="zoom:80%;" />

- 客户端访问服务器，服务器找到对应的jsp
- jsp文件转化为java文件，编译为class字节码文件，传回服务器
- 服务器运行class字节码文件，将执行结果响应给客户端

不难发现，只用servlet也能实现，不过用了jsp后，我们就可以方便的使用html了，网页中动态内容的生成还由我们的java代码实现，但所有内容的输出交给了jsp负责，自动转化成了包含需要的设置的java文件。（比直接访问servlet只多了jsp->java一步）



#### 3.6.2 JSP 基本语法

**JSP表达式**

```jsp
<%--JSP表达式
    作用：输出程序内容到客户端
    格式：<%= 变量或表达式%>
--%>
<%= new java.util.Date()%>
```

**JSP脚本片段**

```jsp
<%--JSP脚本片段--%>
<%
	int sum = 0;
	out.println("<h1> sum:" + sum + "</h1>");
%>
```

因为转化后是一个java文件，所以各个脚本片段实际上是在同一作用域下，同名变量和函数不能重复定义。

将大括号的两端放在两个片段中，以将html包裹（图一乐）

```jsp
<%
   for(int i = 0; i < 10; i++){
%>
   <h1>hello,<%=i%></h1>
<%
    }
%>
```

**JSP声明**

```jsp
<%--JSP声明--%>
<%!
    static{
    int a = 0;
	}
	private int b = 0;
	static int c = 0;
%>
```

一般jsp的内容会放入生成的java文件的jspService方法，JSP声明的内容则被放在生成的java文件的类中，成为和jspService同级的类成员

**其他**

JSP注释<%-- XXX --%>不会显示到页面上；

HTML注释<!-- XXX -->会显示到页面上；



#### 3.6.3 JSP 指令

**1、定义错误页面**

先手动完成错误处理的jsp页面，最好将这些错误处理页面放到一个目录下。

方法1：在可能出错的jsp文件中设置出错时转向的jsp

```jsp
<%@ page errorPage="index.jsp" %>
```

方法2：在web.xml设置服务器，设置出现各错误代码的错误时分别跳转到哪个jsp

```xml
<error-page>
    <error-code>404</error-code>
    <location>/error/404.jsp</location>
</error-page>
```

**2、设置页面格式**

一般新建jsp文件自动生成

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
```

**3、提取公共页面**

导入其他jsp文件的内容

```jsp
<%@include file="index.jsp"%>
```

还可以用jsp标签实现：

```jsp
<jsp:include page="/index.jsp">
```

注：

- 相对地址格式略微不同

- 生成到java文件格式不同

  - @include会直接把内容导入，使之和其他内容没有区别

  - jsp标签会被转化成相关的include指令，并在类下的静态代码块中将文件add进来，本质上还是多个文件

    - ![image-20211126095643115](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211126095643115.png)

    - 

      ![image-20211126095611832](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211126095611832.png)

- 定义变量限制不同

  - @include因为变成了一个文件，两个jsp中不能定义同名变量，否则运行报500错误
  - jsp标签本质上还是三个文件，没有限制



#### 3.6.4 JSP 内置对象

- PageContext：PageContext.foward()可以请求转发
- Request
- Response
- Session
- Application (就是ServletContext，名称不同而已)
- config
- out
- page
- exception

**保存信息**

```jsp
<%
	pageContext.setAttribute("name", "val");   // 在当前页面有效
	request.setAttribute("name", "val");       // 在一次请求中有效，请求转发会携带此数据
	session.setAttribute("name", "val");       // 在一次会话中有效，从打开浏览器到关闭浏览器
	application.setAttribute("name", "val");   // 在服务器中有效，从打开服务器到关闭服务器
%>
```

**读取信息**

```jsp
每个对象都有getAttribute()可以读取，也可以用pageContext.findAttribute()查找
<%
	pageContext.findAttribute("name");
%>
```

pageContext.findAttribute()自底向上查找：page->request->session->application，都不存在放回null

*类似于JVM的双亲委派机制，加载类时从底向上找，优先使用底层的*



习惯（存放客户端向服务器发送请求中产生的数据）：

- request：用户用完不再需要，如新闻
- session：用户频繁使用，如购物车
- application：多个用户使用，如聊天记录



3.6.4 标签与表达式

```xml
<!-- JSTL表达式依赖 -->
<dependency>
    <groupId>javax.servlet.jsp.jstl</groupId>
    <artifactId>jstl-api</artifactId>
    <version>1.2</version>
</dependency>
<!-- Standard标签库 -->
<dependency>
    <groupId>taglibs</groupId>
    <artifactId>standard</artifactId>
    <version>1.1.2</version>
</dependency>
```

##### 1、JSP标签

```jsp
<%--提取公共页面--%>
<jsp:include page="/index.jsp">

<%--带参请求转发--%>
<jsp:forward page="/1.jsp">
	<jsp:param name="name" value="val"/>
</jsp:forward>  
    
<%--转发后的页面获取参数（request级）--%>
<%=request.getParameter("name")%>
```

##### 2、JSTL标签

JSTL标签库用于弥补HTML标签的不足，其标签的功能和java代码一样

> 核心标签 (掌握部分)

```jsp
<%--在jsp文件开头引入核心标签库--%>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<%--如果引入后运行报500错误，把jstl-api和standard的jar包复制到tomcat的该工程的工作空间中的WEB-INF下的lib中--%>
```

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211126151314065.png" alt="image-20211126151314065" style="zoom:67%;" />

```jsp
<%--jstl标签if，test中为判断的表达式，为true则执行<c:if></c:if>中的内容；
    其余两个参数为可选项，var的值为一个变量名，保存test的判断结果true/false; scope规定var属性的作用域--%>
<c:if test="<boolean>" var="<string>" scope="<string>">
   ...
</c:if>

<%--设置变量--%>
<c:set var="score" value="85"/>

<%--实现仿switch--%>
<c:choose>
    <c:when test="${score>=90}">
        优秀
    </c:when>
    <c:when test="${score>=80}">
        优秀
    </c:when>
    <c:otherwise>
        其他
    </c:otherwise>
</c:choose>
```

<%--实现for循环--%>

<%--参数var是每次遍历出的变量，items是遍历对象--%>

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211126154444911.png" alt="image-20211126154444911" style="zoom:67%;" />

<%--begin, end是起始的下标，step是步长--%>![image-20211126154814134](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211126154814134.png)

> 格式化标签

> SQL标签

> XML标签



##### 3、EL表达式：${}

​	获取数据

​	执行运算

​	获取web开发的常用对象



#### 3.6.5 JavaBean

JavaBean 是特殊的 Java 类，使用 Java 语言书写，并且遵守 JavaBean API 规范：

- 提供一个默认的无参构造函数。
- 需要被序列化并且实现了 Serializable 接口。
- 可能有一系列可读写属性。
- 可能有一系列的 getter 或 setter 方法。

（一般是实体类，用来和数据库的字段做映射ORM<对象关系映射>，对应一个表）

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211126162121125.png" alt="image-20211126162121125" style="zoom:80%;" />

在java目录中对应路径下定义好该实体类后，在jsp文件中使用，有两种写法：

第一种，普通的java写法，new创建，set赋值，get取值（一般更喜欢用普通写法...）

第二种，< jsp:useBean >创建，< jsp:setProperty >赋值，< jsp:getProperty >取值



### 3.7 MVC三层架构

M：model，模型 -> 实体类和数据库中对应的字段

V：view，视图 -> 页面

- 展示数据
- 提供可供我们操作的请求

C：controller，控制器 -> Servlet

- 接受用户请求
- 响应给客户端内容
- 重定向或转发



Servlet和JSP都可以写java，为了运行维护：

Servlet专注于处理请求、控制视图跳转；

JSP专注于显示数据；

1、早期

![image-20211126175233664](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211126175233664.png)

客户端直接访问控制层，控制层直接操作数据库。

程序臃肿，不易维护

2、三层架构

![image-20211126180011585](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211126180011585.png)

将具体的操作交给Model，主要由业务层service处理用户请求，数据访问层dao进行具体实现

**Model**

- 业务处理：service
- 数据访问：dao（增删查改 CRUD）

**View**

- 展示数据
- 提供链接供用户发起Servlet请求

**Controller（Servlet）**

- 接收用户请求（req的请求参数、session信息）
- 交给业务层处理对应的代码
- 控制视图跳转

（SSM：spring->service、springmvc->view和controller、mybatis->dao）

**流程实例：**

![image-20211126180936227](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211126180936227.png)

数据库操作的结果从底往上依次传递



### 3.8 过滤器Filter (重点)

位于web服务器和动静态的资源之间，对服务器使用资源的请求和返回资源的内容进行自动处理。

可有多个过滤器，分别进行不同的过滤。



原理和servlet相似，实现过滤器接口。

1、实现

和Servlet一样，先建一个类实现Filter接口

```java
// 要导入javax.servlet的Filter
import javax.servlet.*;
import java.io.IOException;

public class CharacterEncoding implements Filter {

    // 初始化，在服务器启动时初始化
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {}

    // 过滤
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        servletRequest.setCharacterEncoding("utf-8");
        servletResponse.setCharacterEncoding("utf-8");
        servletResponse.setContentType("text/html;charset=UTF-8");

        filterChain.doFilter(servletRequest, servletResponse); // 让过滤器程序继续运行，否则程序就直接停止了
    }

    // 销毁，在服务器关闭时销毁
    @Override
    public void destroy() {}
}
```

再在web.xml中设置

```xml
<filter>
    <filter-name>characterEncoding</filter-name>
    <!--指定使用的过滤器类-->
    <filter-class>com.ghy.filter.CharacterEncoding</filter-class>
</filter>
<filter-mapping>
    <filter-name>characterEncoding</filter-name>
    <!-- /servlet下的任何请求都会被此过滤器过滤-->
    <url-pattern>/servlet/*</url-pattern>
</filter-mapping>
```

2、应用：权限拦截（用户登录）

- 登录页面jsp的表单设置上传对象为一个网址
- web.xml中将此网址指向对应的servlet类
- 该servlet类对上传的信息进行检查，如果是符合要求的，在session中添加对应的属性，并重定向到成功页面jsp（用户界面）；否则重定向到失败页面jsp（失败提示或登录页面）。
- 用户页面添加一个超链接，为退出登录的地址
- web.xml中将此网址指向退出登录的servlet类
- 退出登录的servlet类中，检查session，如果session中有相关属性，将之删除（不整个删除session，那样会导致session频繁地创建和销毁），并重定向到相关页面
- **实现过滤器类，对用户页面进行过滤，检查session，如果是正常登录的，session中肯定有相关属性，放行；如果没有相关属性的话，重定向到失败页面，防止用户页面被异常登录。**

注

- 体现了session对同一个客户端的所有访问共享的优势
- session的属性值可以是对象
- 用得多的话，可以建一个工具类，将各种session的属性名封装为常量，降低耦合度

### 3.9 监听器Listener

实现一个监听器接口（监听器的接口极多，根据需求实现不同的接口）

用得少，GUI编程会用

1、实现

先实现一个监听器接口

```java
public class OnlineCountListener implements HttpSessionListener {
    
    // 监听客户端数量
    
    // 创建Session监听
    @Override
    public void sessionCreated(HttpSessionEvent httpSessionEvent) {
        ServletContext ctx = httpSessionEvent.getSession().getServletContext();
        Integer onlineCount = (Integer) ctx.getAttribute("OnlineCount");
        if(onlineCount == null){
            onlineCount = new Integer(1);
        }
        else{
            int count = onlineCount.intValue();
            onlineCount = new Integer(count + 1);
            ctx.setAttribute("OnlineCount", onlineCount);
        }
    }

    // 销毁Session监听
    @Override
    public void sessionDestroyed(HttpSessionEvent httpSessionEvent) {
        ServletContext ctx = httpSessionEvent.getSession().getServletContext();
        Integer onlineCount = (Integer) ctx.getAttribute("OnlineCount");
        if(onlineCount == null){
            onlineCount = new Integer(0);
        }
        else{
            int count = onlineCount.intValue();
            onlineCount = new Integer(count - 1);
            ctx.setAttribute("OnlineCount", onlineCount);
        }
    }
}
```

再在web.xml中设置

```xml
<listener>
    <listener-class>com.ghy.Listener.OnlineCountListener</listener-class>
</listener>
```



## 4 MyBatis

环境：jdk、Mysql、maven、IDEA

涉及：JDBC、Maven、Junit、Mysql

MyBatis 是一款优秀的持久层框架，它支持自定义 SQL、存储过程以及高级映射。MyBatis 免除了几乎所有的 JDBC 代码以及设置参数和获取结果集的工作。MyBatis 可以通过简单的 XML 或注解来配置和映射原始类型、接口和 Java POJO（Plain Old Java Objects，普通老式 Java 对象）为数据库中的记录。

优点

- 帮助程序员将数据存入数据库，比传统的JDBC更简便、自动化。
- 实现了sql和代码的分离，可维护性好 
- 提供映射标签，支持对象与数据库的orm字段关系映射
- 提供对象关系映射标签，支持对象关系组建维护
- 提供xml标签，支持编写动态sql



> 执行流程

1. 加载resources目录下的核心配置文件
2. 实例化SqlSessionFactoryBuilder构造器
3. 解析xml格式的核心配置文件
4. 将配置信息保存在configuration中
5. 实例化SqlSessionFactory
6. transaction事务管理
7. 创建executor执行器
8. 创建SqlSession
9. 实现CRUD，如果执行出错或失败会回滚到6. transaction事务管理
10. 提交事务
11. 关闭SqlSession

面试会问，可以自己Debug运行test程序，一步一步看



> 获取

- Maven仓库

  ```xml
  <!-- https://mvnrepository.com/artifact/org.mybatis/mybatis -->
  <dependency>
      <groupId>org.mybatis</groupId>
      <artifactId>mybatis</artifactId>
      <version>3.5.7</version>
  </dependency>
  ```

- Github：https://github.com/mybatis/mybatis-3

- 官方文档：https://mybatis.org/mybatis-3/zh/index.html

  

> 持久层

数据持久化：将内存中断电即失的数据转为持久状态，如数据库、IO文件 

数据持久层是完成持久化工作的代码块



### 4.1 实现步骤

#### 1、搭建环境

​	搭建数据库

```sql
CREATE TABLE `user`(
	`id` INT(20) NOT NULL PRIMARY KEY,
	`name` VARCHAR(30) DEFAULT NULL,
	`pwd` VARCHAR(30)  DEFAULT NULL
)ENGINE=INNODB DEFAULT CHARSET=utf8;

INSERT INTO `user`(`id`,`name`,`pwd`) VALUES
(1,'高昊宇','001118'),
(2,'西琳','123456'),
(3,'福斯特','111111')
```

​	新建普通Maven工程，删除src目录

​	导包

```xml
<!--导入依赖-->
<dependencies>
    <!--mysql驱动-->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.25</version>
    </dependency>
    <!--mybatis-->
    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis</artifactId>
        <version>3.5.7</version>
    </dependency>
    <!--junit-->
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
    </dependency>
</dependencies>
```

#### 2、新建模块

编写MyBatis工具类（com.ghy.utils包下）

```java
// sqlSessionFactory，用于构建sqlSession
public class MybatisUtils {

    private static SqlSessionFactory sqlSessionFactory;

    static{

        // 创建sqlSessionFactory对象
        try {
            String resource = "mybatis-config.xml";
            InputStream inputStream = Resources.getResourceAsStream(resource);
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static SqlSession getSqlSession(){
        return sqlSessionFactory.openSession();
        return sqlSessionFactory.openSession(true); // 自动commit
    }
}
```

在resources中新建MyBatis核心配置文件

```xml
<?xml version="1.0" encoding="UTF8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<!--核心配置文件-->
<configuration>
    <environments default="development">

        <!--默认环境-->
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/mybatis?useSSL=true&amp;useUnicode=true&amp;characterEncoding=UTF-8&amp;serverTimezone=UTC"/>
                <property name="username" value="root"/>
                <property name="password" value="001118"/>
            </dataSource>
        </environment>
    </environments>

    <!--每个Mapper.xml都要在MyBatis核心配置文件中注册(路径用斜杠分隔！！！)-->
    <mappers>
        <mapper resource="com/ghy/dao/UserMapper.xml"/>
    </mappers>
</configuration>
```



#### 3、编写代码

- 实体类（与数据库的表对应，属性+初始化方法+getter和setter+toString）

  ```java
  public class User {
      private int id;
      private String name;
      private String pwd;
  
      public User(){
  
      }
      public User(int id, String name, String pwd) {
          this.id = id;
          this.name = name;
          this.pwd = pwd;
      }
  
      public int getId() {
          return id;
      }
  
      public String getName() {
          return name;
      }
  
      public String getPwd() {
          return pwd;
      }
  
      public void setId(int id) {
          this.id = id;
      }
  
      public void setName(String name) {
          this.name = name;
      }
  
      public void setPwd(String pwd) {
          this.pwd = pwd;
      }
  
      @Override
      public String toString() {
          return "User{" +
                  "id=" + id +
                  ", name='" + name + '\'' +
                  ", pwd='" + pwd + '\'' +
                  '}';
      }
  }
  ```

- Dao接口（与实体类对应，提供对实体类在数据库中的表的各种操作）

  ```java
  public interface UserDao {
      List<User> getUserList();
  }
  ```

- 接口实现类 

  用一个Mapper配置文件实现。

  每个sql语句用一个标签实现，

  - id对应接口类中的方法；
  - resultType/resultMap是返回类型，对应实体类；
  - 标签中是具体的sql语句

  ```java
  <?xml version="1.0" encoding="UTF8" ?>
  <!DOCTYPE mapper
          PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
          "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
  
  <!--namespace绑定一个对应的Dao/Mapper接口-->
  <mapper namespace="com.ghy.dao.UserDao">
  
      <!--select查询语句 -->
      <select id="getUserList" resultType="com.ghy.pojo.User" >
          select * from mybatis.user
      </select>
  </mapper>
  ```

namespace内容应和对应的接口名一致

#### 4、测试

在tes/java目录下创建测试类

```java
public class UserDaoTest {

    @Test
    public void test(){

        // 1、获取SqlSession对象
        SqlSession sqlSession = MybatisUtils.getSqlSession();

        try{
            // 2、执行sql
            UserDao userDao = sqlSession.getMapper(UserDao.class);
            List<User> userList = userDao.getUserList();

            for(User user: userList){
                System.out.println(user);
            }
        }catch(Exception e){
            e.printStackTrace();
        }finally {
            // 3、关闭sqlSession
            sqlSession.close();
        }
    }
}
```



### 4.2 CRUD

1、在接口类中定义方法

```java
// 按id查询用户
User getUserById(int id);

// 插入一个用户
int addUser(User user);

// 修改用户
int updateUser(User user);

// 删除用户
int deleteUser(int id);
```

2、在实现接口的Mapper文件中编写sql语句

```xml
<select id="getUserById" parameterType="int" resultType="com.ghy.pojo.User" >
    select * from mybatis.user where id = #{id}
</select>

<insert id="addUser" parameterType="com.ghy.pojo.User">
    insert into mybatis.user (id, name, pwd) values (#{id}, #{name}, #{pwd})
</insert>

<update id="updateUser" parameterType="com.ghy.pojo.User">
    <!--参数为实体类时，可以直接获取其数据成员-->
    update mybatis.user set name=#{name}, pwd=#{pwd} where id=#{id}
</update>

<delete id="deleteUser" parameterType="int">
    delete from mybatis.user where id=#{id}
```

id：对应的接口类的方法名

parameterType：参数类型

resultType：sql查询操作的返回值；增删改不用也不能写，默认返回受影响的行数

- 需要传参时，用**#{参数名}**获取参数，参数名和接口对应方法中的一致，参数为实体类时，可以直接获取其数据成员

- **增删改需要提交事务**

3、编写测试类

```java
@Test
public void updateUser(){
    SqlSession sqlSession = MybatisUtils.getSqlSession();
    UserMapper mapper = sqlSession.getMapper(UserMapper.class);
    int res = mapper.updateUser(new User(2, "西林", "123123"));
    sqlSession.commit();   // 提交事务
    if(res > 0){
        System.out.println("修改成功！");
    }
    sqlSession.close();
}
```

先获得数据库的SqlSession，再获得实体类的Mapper，再调用相应的方法。增删改最后要提交事务。



> map传参

接口中设参数为map

```java
// map查询用户
User getUserById2(Map<String, Object> map);
```

Mapper文件中parameterType为map，#{key}去参数，key是实参map中的key值

```xml
<select id="getUserById2" parameterType="map" resultType="com.ghy.pojo.User" >
	select * from mybatis.user where id = #{userid}
</select>
```

测试类中定义map，调用方法

```java
@Test
public void getUserById2(){
    SqlSession sqlSession = MybatisUtils.getSqlSession();
    UserMapper userDao = sqlSession.getMapper(UserMapper.class);
    Map<String, Object> map = new HashMap<String, Object>();  // 自定义map
    map.put("userid", 3);   // key值和Mapper中取参名一致
    User user = userDao.getUserById2(map);
    System.out.println(user);
    sqlSession.close();
}
```

多用于查询



> 传参

- 1个：直接传

- 多个：map或注解；都是一个实体类的成员的话可以传实体类的对象



> 模糊查询两种方法

![image-20211130163843110](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211130163843110.png)



> sql注入问题

- \#{}是一个参数占位符，对于String类型会自动加上""，其他类型不加。由于Mybatis采用预编译，其后的参数不会再进行SQL编译，所以一定程度上防止SQL注入。
- ${}是一个简单的String替换，字符串是什么，解析就是什么，无法处理sql注入。
- 类如order by。假如前端传的参数是id(假设id是String类型)，
  - 对于order by #{id},对应的sql语句就是 order by “id”;
  - 对于order by ${id},对应的sql语句则是order by id。这种情况，当用户传参为id && 1=1 的时候，就会产生难以预计的后果。



### 4.3 配置解析

核心配置文件 mybatis-config.xml

```
properties（属性）
settings（设置）
typeAliases（类型别名）
typeHandlers（类型处理器）
objectFactory（对象工厂）
plugins（插件）
environments（环境配置）
environment（环境变量）
transactionManager（事务管理器）
dataSource（数据源）
databaseIdProvider（数据库厂商标识）
mappers（映射器）
```

事务管理器transactionManager有JDBC和MANAGED，不过后者基本不用了，默认JDBC；如果用Spring，不需要设置它

数据源类型dataSource type用UNPOOLED、POOLED (默认)、JNDI

尽管能配多个环境，每个SqlSessionFactory实例只能选择一个



#### 4.3.1 属性优化（properties）

**mybatis-config.xml中的标签有顺序**

![image-20211130171742501](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211130171742501.png)

最后读取作为方法参数传递的属性，并覆盖之前读取过的同名属性。



**从外部文件读取环境配置**

```xml
<!--引入外部配置文件-->
<properties resource="db.properties"/>

<environments default="development">
    <environment id="development">
        <transactionManager type="JDBC"/>
        <dataSource type="POOLED">
            <property name="driver" value="${driver}"/>
            <property name="url" value="${url}"/>
            <property name="username" value="${username}"/>
            <property name="password" value="${password}"/>
        </dataSource>
    </environment>
</environments>
```

​	在resources目录下新建文件db.properties

```
driver=com.mysql.cj.jdbc.Driver
url=jdbc:mysql://localhost:3306/mybatis?useSSL=true&useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC
username=root
password=001118
```

​	properties标签间也可以写property，读取顺序（有同名属性时后读取的覆盖之前的）：

- ​	properties 元素体内指定的属性

- ​	properties 元素中的 resource 属性读取类路径下属性文件，或根据 url 属性指定的路径读取属性文件

- ​	作为方法参数传递的属性

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211130172820428.png" alt="image-20211130172820428" style="zoom:80%;" />



#### 4.3.2 别名优化（typeAliases）

类型别名可为 Java 类型设置一个缩写名字，以降低冗余的全限定类名书写。

**方法1：直接取别名**

```xml
<typeAliases>
    <typeAlias type="com.ghy.pojo.User" alias="User"/>
</typeAliases>
```

**方法2：扫描所在包**

```xml
<typeAliases>
    <package name="com.ghy.pojo"/>
</typeAliases>
```

别名默认是包中该类的类名，并将首字母小写。

除非该类有别名注解，以注解优先。

```java
@Alias("man")
public class User {
}
```

常见类型的别名：https://mybatis.org/mybatis-3/zh/configuration.html

对于基本数据类型（如int），直接写是包装类（Integer），加“_”前缀 _int才是基本数据类型（int）



#### 4.3.3 设置优化（settings）

```xml
<settings>
    <!--自动进行java中aName类到数据库中列名a_name的映射-->
  <setting name="mapUnderscoreToCamelCase" value="true"/>
    
    <!--指定 MyBatis 所用日志的具体实现，未指定时将自动查找-->
   <setting name="logImpl" value="COMMONS_LOGGING"/>
    
    <!--全局性地开启所有映射器配置文件中已配置的缓存-->
  <setting name="cacheEnabled" value="true"/>
    
    <!--延迟加载的全局开关。开启时，所有关联对象都会延迟加载-->
  <setting name="lazyLoadingEnabled" value="true"/>
</settings>
```

![image-20211130180430172](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211130180430172.png)

其他设置：https://mybatis.org/mybatis-3/zh/configuration.html



#### 4.3.4 映射器(mappers)

绑定mapper

方法一：

```xml
<!--resource=路径-->
<mappers>
    <mapper resource="com/ghy/dao/UserMapper.xml"/>
</mappers>
```

方法二:

```xml
<!--用class文件绑定-->
<mappers>
    <mapper class="com.ghy.dao.UserMapper"/>
</mappers>
```

使用class文件绑定时，Mapper文件和接口必须同名，且必须在同一包下

方法三：

```xml
<!--用package绑定-->
<mappers>
    <package name="com.ghy.dao"/>
</mappers>
```

Mapper文件和接口必须同名，且必须在同一包下



#### 4.3.5 生命周期（Scope）和作用域

作用域和生命周期类别是至关重要的，错误的使用会导致严重的**并发**问题

1、SqlSessionFactoryBuilder

- 只用于生成SqlSessionFactory，生成后即可释放。

- 作用域：工具类static代码块

2、SqlSessionFactory

- 地位类似于数据库连接池，需要操作数据库时提供连接、

- 一旦被创建就应该在应用的运行期间一直存在

- 作用域：应用的作用域

3、SqlSession

- 对数据库的一次访问，可以连接多个Mapper，由Mapper负责对数据库的操作
- 不是线程安全的，不能被共享

- 作用域：一次请求
- 官方文档：如果你现在正在使用一种 Web 框架，考虑将 SqlSession 放在一个和 HTTP 请求相似的作用域中。 换句话说，每次收到 HTTP 请求，就可以打开一个 SqlSession，返回一个响应后，就关闭它。



> 其他

- [typeHandlers（类型处理器）](https://mybatis.org/mybatis-3/zh/configuration.html#typeHandlers)
- [objectFactory（对象工厂）](https://mybatis.org/mybatis-3/zh/configuration.html#objectFactory)
- [plugins（插件）](https://mybatis.org/mybatis-3/zh/configuration.html#plugins)
  - mybatis-generator-core
  - mybatis-plus
  - 通用mapper



### 4.4 ResultMap

查询中，当数据库中表的列名和定义的实体类中对应的变量名一致时，MyBatis 会在幕后自动创建一个 `ResultMap`，再根据属性名来映射列到 JavaBean 的属性上。（如数据库的id值会存入实体类中的id变量）

但如果二者名称不一致，就无法自动映射了（基于map，key值不一样了自然对不上），例如：

```
数据库：id name pwd
实体类: id name password
```

执行select * ……后，password那一项结果会是null。

一种低级的解决方法时在select语句中起别名：select psw as password……

更优的方法是使用ResultMap



> ResultMap使用

```xml
<resultMap id="UserMap" type="user">
    <result column="pwd" property="password"/>
</resultMap>

<select id="getUserById" parameterType="int" resultMap="UserMap" >
    select * from mybatis.user where id = #{id}
</select>
```

1、将返回类型改为resultMap。

2、新建resultMap标签，id对应刚才返回的resultMap的名字，type对应实体类（有别名用别名，没有全称）

3、resultMap标签中新建result标签，进行属性的映射，column为数据库中的列名，property为实体类中变量名



### 4.5 日志

让程序打印数据库操作的日志。

只需在核心配置文件的settings中设置即可。

```xml
<!--设置-->
<settings>
    <setting name="logImpl" value="STDOUT_LOGGING"/>
</settings>
```

value：**SLF4J** | **LOG4J** | LOG4J2 | JDK_LOGGING | COMMONS_LOGGING | **STDOUT_LOGGING** | NO_LOGGING



#### 4.5.1 STDOUT_LOGGING

STDOUT_LOGGING是标准日志输出

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211201115344401.png" alt="image-20211201115344401" style="zoom:80%;" />



#### 4.5.2 LOG4J

- Log4j是Apache的一个开源项目，使用Log4j，可以控制日志信息输送的目的地是控制台、文件、GUI组件等；
- 我们可以控制每一条日志的输出格式；
- 通过定义每一条日志信息的级别，我们能够更加细致地控制日志的生成过程；
- 可以通过一个配置文件来灵活地进行配置，而不需要修改应用的代码；

1、导入jar包

```xml
<!-- https://mvnrepository.com/artifact/log4j/log4j -->
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.17</version>
</dependency>
```

2、在resources目录下新建配置文件log4j.properties

```properties
#将等级为DEBUG的日志信息输出到console和file这两个目的地，console和file的定义在下面的代码
log4j.rootLogger=DEBUG,console,file

#控制台输出的相关设置
log4j.appender.console = org.apache.log4j.ConsoleAppender
log4j.appender.console.Target = System.out
log4j.appender.console.Threshold=DEBUG
log4j.appender.console.layout = org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=【%c】-%m%n

#文件输出的相关设置
log4j.appender.file = org.apache.log4j.RollingFileAppender
log4j.appender.file.File=./log/ghy.log
log4j.appender.file.MaxFileSize=10mb
log4j.appender.file.Threshold=DEBUG
log4j.appender.file.layout=org.apache.log4j.PatternLayout
log4j.appender.file.layout.ConversionPattern=【%p】【%d{yy-MM-dd}】【%c】%m%n

#日志输出级别
log4j.logger.org.mybatis=DEBUG
log4j.logger.java.sql=DEBUG
log4j.logger.java.sql.Statement=DEBUG
log4j.logger.java.sql.ResultSet=DEBUG
log4j.logger.java.sql.PreparedStatement=DEBUG
```

3、在核心配置文件的settings中设置

```
<settings>
    <setting name="logImpl" value="LOG4J"/>
</settings>
```

4、使用

- 在使用的类中，导入包

- 新建日志对象，参数为当前类的class

  ```
  static Logger logger = Logger.getLogger(UserDaoTest.class);
  ```

- 使用不同级别的日志

  ```java
  logger.info("info:进入logTest");
  logger.debug("debug:进入logTest");
  logger.error("error:进入logTest");
  ```

生成的日志：

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211201143618283.png" alt="image-20211201143618283" style="zoom:80%;" />



### 4.6 分页



#### 4.6.1 使用sql语句的limit分页

```sql
select * from user limit startIndex, pageSize
```

从结果的第startIndex开始输出，最多输出pageSize个



> MyBatis实现

接口：

```java
List<User> getUserByLimit(Map<String, Integer> map);
```

mapper文件：

```xml
<select id="getUserByLimit" parameterType="map" resultMap="UserMap" >
    select * from mybatis.user where id=#{id} limit #{startIndes},#{pageSize}
</select>
```

实际使用：

```java
@Test
public void getUserByLimit(){

    SqlSession sqlSession = MybatisUtils.getSqlSession();
    UserMapper mapper = sqlSession.getMapper(UserMapper.class);

    HashMap<String, Integer> map = new HashMap<String, Integer>();
    map.put("startIndex", 1);
    map.put("pageSize", 2);

    List<User> userList = mapper.getUserByLimit(map);
    for(User user: userList){
        System.out.println(user);
    }
    sqlSession.close();
}
```

手动创建map进行传参



#### 4.6.2 RowBounds分页

不推荐使用



#### 4.6.3 pageHelper插件

见官方文档：https://pagehelper.github.io/



### 4.7 使用注解开发

1、直接在接口上实现，注解sql语句，无需再用mapper实现

```java
@Select("select * from user")  // 和mapper标签中的sql语句内容一样
List<User> getUser();

@Select("SELECT * FROM blog WHERE id = #{id}")
Blog selectBlog(int id);
```

2、需要在核心配置文件**用class**绑定接口

```xml
<mappers>
    <mapper class="com.ghy.dao.UserMapper"/>
</mappers>
```

使用注解来映射简单语句使代码更加简洁，但对于复杂的语句，Java 注解力不从心，还会让 SQL 语句更加混乱

推荐：简单的用注解，复杂的用mapper实现

本质：反射机制实现

底层：动态代理



#### **@Param()注解**

```
@delete("delete from user where id = #{uid}")
void deleteUser(@Param(“uid”) int id);
```

- 基本类型和String类型最好加上，引用类型不需要。

- sql引用的#{}中以其中定义的名称为准

- 只有一个基本类型参数是非必需，但最好加上；多个时必须加上



#### lombok

1、安装lombok的IDEA插件

2、导入lombok的jar包

```xml
<!-- https://mvnrepository.com/artifact/org.projectlombok/lombok -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.22</version>
    <scope>provided</scope>
</dependency>
```

3、功能注解

```
@Getter and @Setter
@FieldNameConstants
@ToString
@EqualsAndHashCode
@AllArgsConstructor, @RequiredArgsConstructor and @NoArgsConstructor
@Log, @Log4j, @Log4j2, @Slf4j, @XSlf4j, @CommonsLog, @JBossLog, @Flogger, @CustomLog
@Data
@Builder
@SuperBuilder
@Singular
@Delegate
@Value
@Accessors
@Wither
@With
@SneakyThrows
@val
@var
experimental @var
@UtilityClass
```

@Data：自动生成无参构造函数，getter和setter，toString，hashcode，equals

@AllArgsConstructor：生成全参数的构造函数（会消除@Data生成的无参构造函数）

@NoArgsConstructor：生成无参构造

@Getter、@Setter：放在类上，生成所有属性的getter、setter；放在属性上，生成该属性的getter、setter



争议：不仅仅是方便了我们写代码，直接修改了我们的源码



### 4.8 多对一

teacher表：

![image-20211202153826256](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211202153826256.png)

student表：

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211202153903572.png" alt="image-20211202153903572" style="zoom:80%;" />

求每个学生及其对应的老师名，可用sql的子查询或多表查询实现。两种查询在mybatis中也分别对应两种实现方法：



#### 4.8.1 按照查询嵌套处理

```xml
<resultMap id="ST" type="student">
    <result column="id" property="id"/>
    <result column="name" property="name"/>
    <!--association处理对象属性-->
    <association column="tid" property="teacher" javaType="teacher" select="getTeacher"/>
</resultMap>

<select id="getStudent" resultMap="ST">
    select * from student
</select>

<select id="getTeacher" resultType="teacher">  <!--参数名随意-->
    select * from teacher where id = #{id}
</select>
```

getStudent函数为实际使用的方法，getTeacher是嵌套到内层的查询。

resultMap中，用association对变量进行设置，column是数据库对应列，property是student类中对应属性名，javaType为该属性对应的实体类，select进行子查询。

处理步骤：

- 查询到tid，发现没有tid，去map里找，发现和teacher属性对应
- 发现teacher属性是“teacher”类的，不能直接赋值，要用tid进行子查询
- 把tid代入select指定的子查询getTeacher，与id为getTeacher的select标签匹配
- 将tid代入进行子查询，返回teacher内容，存入Student类的该属性中



#### 4.8.2 按照结果嵌套处理

```xml
<resultMap id="ST2" type="student">
    <result column="sid" property="id"/>  <!--column与sql取的别名一致-->
    <result column="sname" property="name"/>
    <association property="teacher" javaType="teacher">
        <result column="tname" property="name"/>
    </association>
</resultMap>

<select id="getStudent2" resultMap="ST2">
    select s.id sid,s.name sname,t.name tname
    from student s, teacher t
    where s.tid = t.id
</select>
```

select部分采用和sql多表查询一致的写法。在resultMap中，对Teacher类的属性也进行设置。

处理步骤：

- 前两个属性，由于sql起了别名，因此是sid和sname，查map后和Student类的id、name对应并存储

- 最后一个属性在resultMap是association，表明是一个对象

- 根据javaType，新建一个Teacher类的对象赋给Student中的teacher属性（无参构造函数）

- 遍历association内部的result，对匹配的属性用set方法赋值

  - 数据库查到的tname，根据association内部的result的property，对应Teacher类的name属性，故用set赋值

  

### 4.9 一对多



teacher表：

![image-20211202153826256](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211202153826256.png)

student表：

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211202153903572.png" alt="image-20211202153903572" style="zoom:80%;" />

求一个老师和其所有的学生，可用sql的子查询或多表查询实现。两种查询在mybatis中也分别对应两种实现方法：

#### 4.9.1 按照查询嵌套处理

```xml
<resultMap id="TS" type="teacher">
    <result column="id" property="id"/>
    <!--集合用collection-->
    <collection property="students" javaType="ArrayList" ofType="student" select="getStudent" column="id">
    </collection>
</resultMap>

<select id="getTeacher" resultMap="TS">
    select * from teacher where id = #{tid}
</select>

<select id="getStudent" resultType="student">
    select * from student where tid = #{teacherId}  <!--参数名随意-->
</select>
```

getTeacher函数为实际使用的方法，Student是嵌套到内层的查询。

resultMap中，用collection对列表进行设置，property是student类中对应的列表属性名，javaType为集合的类（因为是列表，所以是ArrayList），ofType是列表元素的类型，select进行子查询，column中指定向子查询的传参。

处理步骤：

- 前两个属性，直接对应

- 第三个属性是集合，根据javaType和ofType，新建ArrayList<Student>并赋给property中指向的students属性

- 调用select中设好的getStudent子查询，并根据column的设置，把teacher表的id属性传给子查询

- 子查询通过收到的参数进行操作，返回所有符合条件的student对象

- 把子查询输出的每个student对象插入students列表

  

#### 4.9.2 按照结果嵌套处理

```xml
<resultMap id="TS" type="teacher">
    <result column="tid" property="id"/>
    <result column="tname" property="name"/>
    <!--集合用collection-->
    <collection property="students" ofType="student">  <!--集合泛型的类型用ofType-->
        <result column="sid" property="id"/>
        <result column="sname" property="name"/>
    </collection>
</resultMap>

<select id="getTeacher2" resultMap="TS">
    select t.id tid, t.name tname, s.id sid, s.name sname
    from teacher t, student s
    where t.id = #{tid} and t.id = s.tid
</select>
```

select部分采用和sql多表查询一致的写法。在resultMap中，对Student的列表也进行设置。

处理步骤：

- 前两个属性，由于sql起了别名，因此是tid和tname，查map后和Teacher类的id、name对应并存储
- 后两个属性在resultMap是collection，表明是一个集合
- 根据ofType，集合的元素是student对象
- 遍历collection内部的result，进行属性的匹配，数据库查到的sid、sname分别对应Student的id、name属性
- 对每一组查到的student内容，先使用无参构造方法新建列表元素，再用set方法对两个属性进行赋值



### 4.10 动态SQL

动态SQL：根据不同的条件生成不同的SQL语句。

本质还是SQL，但可以使用一些逻辑代码进行SQL语句的拼接，要保证拼接后语句的格式正确

#### 4.10.1 if、where

```XML
<select id="queryBlogIF" parameterType="map" resultType="blog">
    select * from blog where 1=1  <!--不合理，用where-->
    <if test="title != null">
        and title = #{title}
    </if>
    <if test="author != null">
        and author = #{author}
    </if>
</select>
```

用if标签进行sql的动态生成，使用map传参，在if标签中对map中的内容进行检查，根据结果进行sql语句的拼接

```java
HashMap map = new HashMap();
//map.put("title", "Java");
map.put("author", "西琳");
List<Blog> blogs = mapper.queryBlogIF(map);
```

这样map中的数据不同时，执行的搜索也不一样



> where标签

```xml
<select id="queryBlogIF" parameterType="map" resultType="blog">
    select * from blog
    <where>
        <if test="title != null">
        	title = #{title}
    	</if>
    	<if test="author != null">
        	and author = #{author}
    	</if>
    </where>
</select>
```

where 元素只会在子元素返回任何内容的情况下才插入 “WHERE” 子句。而且，若子句的开头为 “AND” 或 “OR”，where 元素也会将它们去除。

#### 4.10.2 choose、when、otherwise

```xml
<select id="queryBlogChoose" parameterType="map" resultType="blog">
    select * from blog
    <where>
        <choose>
            <when test="title != null">
                title = #{title}
            </when>
            <when test="author != null">
                author = #{author}
            </when>
            <otherwise>
                views = #{views}
            </otherwise>
        </choose>
    </where>
</select>
```

用choose实现仿switch的结构，从上往下依次匹配各个when标签，只会拼接第一个test成立的



#### 4.10.3 set、trim

> set

```xml
<update id="updateBlog" parameterType="map">
    update blog
    <set>
        <if test="title != null">title = #{title},</if>
        <if test="author != null">author = #{author},</if>
        <if test="createTime != null">create_time = #{createTime},</if>
        <if test="views != null">views = #{views}</if>
    </set>
    where id = #{id}
</update>
```

```java
HashMap map = new HashMap();
map.put("id", "55661b82283f43b4b765e2d55f92778f");
map.put("title", "看那边啊！");
//map.put("author", "福斯特.L");
map.put("createTime", new Date());
//map.put("views", "1067");
mapper.updateBlog(map);
```

set用于更新语句，可以动态修改插入的列。

set 元素自动在插入语句的开头增加 SET 关键字，并会删除插入语句最后额外的逗号。

注：无列插入时也不会增加 SET 关键字，但sql语句本身运行时会报语法错误



> trim

```xml
<trim prefix="" prefixOverrides="" suffix="" suffixOverrides="">
  ...
</trim>
```

整合标签内的内容后，删除语句开头的prefixOverrides和结尾的suffixOverrides，再在开头加上prefix，在结尾加上suffix。

可用于实现where、set标签

```xml
<!--实现where-->
<trim prefix="WHERE" prefixOverrides="AND |OR ">
  ...
</trim>

<!--实现set-->
<trim prefix="SET" suffixOverrides=",">
  ...
</trim>
```



#### 4.10.4 foreach

用于对某个属性的遍历（如2<id<10）

```xml
<select id="queryBlogForeach" parameterType="map" resultType="blog">
    select * from blog
    <where>
        id in
        <foreach collection="ids" item="id" open="(" separator="," close=")">
            #{id}
        </foreach>
    </where>
</select>
<!--等价于 select * from blog where id in (X,X,…)-->
```

foreach会生成一段语句：

- 开头是open中的内容，结尾是close，遍历对象由collection指定，每次遍历得到的元素用item中的内容命名；

- foreach标签中的是每次遍历向已生成的语句后添加的内容，一般是遍历得到的元素，每两个内容间添加separator中的间隔内容；



#### 4.10.5 sql（SQL脚本片段）

实现sql语句的复用

1、先用sql标签提取出公共的sql片段

```xml
<sql id="updateInfo">
    <if test="title != null">title = #{title},</if>
    <if test="author != null">author = #{author},</if>
    <if test="createTime != null">create_time = #{createTime},</if>
    <if test="views != null">views = #{views}</if>
</sql>
```

2、需要使用时，用include标签调用

```xml
<update id="updateBlog" parameterType="map">
    update blog
    <set>
        <include refid="updateInfo">
    </set>
     where id = #{id}
</update>
```



注：最好只提取基于单表的sql片段；不要把<where>、<set>等会自动优化sql语句的标签提取到sl片段，会失效



### 4.11 缓存

- 把查询结果存在缓存中，再需要时直接取用，以减少和数据库的交互，节约性能，提高系统效率，适应高并发问题

- 适合缓存的数据：经常查询且不常改变的数据

- MyBatis 内置了一个强大的事务性查询缓存机制，可以方便地配置和定制缓存。默认定义了二级缓存



#### 4.11.1 一级缓存

也叫本地缓存，默认开启

SqlSession级别，与数据库的一次会话期间查询到的数据保存到本地缓存中。

- 映射语句文件中的所有 select 语句的结果将会被缓存。
- 映射语句文件中的所有 insert、update 和 delete 语句会刷新所有缓存。
- 缓存使用最近最少使用算法（LRU, Least Recently Used）算法来清除不需要的缓存。
- 缓存不会定时进行刷新（也就是说，没有刷新间隔）。



#### 4.11.2 二级缓存

1、实现

- 先在核心配置文件中开启缓存（虽然默认就是开启）

```xml
<!--设置-->
<settings>
    <setting name="logImpl" value="STDOUT_LOGGING"/>
    <setting name="cacheEnabled" value="true"/>
</settings>
```

- 再序列化实体类

```java
import java.io.Serializable;

@Data
public class User implements Serializable {
    private int id;
    private String name;
    private String password;
}
```

- 在实现接口的Mapper.xml文件中进行设置：

```xml
<cache/>
```

​		也可以进行详细设置

```xml
<cache
  eviction="FIFO"
  flushInterval="60000"
  size="512"
  readOnly="true"/>
```

​		eviction（清除策略）：最近最少使用LRU、先进先出FIFO、软引用SOFT、弱引用WEAK

​		flushInterval（刷新间隔）：单位毫秒

​		size（最多的缓存数目）：默认1024

​		readOnly（只读）属性可以被设置为 true 或 false。默认false



​		也可以对单个的sql语句进行设置

```xml
<!--select可以设置用不用缓存-->
<select id="getUserById" parameterType="int" resultType="user" useCache="false">  
    select * from user where id=#{id}  
</select>

<!--insert、update、delete可以设置是否刷新缓存-->
<update id="updateUser" flushCache="false">
    update user set id=1  
</update>
```

2、工作机制

- **namespacce级别**，不同的mapper查询的数据放在各自对应的二级缓存中

- 二级缓存保存的内容从一级缓存中获取
  - 一个会话查询一条数据，数据被保存在该对话的一级缓存中；
  - 该会话关闭时，清空其一级缓存，将其中的内容保存到二级缓存中；
  - 新的会话查询可以从二次缓存中获取内容；
  - （如果会话未关闭，其一级缓存就不会上传到二级缓存，其他会话便访问不到）

  

  **查询的时候，先看二级缓存中有没有，再看一级缓存中有没有，都没有再进行数据库操作**

  

#### 4.11.3 自定义缓存Ehcache

EhCache 是一个纯Java的进程内缓存框架，是一种广泛使用的开源分布式缓存。

https://mvnrepository.com/artifact/org.mybatis.caches/mybatis-ehcache

用法：

```
在<cache type=""/> 
```





## 5 Spring

Spring是为了解决企业级应用开发的复杂性而创建的，简化开发



### 5.1 简介

Spring是一个开源的免费框架（容器），是轻量级、非入侵式（不会导致原本的源码无法运行）的框架

**核心思想：控制反转（IOC），面向切面编程（AOP）**

支持事务的处理，对框架整合的支持

综上：**spring是一个轻量级的控制反转(IOC)和面向切面编程(AOP)的框架**

弊端：目前发展得过于庞大，后期配置繁琐



Spring有如下7个模块构成：

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211204151611512.png" alt="image-20211204151611512" style="zoom:80%;" />

SpringBoot：一个快速开发的脚手架，可以快速开发单个微服务

Spring Cloud：基于SpringBoot



官方文档：https://docs.spring.io/spring-framework/docs/current/reference/html/

官方下载地址：https://repo.spring.io/ui/native/release/org/springframework/spring/

Github：https://github.com/spring-projects/spring-framework

Maven

```xml
<!-- https://mvnrepository.com/artifact/org.springframework/spring-webmvc -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>5.3.13</version>
</dependency>

<!-- https://mvnrepository.com/artifact/org.springframework/spring-jdbc -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>5.3.13</version>
</dependency>

```



### 5.2 控制反转IoC



#### 5.2.1 Ioc概念

控制反转IoC (Inversion of Control)，是一种设计思想，通过描述（xml或注解）和第三方去获取特定对象。

在Spring中由IoC容器实现，实现方法是依赖注入(Denpendency Injection, DI)

> DAO层

接口类

```java
public interface UserDao {
    public void getUser();
}
```

实现类

```java
public class UserDaoImpl implements UserDao{
    public void getUser(){}
}
```



> 业务层

接口类

```java
public interface UserService {
    void getUser();
}
```

实现类

```java
// 原来
public class UserServiceImpl implements UserService{

    private UserDao userDao = new UserDaoImpl(); 

    public void getUser(){
        userDao.getUser();
    }
}

// 依赖注入：用set动态注入
public class UserServiceImpl implements UserService{

    private UserDao userDao;
    
    public void setUser(UserDao user){
        userDao = user;
    }
    public void getUser(){
        userDao.getUser();
    }
}
```

- 原来：程序主动创建对象，主动权在程序员，用户需求变更时（如更换数据库）程序员需要手动创建新的dao层实现类，并在业务层也要创建与之对应的业务类。

- 依赖注入：使用set注入后，程序不再具有主动性，而变成了被动的接受对象，程序员再新建dao层实现类后，不再需要修改业务层

依赖注入是实现IoC的方法之一，所谓控制反转，就是获得依赖对象的方式反转了



#### 5.2.2 Spring中实际的写法

> DAO层

接口类

```java
public interface UserDao {
    public void getUser();
}
```

实现类

```java
public class UserDaoMysqlImpl implements UserDao{
    public void getUser(){ System.out.println("mysql");}
}

public class UserDaoOracleImpl implements UserDao{
    public void getUser(){ System.out.println("oracle");}
}
```



> 业务层

接口类

```java
public interface UserService {
    void setUserDao(UserDao userDao);
    void getUser();
}
```

实现类

```java
public class UserServiceImpl implements UserService{

    private UserDao userDao;

    // set方法由Spring自动调用，进行依赖注入，必须有
    public void setUserDao(UserDao userDao){
        this.userDao = userDao;
    }

    public void getUser(){
        userDao.getUser();
    }

}
```

resources下的xml配置文件beans.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <!--使用Spring创建的对象，在Spring中称为bean-->
    <!--bean标签相当于创建一个对象，id为变量名，class为对应类-->
    <bean id="mysqlImpl" class="com.ghy.dao.UserDaoMysqlImpl"/>
    <bean id="oracleImpl" class="com.ghy.dao.UserDaoOracleImpl"/>

    <bean id="userServiceImpl" class="com.ghy.service.UserServiceImpl">
        <!--property为对象中的属性，name属性名；value是值；ref指定了该属性和别的Bean的引用关系，是别的Bean的id-->
        <property name="userDao" ref="userDaoImpl"/>  
        <!-- <property name="userDao" ref="oracleImpl"/>-->
    </bean>

</beans>
```

Spring的Ioc：对象由Spring创建，管理，装配。

需要创建对象时，从专门的对象中getBean()，不再自己new

```java
// 获取ApplicationContext代替new
ApplicationContext contact = new ClassPathXmlApplicationContext("beans.xml");
UserServiceImpl userServiceImpl = (UserServiceImpl)contact.getBean("userServiceImpl");
userServiceImpl.getUser();
```

我们发现，需要更换UserDao接口的实现类时，只需要改变xml配置文件中该属性对应的property的ref即可。



> 理解：

本质是set注入，因此业务层实现类中必须有对应的set方法，参数是DAO接口类。使用对象时，Spring自动调用set方法，并把ref设置的实现类的对象作为参数传入赋值，即用该实现类实现了接口。

（因为由Spring自动进行接口的实现，所以**业务层中**对应的地方用的都是DAO层的接口类，**不会出现DAO层的实现类**）



#### 5.2.3 IoC创建对象的方式

1、对应xml配置文件中所有的bean在获取ApplicationContext时被创建

2、使用无参构造函数，如果有property标签，用set赋值

3、一个bean只创建了一个，多次获取同一个bean是**浅拷贝**

使用有参构造的方式

```xml
<!--下标赋值-->
<bean id="user" class="com.ghy.pojo.User">
    <constructor-arg index="0" value="西琳"/>
</bean>

<!--变量类型赋值-->
<bean id="user" class="com.ghy.pojo.User">
    <constructor-arg type="java.lang.String" value="西琳"/>
</bean>

<!--变量名赋值-->
<bean id="user" class="com.ghy.pojo.User">
    <constructor-arg name="name" value="西琳"/> <!--和property不同，这个是有参构造，之前的是无参构造+set-->
</bean>
```

需要对应的有参构造函数：

下标赋值指定构造函数形参的下标，变量类型赋值针对形参的类型（有相同的时按形参顺序），变量名赋值按形参变量名





### 5.3 Spring配置

#### 5.3.1 别名

给bean取别名，getBean时用别名或原来的id都行

- 方法1:alias

```xml
<bean id="user" class="com.ghy.pojo.User">
    <property name="name" value="西琳"/>
</bean>

<alias name="user" alias="user2"/>
```

- 方法2: bean name

```xml
<bean id="user" class="com.ghy.pojo.User" name="user2,u">
    <property name="name" value="西琳"/>
</bean>
```

方法2可取多个别名，每个别名之间用","隔开



#### 5.3.2 import

可以将多个配置文件导入并合并。

多用与团队开发

```xml
<import resource="applicationContext.xml"/>
```

会冲突的内容，后导入的会覆盖之前的

相同对象但未冲突的（如多个别名），会合并



### 5.4 依赖注入

依赖：对象依赖于容器

注入：对象的属性由容器注入

#### 5.4.1 构造器注入

见5.2.3使用有参构造函数



#### 5.4.2 set注入（重要）

默认方法

```xml
<bean id="student" class="com.ghy.pojo.Student">
    
    <!--一般类型-->
    <property name="name" value="爱丽丝"/>
    
    <!--对象-->
    <property name="address" ref="address"/>
    
    <!--数组-->
    <property name="books">
        <array>
            <value>基督山伯爵</value>
            <value>罪与罚</value>
        </array>
    </property>
    
    <!--列表-->
    <property name="hobbys">
        <list>
            <value>小提琴</value>
            <value>switch</value>
        </list>
    </property>
    
    <!--map-->
    <property name="card">
        <map>
            <entry key="身份证" value="123"/>
            <entry key="银行卡" value="19870287"/>
        </map>
    </property>
    
    <!--set-->
    <property name="games">
        <set>
            <value>宝可梦</value>
            <value>fgo</value>
        </set>
    </property>
    
    <!--注入null-->
    <property name="wife">
        <null/>
    </property>
    
    <!--property-->
    <property name="info">
        <props>
            <prop key="driver">jdbc</prop>
            <prop key="url">localhost</prop>
        </props>
    </property>
</bean>
```



#### 5.4.3 扩展方式注入

1、p-命名空间注入 (简化set注入)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="address" class="com.ghy.pojo.Address" p:address="darby" p:address-ref=""/>
</beans>
```

在beans标签中增加 xmlns:p="http://www.springframework.org/schema/p" 后，可以用 p:属性名 直接在bean标签上赋值。

仅限一般数据类型属性和对象属性。



2、c-命名空间注入 (简化构造器注入)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:c="http://www.springframework.org/schema/c"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="address" class="com.ghy.pojo.Address" c:address="darby" c:address-ref=""/>
```

在beans标签中增加 xmlns:c="http://www.springframework.org/schema/c" 后，可以用 c:属性名 直接在bean标签上赋值。

类中要有对应的有参构造函数，仅限一般数据类型属性和对象属性。



#### 5.4.4 Bean作用域

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211205145135506.png" alt="image-20211205145135506" style="zoom: 50%;" />

```xml
<bean id="address" class="com.ghy.pojo.Address" scope="singleton"/>
```

在bean标签的scope属性中设置。

| 作用域    |      | 说明                                                   |
| --------- | ---- | ------------------------------------------------------ |
| singleton | 单例 | 默认设置，多次取同一个bean是在共享同一个bean（浅拷贝） |
| prototype | 原型 | 每次getBean会创建一个新的对象                          |

其余的在web开发中使用



### 5.5 Bean的装配

Spring在上下文自动查找和给bean装配对象属性。

（注入可以指设置任何属性，装配一般就指对象属性）



Spring的装配方式

- xml显式装配
- java显式装配
- 隐式自动装配



#### 5.5.1 xml显式装配

普通的方式，见5.2.3的构造器设置和5.4.2的set注入

```xml
<bean id="cat" class="com.ghy.pojo.Cat"/>
<bean id="dog" class="com.ghy.pojo.Dog"/>

<bean id="human" class="com.ghy.pojo.Human">
    <property name="name" value="温迪戈"/>
    <property name="cat" ref="cat"/>
    <property name="dog" ref="dog"/>
</bean>
```



#### 5.5.2 隐式自动装配

1、byName：在bean标签的autowire属性中设置

```xml
<bean id="cat" class="com.ghy.pojo.Cat"/>
<bean id="dog" class="com.ghy.pojo.Dog"/>

<bean id="human" class="com.ghy.pojo.Human" autowire="byName">
    <property name="name" value="温迪戈"/>
</bean>
```

Spring会自动地在bean中查找。如果满足：

- bean对应类有：setCat(Cat cat)方法
- set方法set后的内容和对应bean的id一致（set后首字母大小写不影响）
- set方法的形参名和对应bean的id完全一致（大小写敏感）

就会把该bean自动填充给对应的对象属性。

限制：因为对set方法有要求，而set方法一般不会改，所以xml中有多个同类的bean时，一般只能固定装配一个；

​			xml中所有的bean id 必须唯一。



2、byType：在bean标签的autowire属性中设置

```xml
<bean class="com.ghy.pojo.Cat"/>
<bean class="com.ghy.pojo.Dog"/>

<bean id="human" class="com.ghy.pojo.Human" autowire="byType">
    <property name="name" value="温迪戈"/>
</bean>
```

Spring会自动地在bean中查找。如果对应bean的类和该对象属性的类一致的话，自动填充。（甚至不用写bean id）

限制：强制xml文件中一个类只能有一个bean，否则报错。



#### 5.5.3 java显式装配 : 注解

使用注解前，要在beans标签中进行配置。在beans属性中进行设置 ，再加上<context:annotation-config/>标签；

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <!--开启注解支持-->
    <context:annotation-config/>
    
</beans>
```

然后在类对应的属性或set方法上加标注即可

##### @Autowired

```java
public class Human {

    private String name;

    //@Autowired
    private Cat cat;

    //@Autowired
    private Dog dog;
    
    @Autowired
    public void setCat(Cat cat) {
        this.cat = cat;
    }

    @Autowired
    public void setDog(Dog dog) {
        this.dog = dog;
    }
}
```

机制首先类似autowire="byType"，按bean的类来，存在多个同类bean时，byType无法匹配了，再按autowire="byName"方法看id。

用反射实现，给属性加@Autowired后不再需要类的set方法



再加上@Qualifier()可以指定填充的bean的id

```java
@Autowired
@Qualifier("dog2")  // 或 @Qualifier(value = "dog2")
private Dog dog;
```



##### @Resource 

```java
@Resource
private Cat cat;

@Resource(name="dog2") // 也可以指定 bean id
private Dog dog;
```

jdk的注解 @Resource 也可以实现自动装配，与@Autowired相反，先找 bean id 有没有匹配的，没有再找同类的

限制：而有多个同类但id不同的bean，就会报错，无法匹配



（有jdk版本兼容问题，高版本jdk可能没有）



### 5.6 注解

- xml：万能，维护简单

- 注解：只有本类才能使用自己的注解，维护相对复杂
  - 例如，不能实现<property ref="其他类对象">



Spring4后需要导入aop的jar包（一般spring-webmvc包会包含）

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211205170520827.png" alt="image-20211205170520827" style="zoom:80%;" />

配置文件中需要添加**注解驱动**和**包扫描**

```xml
<!--注解驱动-->
<context:annotation-config/>

<!--扫描指定包，使其中的注解生效-->
<context:component-scan base-package="com.ghy"/>
```



#### 5.6.1 获取Bean

**@Component**

```java
@Component
public class User {}
```

组件，放在类上，代替xml文件的创建的bean，自动生成的bean的id是类名（首字母小写）



> 衍生标签（和@Component功能一样，但不同层的最好用自己对应的标签）

- @Repository（对应DAO层）

- @Service（对应Service层）
- @Controller（对应Controller层）

将不同层的类装配到Spring容器中



#### 5.6.2 Bean属性注入

**@Value**

```java
@Component
public class User {

    @Value("ghy")
    private String name;
    
    // @Value("ghy")
    public void setName(String name) {
        this.name = name;
    }
}
```

用注解给变量赋值（相当于bean的property标签），不需要set方法，但有set方法时也可以注在set方法上

适合简单的变量



#### 5.6.3 Bean自动装配

见5.5.3 java显式装配 : 注解



#### 5.6.4 作用域

填写选项同5.4.4 Bean作用域



### 5.7 Java实现Spring配置

用java代替Spring的xml配置文件。

在与pojo目录同级的位置新建config目录，保存Config类



#### 方法一：@Bean

```java
@Configuration  // 说明是配置；类
public class ghyConfig {
 
    @Bean       // 用@Bean标记创建bean
    public User getUser(){
        return new User();
    }
}
```

@Bean相当于xml的bean标签，方法名相当于bean id，方法返回值相当于bean class

```java
public class User {
    private String name;

    public String getName() {
        return name;
    }

    @Value("西琳")
    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                '}';
    }
}
```

实体类中可以用@Value进行注入

```java
public class MyTest {
    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(ghyConfig.class);
        User user = (User) context.getBean("getUser");
        System.out.println(user.hashCode());
    }
}
```

使用时先获取容器 new AnnotationConfigApplicationContext(ghyConfig.class);

再用方法名getBean，就可以得到对象。



**方法二：@Component**

```java
@Configuration
@ComponentScan("com.ghy.pojo")  // 
public class ghyConfig {
}
```

设置类中用 @ComponentScan 对目标类所在目录进行扫描

```java
//@Component
public class User {
    private String name;

    public String getName() {
        return name;
    }

    @Value("西琳")
    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                '}';
    }
}
```

实体类上用@Component将类录入Spring

```java
public class MyTest {
    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(ghyConfig.class);
        User user = (User) context.getBean("user");
        System.out.println(user.hashCode());
    }
}
```

使用时先获取容器 new AnnotationConfigApplicationContext(ghyConfig.class);

再用类名（首字母小写）getBean，就可以得到对象。



**（@Component 、 @ComponentScan）和@Bean用一组即可。**



### 5.8 代理模式

代理模式是SpringAOP的底层



#### 5.8.1 静态代理

角色分析:

- 抽象角色：要完成的功能，一般用接口或抽象类表示
- 真实角色：完成功能的实体类
- 代理角色：代理真实角色，代理后会做一些附属操作
- 客户：访问代理对象以执行功能



代理模式的优点

- 业务分工：让真实角色专注与本身的操作，公共操作交给代理角色
- 公共业务发生拓展时，方便集中管理



代码：

1、接口

```java
// 抽象角色（出租房屋）
public interface Rent {
    public void rent();
}
```

2、真实角色

```java
// 真实角色（房东）
public class Host implements Rent{
    @Override
    public void rent() {
        System.out.println("房东租房子");
    }
}
```

3、代理角色

```java
// 代理角色（中介）
public class Proxy implements Rent{

    private Host host;

    public Proxy() {
    }

    public Proxy(Host host) {
        this.host = host;
    }

    @Override
    public void rent() {
        seeHouse();
        host.rent();
        contract();
        fee();
    }

    // 看房
    public void seeHouse(){
        System.out.println("中介看房");
    }

    // 签合同
    public void contract(){
        System.out.println("中介签合同");
    }

    // 看房
    public void fee(){
        System.out.println("中介收中介费");
    }
}
```

4、客户端访问代理角色

```java
// 客户（房客）
public class Client {
    public static void main(String[] args) {
        Host host= new Host();

        // 代理
        Proxy proxy = new Proxy(host);
        proxy.rent();
    }
}
```

 

静态代理的缺点：一个实现类要对应一个代理类，程序量翻倍。

为避免这个缺点，进行动态代理



#### 5.8.2 动态代理

底层：反射

角色同静态代理，但实际代理类不是提前定义好的，是动态生成的，一个动态代理类能代理一类接口，即代理一类业务

分类：基于接口（如**JDK动态代理**）、基于类（cglib）、java字节码（JAVAssist）



代理类Proxy，调用处理程序InvocationHandler

1、接口

```java
// 抽象角色（出租房屋）
public interface Rent {
    public void rent();
}
```

2、接口实现类

```java
// 真实角色（房东）
public class Host implements Rent {
    @Override
    public void rent() {
        System.out.println("房东租房子");
    }
}
```

**3、代理类（核心）**

```java
// 用此类自动生成代理类
public class ProxyInvocationHandler implements InvocationHandler {

    // 被代理的接口
    private Object object;

    public void setObject(Object object) {
        this.object = object;
    }

    // 生成代理类
    public Object getProxy(){
        return Proxy.newProxyInstance(this.getClass().getClassLoader(), object.getClass().getInterfaces(), this);
    }

    // 代理调用方法的执行，返回结果
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        seeHouse();

        // 动态代理的本质，是反射
        Object result = method.invoke(object, args);

        contract();
        fee();
        return null;
    }

    // 代理角色特有操作
    // 看房
    public void seeHouse(){
        System.out.println("中介看房");
    }

    // 签合同
    public void contract(){
        System.out.println("中介签合同");
    }

    // 看房
    public void fee(){
        System.out.println("中介收中介费");
    }
}
```

4、实际调用类

```java
// 客户（房客）
public class Client {
    public static void main(String[] args) {

        // 真实角色
        Host host = new Host();

        // 代理角色
        ProxyInvocationHandler pih = new ProxyInvocationHandler();
        pih.setObject(host);   // 设置要代理的对象
        Rent proxy = (Rent) pih.getProxy();  // 动态生成的代理

        proxy.rent();
    }
}
```



代理类中除特有操作以外的部分都是通用的，实际调用类在需要对象时，主动设置其代理的类；

对于同一类业务的不同实现类，可用一个代理类进行代理；

不同类的业务，代理类的差别基本也只有特有操作不同。



### 5.9 面向切面编程 AOP

面向切面编程 (Aspect Oriented Programming, AOP)

AOP在Spring中可以提供声明式事务，允许用户自定义切面

![image-20211206171052045](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211206171052045.png)

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211206171118795.png" alt="image-20211206171118795" style="zoom:67%;" />



通知类型：

- Before：作为切入点的method执行前

- After：作为切入点的method执行后

- Around：作为切入点的method执行前后
- AfterReturning：作为切入点的method返回后
- AfterThrowing：作为切入点的method抛出异常后



顺序：Around前、Before、执行method、AfterReturning、Around后、After（Around后、After顺序不一定）

#### 5.9.1 方法一：原生Spring API接口

1、导入依赖

```xml
<!-- https://mvnrepository.com/artifact/org.aspectj/aspectjweaver -->
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.9.7</version>
    <scope>runtime</scope>
</dependency>
```

2、接口

```java
public interface UserService {
    public void add();
    public void delete();
    public void update();
    public void select();
}
```

3、实现类

```java
public class UserServiceImpl implements UserService{
    @Override
    public void add() {
        System.out.println("增加一个用户");
    }

    @Override
    public void delete() {
        System.out.println("删除一个用户");
    }

    @Override
    public void update() {
        System.out.println("修改一个用户");
    }

    @Override
    public void select() {
        System.out.println("查询一个用户");
    }
}
```

4、日志（切面要实现的功能）

```java
public class Log implements MethodBeforeAdvice {

    //本方法会在method执行前被调用
    // method要执行的方法，args参数，target目标对象
    @Override
    public void before(Method method, Object[] args, Object target) throws Throwable {
        System.out.println(target.getClass().getName() + "的" + method.getName() + "被调用了");
    }
}

public class AfterLog implements AfterReturningAdvice {

    //本方法会在method返回后被调用
    // returnValue为method执行的返回值，method要执行的方法，args参数，target目标对象
    @Override
    public void afterReturning(Object returnValue, Method method, Object[] args, Object target) throws Throwable {
        System.out.println(target.getClass().getName() + "的" + method.getName() + "执行了，结果为：" + returnValue);
    }
}
```

5、配置文件 ApplicationContext.xml

要导入aop约束，并配置aop

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">   <!--导入aop约束-->

    <!--注册Bean-->
    <bean id="userService" class="com.ghy.service.UserServiceImpl"/>
    <bean id="log" class="com.ghy.log.Log"/>
    <bean id="afterLog" class="com.ghy.log.AfterLog"/>

    <!--方法一：使用原生Spring API接口-->

    <!--配置aop-->
    <aop:config>
        <!--切入点修饰词-->
        <aop:pointcut id="pointcut1" expression="execution(* com.ghy.service.UserServiceImpl.*(..))"/>

        <!--执行环绕-->
        <aop:advisor advice-ref="log" pointcut-ref="pointcut1"/>
        <aop:advisor advice-ref="afterLog" pointcut-ref="pointcut1"/>
    </aop:config>

</beans>
```

expression="execution(* com.ghy.service.UserServiceImpl.*(..))"

- 第1个 *：表示返回值类型，号表示所有的类型。

- 包名：com.ghy.service.UserServiceImpl 类/包下所有类的方法。
  - UserServiceImpl..* 类或所有包/子包下所有类的方法

- (..)：表示方法的参数，两个句点表示任何参数。



6、实际使用

```java
public class MyTest {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("ApplicationContext.xml");
        UserService userService = context.getBean("userService", UserService.class);
        userService.add();
    }
}
```

 getBean返回值要强转成接口类的对象。因为aop的本质是代理，context.getBean会返回代理类，由于代理类和实现类都实现了同一个接口，强转成实现类会有接口冲突。



#### 5.9.2 方法二：自定义类（定义切面）

只有xml配置文件中aop的设置和方法一不同

- 自定义的日志类（切面要实现的功能）

```java
public class PointCut {

    public void before(){
        System.out.println("=========前=========");
    }
    public void after(){
        System.out.println("=========后=========");
    }
}
```

- xml配置文件中aop的设置

```xml
<!--方法二：自定义类-->
<bean id="pointCut" class="com.ghy.diy.PointCut"/>  <!--自定义的处理类-->
<aop:config>
    <!--自定义切面-->
    <aop:aspect ref="pointCut">
        <!--切入点-->
        <aop:pointcut id="pointcut1" expression="execution(* com.ghy.service.UserServiceImpl.*(..))"/>
        <!--通知-->
        <aop:before method="before" pointcut-ref="pointcut1"/>
        <aop:after method="after" pointcut-ref="pointcut1"/>
    </aop:aspect>
</aop:config>
```



方法一功能更强大



#### 5.9.3 方法三：注解

和方法二类似，只是将原本xml的设置用注解实现

- 自定义的日志类（切面要实现的功能）

```java
// 用注解实现AOP
@Aspect  // 标注此类为切面
public class AnnotationPointCut {

    @Before("execution(* com.ghy.service.UserServiceImpl.*(..))")  // 切入点
    public void before(){
        System.out.println("=========前=========");
    }

    @After("execution(* com.ghy.service.UserServiceImpl.*(..))")  // 切入点
    public void after(){
        System.out.println("=========后=========");
    }
    
    @AfterReturning("execution(* com.ghy.service.UserServiceImpl.*(..))") // 切入点
    public void afterReturning(){
        System.out.println("=========返=========");
    }

    @Around("execution(* com.ghy.service.UserServiceImpl.*(..))")  // 切入点
    public void around(ProceedingJoinPoint jp) throws Throwable {  
        System.out.println("=========环前=========");
        Signature signature = jp.getSignature();  // 获取签名
        Object proceed = jp.proceed();            // 代表method的执行
        System.out.println("=========环后=========");
    }
}
```

ProceedingJoinPoint为连接点，用于around中可以分隔环绕前后，也可以获取签名

- xml配置文件中的设置

```xml
<!--方法三：自定义类-->
<bean id="annotationPointCut" class="com.ghy.diy.AnnotationPointCut"/>
<!--开启注解支持-->
<aop:aspectj-autoproxy/>
<!--<aop:aspectj-autoproxy proxy-target-class="true"/>-->  <!--默认false，true时用cglib实现-->
```



### 5.10 整合MyBatis

要导入的包

- junit
- mybaits
- mysql相关包
- spring相关包
- aop织入
- mybatis-spring

```xml
<dependencies>

    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
    </dependency>

    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis</artifactId>
        <version>3.5.7</version>
    </dependency>
    
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.25</version>
    </dependency>
    
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>5.3.13</version>
    </dependency>
    
    <!-- https://mvnrepository.com/artifact/org.aspectj/aspectjweaver -->
    <dependency>
        <groupId>org.aspectj</groupId>
        <artifactId>aspectjweaver</artifactId>
        <version>1.9.7</version>
        <scope>runtime</scope>
    </dependency>
    
    <!-- https://mvnrepository.com/artifact/org.springframework/spring-jdbc -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-jdbc</artifactId>
        <version>5.3.13</version>
    </dependency>
    
    <!-- https://mvnrepository.com/artifact/org.mybatis/mybatis-spring -->
    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis-spring</artifactId>
        <version>2.0.6</version>
    </dependency>

</dependencies>
```



#### 5.10.1 方法一

##### 1、实体类和接口

```java
@Data
public class User {
    private int id;
    private String name;
    private String pwd;
}
```

```java
public interface UserMapper {
    List<User> selectUser();
}
```

##### 2、数据源、sqlSessionFactory、sqlSession

spring-dao.xml

```xml
<?xml version="1.0" encoding="UTF8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <!--Data Source:使用Spring的数据源，不再需要MyBatis的数据源设置-->
    <!--只要是数据源就可行，如C3P0，DBCP，Druid等，都只需要设好id、class和property即可-->
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"/>
        <property name="url" value="jdbc:mysql://localhost:3306/mybatis?useSSL=true&amp;useUnicode=true&amp;characterEncoding=UTF-8&amp;serverTimezone=UTC"/>
        <property name="username" value="root"/>
        <property name="password" value="001118"/>
    </bean>

    <!--sqlSessionFactory-->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource" />
        <!--绑定MyBatis配置文件-->
        <property name="configLocation" value="classpath:mybatis-config.xml"/>
        <property name="mapperLocations" value="classpath:com/ghy/mapper/*.xml"/>
    </bean>

    <!--SqlSessionTemplate: 我们使用的sqlSession-->
    <bean id="sqlSession" class="org.mybatis.spring.SqlSessionTemplate">
        <!--因为SqlSessionTemplate没有set方法，所以只能使用构造器注入-->
        <constructor-arg index="0" ref="sqlSessionFactory"/>
    </bean>
    
</beans>
```

- 用Spring的Bean实现sqlSessionFactory和sqlSession，不再需要new他们的对象，更不需要实现此功能的工具类了

- sqlSessionFactory的property可实现原Mybatis核心配置文件的所有功能，但也可以用`<property name="configLocation" value="">` 引入写好的Mybatis核心配置文件
- sqlSession的Bean用SqlSessionTemplate代替实现，使用构造器注入方法注入之前定义的sqlSessionFactory



mybatis-config.xml（虽然可完全用sqlSessionFactory的Bean的property实现，但是还是可以单独写一个，单独存放一些设置）

```xml
<?xml version="1.0" encoding="UTF8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<!--核心配置文件-->
<configuration>

    <!--给实体类其别名-->
    <typeAliases>
        <package name="com.ghy.pojo"/>
    </typeAliases>
    
</configuration>
```

##### 3、接口的实现类

仍用与接口同名的xml文件实现sql

UserMapper.xml

```xml
<?xml version="1.0" encoding="UTF8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.ghy.mapper.UserMapper">

    <select id="selectUser" resultType="User">
        select * from mybatis.user
    </select>
</mapper>
```

因为Spring在service层，要再加一层进行代理，将对应操作封装成函数，只返回sql操作的结果。

UserMapperImpl.java

```java
public class UserMapperImpl implements UserMapper{

    // 代替SqlSession执行所有操作
    private SqlSessionTemplate sqlSession;

    public void setSqlSession(SqlSessionTemplate sqlSession) {
        this.sqlSession = sqlSession;
    }

    @Override
    public List<User> selectUser() {
        UserMapper mapper = sqlSession.getMapper(UserMapper.class);
        List<User> users = mapper.selectUser();
        //sqlSession.close();  //Spring不允许
        return users;
    }
}
```

Spring中不能也不允许手动关闭sqlSession

##### 4、Spring核心设置文件

ApplicationContext.xml

```xml
<?xml version="1.0" encoding="UTF8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">   <!--导入aop约束-->

    <!--引入Spring数据库设置-->
    <import resource="spring-dao.xml"/>

    <bean id="userMapper" class="com.ghy.mapper.UserMapperImpl">
        <property name="sqlSession" ref="sqlSession"/>
    </bean>

</beans>
```

ApplicationContext.xml是Spring的总设置文件，其他设置文件需要import进来，负责实际用户使用的Bean的定义

##### 5、实际使用

```java
@Test
public void test() {
    ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
    UserMapper userMapper = context.getBean("userMapper",UserMapper.class);
    List<User> users = userMapper.selectUser();
    for(User user: users){
        System.out.println(user);
    }
}
```



#### 5.10.2 方法二 SqlSessionDaoSupport类

修改代理层新加的一层实现类UserMapperImpl.java

```java
public class UserMapperImpl extends SqlSessionDaoSupport implements UserMapper{
    @Override
    public List<User> selectUser() {
        return getSqlSession().getMapper(UserMapper.class).selectUser();
    }
}
```

方法一是在类中手动定义了sqlSession，并用set注入。

也可以让本类继承SqlSessionDaoSupport类，这样用该类的getSqlSession()方法就可以直接得到sqlSession了。

也就是说不再需要下述的sqlSession Bean了。

```xml
<bean id="sqlSession" class="org.mybatis.spring.SqlSessionTemplate">
    <constructor-arg index="0" ref="sqlSessionFactory"/>
</bean>
```

但是sqlSession仍需要和sqlSessionFactory相联系，所以要在该类的Bean中直接注入sqlSessionFactory（继承的父类的属性）

```java
<bean id="userMapper" class="com.ghy.mapper.UserMapperImpl2">
    <property name="sqlSessionFactory" ref="sqlSessionFactory"/>
</bean>
```



相比于方法一，省去了sqlSession的一步。



### 5.11 事务

为了实现数据库事务的ACID特性，必须要实现事务，反例：

```java
@Override
public List<User> selectUser() {
    UserMapper mapper = getSqlSession().getMapper(UserMapper.class);
    mapper.addUser(new User(5,"po","123098"));
    mapper.deleteUser(5);
    return mapper.selectUser();
}
```

```xml
<delete id="deleteUser" parameterType="int">
    deletea from user where id=#{id}
</delete>
```

selectUser()方法中调用了多个操作，其中deleteUser操作出错了，如果未实现事务，在其之前的addUser仍能成功地向表中添加数据，这显然和事务的原子性相矛盾，会导致数据不一致等问题。



事务分类：

- 声明式事务：如aop，事务不影响原有代码**（推荐）**

- 编程式事务：再原有代码中增加事务代码



#### 5.11.1 Spring AOP实现声明式事务

先在数据库配置文件的beans标签属性中引入事务tx和面向切面aop依赖

spring-dao.xml

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:tx="http://www.springframework.org/schema/tx" 
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd
        http://www.springframework.org/schema/tx
        https://www.springframework.org/schema/tx/spring-tx.xsd">
</beans>
```

spring-dao.xml

```xml
<!--结合AOP实现事务的织入-->
<!--配置事务通知-->
<tx:advice id="txAdvice" transaction-manager="transactionManager">
    <!--给哪些方法配置事务-->
    <tx:attributes>
        <!--配置事务的传播特性-->
        <tx:method name="add" propagation="REQUIRED"/>
        <tx:method name="delete" propagation="REQUIRED"/>
        <tx:method name="update" propagation="REQUIRED"/>
        <tx:method name="query" read-only="true"/>
        <!--<tx:method name="*" propagation="REQUIRED"/>-->
    </tx:attributes>
</tx:advice>

<!--配置事务切入-->
<aop:config>
    <aop:pointcut id="txPointCut" expression="execution(* com.ghy.mapper.*.*(..))"/>
    <!--执行环绕-->
    <aop:advisor advice-ref="txAdvice" pointcut-ref="txPointCut"/>
</aop:config>
```

- 用tx:advice标签配置事务；

- tx:method的name可以设为add/delete/update/query，分别对应增删改查，或也可以用*代表所有操作；

- tx:method的propagation的值有7种，默认为REQUIRED

  - ```
    REQUIRED：支持当前事务，如果当前没有事务，就新建一个事务。这是最常见的选择。 
    SUPPORTS：支持当前事务，如果当前没有事务，就以非事务方式执行。 
    MANDATORY：支持当前事务，如果当前没有事务，就抛出异常。 
    REQUIRES_NEW：新建事务，如果当前存在事务，把当前事务挂起。 
    NOT_SUPPORTED：以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。 
    NEVER：以非事务方式执行，如果当前存在事务，则抛出异常。 
    NESTED：支持当前事务，如果当前事务存在，则执行一个嵌套事务，如果当前没有事务，就新建一个事务。
    ```

- tx:method的read-only为设置操作的只读性，一般设给query的条目，避免查询操作修改数据库

- 之后按5.9.1方法实现上述事务的aop

  - ```
    execution(* com.ghy.mapper.*.*(..))
    ```

  - 含义：对于，com.ghy.mapper包下所有类的返回值为任意类的所有方法，参数任意，aop有效



## 6 SpringMVC

MVC框架的作用

- 将url映射到java类或java类的方法
- 封装客户端提交的数据
- 处理客户端请求，调用相关的业务处理，封装响应数据
- 将响应的数据进行渲染，在表示层返回并显示



SpringMVC的特点

- 简单易学，简洁灵活
- 高效，基于响应和请求
- 与Spring兼容性好
- 约定大于配置
- 功能强大，如数据验证、RESTful等



**SpringMVC以请求为驱动，围绕一个中心servlet--DispatcherServlet分派请求及提供其他功能。DispatcherServlet继承自Servlet，本质上也是一个Servlet类**



重点：SpringMVC执行流程



### 6.1 第一个SpringMVC程序

1、新建module，添加web支持；设置好Tomcat

2、导入SpringMVC依赖（可以导到父工程）

```xml
<dependencies>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
    </dependency>
    
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>5.3.13</version>
    </dependency>
    
    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>javax.servlet-api</artifactId>
        <version>4.0.1</version>
    </dependency>
    
    <dependency>
        <groupId>javax.servlet.jsp</groupId>
        <artifactId>jsp-api</artifactId>
        <version>2.2</version>
    </dependency>

    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>jstl</artifactId>
        <version>1.2</version>
    </dependency>
</dependencies>
```

3、配置web.xml，注册DispatcherServlet

DispatcherServlet包中自带，直接取用，不需要和以前一样自己新建

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">

    <!--1、注册DispatcherServlet-->
    <servlet>
        <servlet-name>springmvc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!--关联一个springMVC配置文件：（格式）【servlet-name】-servlet.xml-->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:springmvc-servlet.xml</param-value>
        </init-param>
        <!--启动级别1-->
        <load-on-startup>1</load-on-startup>
    </servlet>

    <!-- / 匹配所有请求（不包括.jsp）-->
    <!-- /* 匹配所有请求（包括.jsp）-->
    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```

4、在resources目录下新建SpringMVC配置文件

添加处理映射器、处理器适配器、视图解析器

springmvc-servlet.xml

```xml
<?xml version="1.0" encoding="UTF8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           https://www.springframework.org/schema/beans/spring-beans.xsd">  

    <!--添加 处理映射器-->
    <bean class="org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping"/>
    <!--添加 处理器适配器-->
    <bean class="org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter"/>
    <!--添加 视图解析器-->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver" id="internalResourceViewResolver">
        <!--前缀-->
        <property name="prefix" value="/WEB-INF/jsp/"/>
        <!--后缀-->
        <property name="suffix" value=".jsp"/>
    </bean>

    <!--handler-->
    <bean id="/hello" class="com.ghy.controller.HelloController"/>
</beans>
```

5、在java目录下的controller目录下编写操作业务的Controller，可以实现Controller接口或用注解

Controller返回封装了数据和视图的ModelAndView对象

```java
public class HelloController implements Controller {
    @Override
    public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
        // ModelAndView模型和视图
        ModelAndView mv = new ModelAndView();
        
        //  

        // 封装对象于ModelAndView中（不再需要将数据存到session）
        mv.addObject("msg","helloSpringMVC");

        // 封装要跳转的视图于ModelAndView中（不再需要用redirect重定向）
        mv.setViewName("hello");  // 根据mvc配置文件的视图解析器，完全的相对地址为：/WEB-INF/jsp/hello.jsp
        return mv;
    }
}
```

6、将编写的Controller类交给Spring IOC容器，注册bean

​	  代码见第4步

7、编写涉及的jsp页面（可以使用${}获取ModelAndView存放的数据）

web/WEB-INF/jsp/hello.jsp

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>ghySpringMVC</title>
</head>
<body>
    ${msg}
</body>
</html>
```

注：如果报404，可能是Tomcat缺少jar包，在Project Structure的Artifacts的对应包中新建lib，并导入所有jar包

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211208164756376.png" alt="image-20211208164756376" style="zoom: 50%;" />



**程序运行过程**

- 在浏览器地址栏输入地址，被servlet配置文件web.xml的servlet-mapping匹配给DispatcherServlet（自定义名springmvc）
- 查看DispatcherServlet设置的配置文件，springmvc-servlet.xml，用处理映射器和处理器适配器分别对handler进行匹配和执行
- 匹配的handler类的bean执行后，得到含有数据和视图的ModelAndView
- 用springmvc-servlet.xml中的视图解析器解析ModelAndView中的视图，在工程中寻找对应的页面文件并跳转

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211208170609551.png" alt="image-20211208170609551" style="zoom:67%;" />

以前是用多个servlet进行地址匹配，现在是只有一个总的Servlet，根据地址分配给不同的Controller

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211208171150884.png" alt="image-20211208171150884" style="zoom: 80%;" />



### 6.2 使用注解实现SpringMVC程序

1、新建module，添加web支持；设置好Tomcat

2、导入SpringMVC依赖（可以导到父工程）

3、配置web.xml，注册DispatcherServlet

DispatcherServlet包中自带，直接取用，不需要和以前一样自己新建

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">

    <!--1、注册DispatcherServlet-->
    <servlet>
        <servlet-name>springmvc</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!--关联一个springMVC配置文件：（格式）【servlet-name】-servlet.xml-->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:springmvc-servlet.xml</param-value>
        </init-param>
        <!--启动级别1-->
        <load-on-startup>1</load-on-startup>
    </servlet>

    <!-- / 匹配所有请求（不包括.jsp）-->
    <!-- /* 匹配所有请求（包括.jsp）-->
    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
    
</web-app>
```

4、在resources目录下新建SpringMVC配置文件

使用注解实现：

- beans标签属性添加context和mvc相关内容
- 设置扫描注解的包
- 处理映射器、处理器适配器用一个标签完成
- 视图解析器同6.1
- handler会在扫描注解时自动获取，不需要再设置

springmvc-servlet.xml

```xml
<?xml version="1.0" encoding="UTF8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/mvc
        https://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <!--扫描指定包下的注解-->
    <context:component-scan base-package="com.ghy.controller"/>

    <!--设置SpringMVC不处理静态资源-->
    <mvc:default-servlet-handler/>

    <!--添加 处理映射器、处理器适配器--><!--工程默认就有，如果不需要额外设置可以省略-->
    <mvc:annotation-driven/>

    <!--添加 视图解析器-->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver" id="internalResourceViewResolver">
        <!--前缀-->
        <property name="prefix" value="/WEB-INF/jsp/"/>
        <!--后缀-->
        <property name="suffix" value=".jsp"/>
    </bean>

</beans>
```

5、在java目录下的controller目录下编写操作业务的Controller

- 用@Controller设置为Controller
- RequestMapping注解用于映射url到控制器类或一个特定的处理程序方法
  - 类的@RequestMapping("/hello")是本类各视图的公共地址前缀
  - 方法的@RequestMapping("/h1")是对应视图的地址
- 对应视图的方法的参数model用于封装数据
- 对应视图的方法的返回值为字符串，该字符串会被视图解析器解析，从而到工程中寻找对应的页面文件（如jsp）

```java
@Controller
@RequestMapping("/hello")
public class helloController {

    // localhost:8080/hello/h1
    @RequestMapping("/h1")
    public String hello(Model model){
        // 封装数据
        model.addAttribute("msg","使用注解实现SpringMVC");
        // 返回的字符串会被视图解析器处理
        return "hello";
    }
}
```

​	动态页面：多个不同地址的方法返回同一个前端页面，通过model中数据的不同使网页显示不同的内容

（视图是被复用的，控制器和视图之间低耦合的）

6、编写页面文件

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>ghySpringMVC-注解</title>
</head>
<body>
${msg}
</body>
</html>
```



**对于开发者，只需要编写操作业务的Controller、视图解析器和前端页面文件，其余内容都成了框架，内容较为固定。**



### 6.3 传参与回显



#### 6.3.1 传统方式

根据不同的参数实现不同的效果。

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211209140951767.png" alt="image-20211209140951767" style="zoom:67%;" />

```java
// 传统风格：http://localhost:8080/add?a=1&b=2
@RequestMapping("/add")
public String hello(int a, int b, Model model){
    int res = a + b;
    model.addAttribute("msg","结果：" + res);
    return "hello";
}
```

​	url中用?和&间隔各个参数，参数根据反射机制被读取到方法的同名参数中。

注1：url参数和对应方法的参数名不同时，也可以加上`@RequestParam`注解手动设置

```java
// http://localhost:8080/user1?username=123
@GetMapping("/user1")
public String test1(@RequestParam("username") String name, Model model){
    model.addAttribute("msg", name);
    return "hello";
}
```

注2：方法的参数是对象时

```java
// http://localhost:8080/user1?name=ghy&id=2
@GetMapping("/user1")
public String test1(User user, Model model){
    model.addAttribute("msg", user.toString());
    return "hello";
}
```

会去匹配类的属性名，如果匹配上了，用对应的构造函数生成对象并传给参数



#### 6.3.2 RestFul风格

一种资源定位及资源操作的风格，较为简洁，有层次，更易于实现缓存等机制

可以通过不同的请求方式实现：地址一样，但效果不同

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211209141019450.png" alt="image-20211209141019450" style="zoom:67%;" />

```java
// RestFul风格：localhost:8080/add/1/2
@RequestMapping("/add/{a}/{b}")
public String hello(@PathVariable int a, @PathVariable int b, Model model){
    int res = a + b;
    model.addAttribute("msg","结果：" + res);
    return "hello";
}
```

在@RequestMapping用｛name｝指定参数名，各参数用/隔开。

方法参数中，在对应的参数前加上@PathVariable注解。



**同一url不同请求方式实现不同效果：**

```java
 @RequestMapping(value = "/add/{a}/{b}", method = RequestMethod.GET)
```

​	也可以直接用对应的注解

```
@GetMapping
@PostMapping
@PutMapping
@DeleteMapping
@PatchMapping
```

​	具体实现:

```java
@GetMapping("/add/{a}/{b}")
public String hello(@PathVariable int a, @PathVariable int b, Model model){
    int res = a + b;
    model.addAttribute("msg","get结果：" + res);
    return "hello";
}

@PostMapping("/add/{a}/{b}")
public String hello(@PathVariable int a, @PathVariable int b, Model model){
    int res = a + b;
    model.addAttribute("msg","post结果：" + res);
    return "hello";
}
```



#### 6.3.3 回显

一般用Model，比较简单

还可以用ModelMap，其继承了LinkedHashMap，可以当作LinkedHashMap用

还可以用ModelAndView，即可以存储数据，又可以设置跳转的视图（已被视图解析器代替）



在方法上标注@ResponseBody ，不走视图解析器跳转，直接把字符串返回到前端

```java
@ResponseBody 
public String json1() throws JsonProcessingException {
    User user = new User("西琳", 14, "女");
    ObjectMapper mapper = new ObjectMapper();
    String s = mapper.writeValueAsString(user);
    return s;
}
```

或在类上标注@RestController，该类的所有方法都不走视图解析器跳转，直接把字符串返回到前端

```java
//@RestController
public class UserController {...}
```



#### 6.3.4 回显乱码问题

Get方法本身就不会有乱码，Post会有

1、调整Tomcat的编码

在其config目录下的server.xml中加上

```XML
<Connector port="8080" protocol="HTTP/1.1"
		   connectionTimeout="20000"
		   redirectPort="8443"
		   URIEncoding="UTF-8"/>
```

2、添加过滤器

按照3.8的做法设置过滤器，可以自己写过滤器类。也可以用Spring自带的：

在web.xml配置文件中编写

```xml
<filter>
    <filter-name>encoding</filter-name>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <init-param>
        <param-name>encoding</param-name>
        <param-value>utf-8</param-value>
    </init-param>
</filter>
<filter-mapping>
    <filter-name>encoding</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

注：/匹配不含jsp的一切地址，/*匹配含jsp的一切地址



### 6.4 重定向和请求转发

Controller类的方法可使用HttpServletRequest、HttpServletResponse，像直接操作Servlet一样使用（不推荐）

```java
@RequestMapping(value = "/add/{a}/{b}", method = RequestMethod.GET)
public String hello(HttpServletRequest request, HttpServletResponse response){
    request.getSession();
    return "hello";
}
```

- 转发

  Controller方法return的内容默认进行转发，在页面名前面加上forward也可以

```java
// 等价
return "hello"
return "forward:/WEB-INF/jsp/hello.jsp";
```

​		加上forward后不通过视图解析器，要写全地址

- 重定向

  在页面名加redirect，不通过视图解析器，要写全地址，而且**无法访问WEB-INF目录**

```java
return "redirect:/index.jsp"
```

​		WEB-INF是服务器端才能访问的，而重定向相当于客户端再次请求服务器，自然无法访问

​	（WEB-INF下的所有资源只能通过Controller或Servlet访问）



### 6.5 Json

JSON (JavaScript Object Notation, JS 对象简谱) 是一种轻量级的数据交换格式。

**作用：作为数据格式的标准，进行数据格式的转换，让前端和服务器端都能拿到对于那一方可用的数据类型。**



#### 6.5.1 Jackson

1、导入jar包

```xml
<!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-databind -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.13.0</version>
</dependency>
```

2、SpringMVC配置（固定）

web.xml、springmvc-servlet.xml

3、实体类

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class User {
    private String name;
    private int age;
    private String sex;
}
```

4、Controller与Jackson

- 方法一：Jackson的工具类ObjectMapper

  ```java
  // 接收Json字符串，转为对象
  Student s = mapper.readValue(jsonString, Student.class);
  
  // 对象转为Json字符串，返回给前端
  return mapper.writeValueAsString(user);
  ```

- 方法二：注解

  在对应的Controller方法上加上@ResponseBody注解

```java
@RequestMapping(value = "/j1")
@ResponseBody  // 不走视图解析器跳转，直接把字符串返回到前端
public String json1() throws JsonProcessingException {

    User user = new User("西琳", 14, "女");

    // jackson，将user转为JSON字符串
    ObjectMapper mapper = new ObjectMapper();
    String s = mapper.writeValueAsString(user);
    
    return s;
}
```

​	直接返回User对象，@ResponseBody也能将之自动转换成Json字符串，但手动调用工具类可能性能更好

​	与@ResponseBody相对的是@RequestBody，一般用于接收前端发送的Json数据，并自动封装到pojo中



- 乱码问题

  - 方法1：在方法的@RequestMapping注解中设置编码

    ```
    @RequestMapping(value = "/j1", produces = "application/json;charset=utf-8")
    ```

  - ，方法2：在springmvc-servlet.xml进行总体设置

    用Spring自带的处理类
    
    ```xml
    <!-- JSON乱码问题配置 -->
    <mvc:annotation-driven>
        <mvc:message-converters register-defaults="true">
            <bean class="org.springframework.http.converter.StringHttpMessageConverter">
                <constructor-arg value="UTF-8"/>
            </bean>
            <bean class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter">
                <property name="objectMapper">
                    <bean class="org.springframework.http.converter.json.Jackson2ObjectMapperFactoryBean">
                        <property name="failOnEmptyBeans" value="false"/>
                    </bean>
                </property>
            </bean>
        </mvc:message-converters>
    </mvc:annotation-driven>
    ```

- 时间Date问题（Jackson默认把Date对象转为时间戳格式）

  - 方法1：转JSON前调整date对象格式

    ```java
    @RequestMapping(value = "/j2")
    @ResponseBody  // 不走视图解析器跳转，直接把字符串返回到前端
    public String json2() throws JsonProcessingException {
        Date date = new Date();
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        return new ObjectMapper().writeValueAsString(sdf.format(date));
    }
    ```

  - 方法2：设置JSON解析器

    ```java
    @RequestMapping(value = "/j2")
    @ResponseBody  // 不走视图解析器跳转，直接把字符串返回到前端
    public String json2() throws JsonProcessingException {
        Date date = new Date();
        ObjectMapper mapper = new ObjectMapper();
        // 关闭时间戳
        mapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
        // 手动创建、导入时间格式
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        mapper.setDateFormat(sdf);
        return new mapper.writeValueAsString(date);
    }
    ```

    

5、Jackson的封装

将Jackson的操作封装成工具类，直接调用即可

com.ghy.utils.JsonUtils.java

```java
public class JsonUtils {
    public static String GetJson(Object obj) throws JsonProcessingException {
        return GetJson(obj, "yyyy-MM-dd HH:mm:ss");  // 默认用这个时间格式
    }

    public static String GetJson(Object obj, String dataFormat) throws JsonProcessingException {
        ObjectMapper mapper = new ObjectMapper();
        mapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
        SimpleDateFormat sdf = new SimpleDateFormat(dataFormat);
        mapper.setDateFormat(sdf);
        return mapper.writeValueAsString(obj);
    }
}
```

使用时直接调用

```java
@RequestMapping(value = "/j3")
@ResponseBody  
public String json3() throws JsonProcessingException {
    Date date = new Date();
    return JsonUtils.GetJson(date);
}
```



#### 6.5.2 Fastjson

1、导入jar包

```xml
<!-- https://mvnrepository.com/artifact/com.alibaba/fastjson -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.78</version>
</dependency>
```

2、四种方法

```java
// Java对象 -> JSON字符串
String s = JSON.toJSONString(obj);

// JSON字符串 -> Java对象
User user = JSON.parseObject(s, User.class);

// Java对象 -> JSON对象（本质是JS对象）
JSONObject jsobj = (JSONObject)JSON.toJSON(obj);

// JSON对象（本质是JS对象） -> Java对象
User user = JSON.toJavaObject(jsobj, User.class);
```

3、用注解进行设置

```java
// 标注到目标类的每个属性上
@JSONField(参数)    
ordinal = 0                 // 标注属性的顺序
name = “”                   // 给属性改名
format = “”                 // 改格式，如Date对象
serialize = true/false      // 是否将该属性转为JSON字符串
deserialize = true/false    // 解析JSON字符串为Java类时，是否解析该属性
```



### 6.6 整合SSM框架（重点）



#### 6.6.1 工程准备

##### 1、新建数据库

Navicat + MySQL

```sql
CREATE DATABASE `ssmbuild`;

USE `ssmbuild`;

DROP TABLE IF EXISTS `books`;

CREATE TABLE `books`(
`bookID` INT(10) NOT NULL AUTO_INCREMENT COMMENT '书id',
`bookName` VARCHAR(100) NOT NULL COMMENT '书名',
`bookCounts` INT(11) NOT NULL COMMENT '数量',
`detail` VARCHAR(200) NOT NULL COMMENT '描述',
KEY `bookID`(`bookID`)
)ENGINE=INNODB DEFAULT CHARSET=utf8;

INSERT INTO `books`(`bookID`,`bookName`,`bookCounts`,`detail`) VALUES
(1,'Java',1,'从入门到放弃'),
(2,'MySQL',10,'从删库到跑路'),
(3,'Linux',5,'从进门到进牢');
```

##### 2、新建普通Maven工程

##### 3、导入依赖

- junit
- 数据库相关：数据库驱动mysql-connector-java，数据库连接池c3p0
- Servlet和Jsp相关：javax.servlet-api、jsp-api、jstl
- MyBatis相关：mybatis、mybatis-spring
- Spring相关：spring-webmvc、spring-jdbc、aop支持aspectjweaver
- 其他工具：如lombok

pom.xml

```xml
<dependencies>

    <!--Junit-->
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
    </dependency>

    <!--数据库驱动、数据库连接池c3p0-->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.25</version>
    </dependency>

    <dependency>
        <groupId>com.mchange</groupId>
        <artifactId>c3p0</artifactId>
        <version>0.9.5.5</version>
    </dependency>

    <!--Servlet - JSP-->
    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>javax.servlet-api</artifactId>
        <version>4.0.1</version>
    </dependency>

    <dependency>
        <groupId>javax.servlet.jsp</groupId>
        <artifactId>jsp-api</artifactId>
        <version>2.2</version>
    </dependency>

    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>jstl</artifactId>
        <version>1.2</version>
    </dependency>


    <!--MyBatis-->
    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis</artifactId>
        <version>3.5.7</version>
    </dependency>

    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis-spring</artifactId>
        <version>2.0.6</version>
    </dependency>

    <!--Spring-->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>5.3.13</version>
    </dependency>

    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-jdbc</artifactId>
        <version>5.3.13</version>
    </dependency>

    <dependency>
        <groupId>org.aspectj</groupId>
        <artifactId>aspectjweaver</artifactId>
        <version>1.9.7</version>
    </dependency>

    <!--工具-->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.22</version>
    </dependency>

</dependencies>
```

##### 4、解决静态资源导出的问题

pom.xml

```xml
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <includes>
                <include>**/*.properties</include>
                <include>**/*.xml</include>
            </includes>
            <filtering>true</filtering>
        </resource>
        <resource>
            <directory>src/main/java</directory>
            <includes>
                <include>**/*.properties</include>
                <include>**/*.xml</include>
            </includes>
            <filtering>true</filtering>
        </resource>
    </resources>
</build>
```

##### 5、连接数据库

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211210173107097.png" alt="image-20211210173107097" style="zoom: 50%;" />

##### 6、创建各层目录

**在java下创建目录：**

- 实体类的com.ghy.pojo（Model）
- 实现对数据库和实体类操作的com.ghy.dao（Model）
- 业务逻辑的com.ghy.service（Model）
- 连接前后端的com.ghy.controller（C）



##### 7、创建总配置文件

在resources目录下创建工程总配置文件 applicationContext.xml，**负责连接所有配置文件**

```xml
<?xml version="1.0" encoding="UTF8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <!--引入Spring数据库设置-->
    <import resource="classpath:spring-dao.xml"/>
    <import resource="classpath:spring-service.xml"/>
    <import resource="classpath:spring-mvc.xml"/>

</beans>
```



#### 6.6.2 DAO层

##### 1、mybatis配置文件

​	  在resources目录下创建mybatis配置文件mybatis-config.xml

​	  实际上所有设置都可以在Spring的dao层实现，把他单独创建出来只是为了单独设置一些不重要的内容，使Spring的dao层更清爽。

- mybatis配置文件：mybatis-config.xml

  ```xml
  <?xml version="1.0" encoding="UTF8" ?>
  <!DOCTYPE configuration
          PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
          "http://mybatis.org/dtd/mybatis-3-config.dtd">
  
  <configuration>
  
      <!--给实体类其别名-->
      <typeAliases>
          <package name="com.ghy.pojo"/>
      </typeAliases>
  
  </configuration>
  ```

##### 2、数据库连接文件

​	  连接数据库所需的内容

- 数据库连接文件：db.properties

  ```properties
  driver=com.mysql.cj.jdbc.Driver
  url=jdbc:mysql://localhost:3306/ssmbuild?useSSL=true&useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC
  username=root
  password=001118
  ```

##### 3、Spring的DAO层配置文件

引入之前写的mybatis配置文件，进行数据库的连接和配置，把DAO层的接口注入Spring容器

- Spring的DAO层配置文件：Spring-dao.xml

  ```xml
  <?xml version="1.0" encoding="UTF8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns:context="http://www.springframework.org/schema/context"
         xsi:schemaLocation="http://www.springframework.org/schema/beans
                             https://www.springframework.org/schema/beans/spring-beans.xsd
                             http://www.springframework.org/schema/context
                             https://www.springframework.org/schema/context/spring-context.xsd">
  
      <!--1、导入数据库配置文件-->
      <context:property-placeholder location="classpath:db.properties"/>
  
      <!--2、数据源（连接池）-->
      <bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">
      <!--数据源（Spring自带的普通数据库连接）-->
      <!--<bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">-->
          <!--数据源基本属性-->
          <property name="driverClass" value="${driver}"/>
          <property name="jdbcUrl" value="${url}"/>
          <property name="user" value="${username}"/>
          <property name="password" value="${password}"/>
  
          <!--c3p0私有属性-->
          <property name="maxPoolSize" value="30"/>
          <property name="minPoolSize" value="10"/>
          <property name="autoCommitOnClose" value="false"/> <!--不自动commit-->
          <property name="checkoutTimeout" value="10000"/>   <!--获取连接超时时间-->
          <property name="acquireRetryAttempts" value="2"/>  <!--获取连接失败重试次数-->
      </bean>
  
      <!--3、sqlSessionFactory-->
      <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
          <property name="dataSource" ref="dataSource" />
          <!--绑定MyBatis配置文件-->
          <property name="configLocation" value="classpath:mybatis-config.xml"/>
          <property name="mapperLocations" value="classpath:com/ghy/dao/*.xml"/>
      </bean>
  
      <!--4、配置dao接口扫描包，动态地实现dao接口注入Spring容器--><!--不需要再手动编写dao层的实现类-->
      <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
          <!--注入sqlSessionFactory-->
          <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
          <!--扫描dao的包-->
          <property name="basePackage" value="com.ghy.dao"/>
      </bean>
  
  </beans>
  ```

  在sqlSessionFactory的设置中引入mybatis配置文件、绑定dao层的mapper

  这里第4步动态地实现dao接口注入Spring容器，因此不需要手动编写dao接口的具体实现类了，否则见 *5.10.1-3*

##### 4、实体类

实体类及其属性和数据库的表对应

com.ghy.pojo.Books.java

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Books {
    private int bookID;
    private String bookName;
    private int bookCounts;
    private String detail;
}
```

##### 5、dao接口

定义对对应类的数据库操作

com.ghy.dao.BookMapper.java

```java
public interface BookMapper {

    // 增加一本书
    int addBook(Books books);

    // 删除一本书
    int deleteBook(@Param("bookID") int id);

    // 更新一本书
    int updateBook(Books books);

    // 查询一本书
    Books queryBook(@Param("bookID") int id);

    // 查询所有书
    List<Books> queryAllBook();
}
```

##### 6、dao接口实现Mapper.xml

定义dao接口各操作的sql语句，**也可以在dao接口类中用注解实现**

com.ghy.dao.BookMapper.xml

```xml
<?xml version="1.0" encoding="UTF8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.ghy.dao.BookMapper">

    <insert id="addBook" parameterType="Books">
        insert into ssmbuild.books (bookName, bookCounts, detail)
        values(#{bookName}, #{bookCounts}, #{detail})
    </insert>

    <delete id="deleteBook" parameterType="int">
        delete from ssmbuild.books where bookID=#{bookID}
    </delete>

    <update id="updateBook" parameterType="Books">
        update ssmbuild.books
        set bookName=#{bookName},bookCounts=#{bookCounts},detail=#{detail}
        where bookID=#{bookID}
    </update>

    <select id="queryBook" resultType="Books">
        select * from ssmbuild.books where bookID=#{bookID}
    </select>

    <select id="queryAllBook" resultType="Books">
        select * from ssmbuild.books
    </select>

</mapper>
```

注：在Mapper.xml实现sql自动补全的方法

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211210180257942.png" alt="image-20211210180257942" style="zoom: 50%;" />

##### 7、在配置文件中绑定dao接口实现类Mapper

可以在MyBatis配置文件中绑定

```xml
<mappers>
    <mapper class="com.ghy.dao.BookMapper"/>
</mappers>
```

也可以在spring和dao的联合配置文件中绑定，见 *6.6.2-3*



#### 6.6.3 Service层

service层是Model层和Controller层的连接处，其接收Controller层发送来的、来自客户端的请求，调用更底层的dao层具体实现数据库操作，从dao层获得操作的结果，并把结果返回给Controller层

##### 1、Spring的Service层配置文件

- Spring的Service层配置文件：Spring-service.xml

  ```xml
  <?xml version="1.0" encoding="UTF8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns:context="http://www.springframework.org/schema/context"
         xmlns:tx="http://www.springframework.org/schema/tx"
         xmlns:aop="http://www.springframework.org/schema/aop"
         xsi:schemaLocation="http://www.springframework.org/schema/beans
          https://www.springframework.org/schema/beans/spring-beans.xsd
          http://www.springframework.org/schema/context
          https://www.springframework.org/schema/context/spring-context.xsd
          http://www.springframework.org/schema/aop
          https://www.springframework.org/schema/aop/spring-aop.xsd
          http://www.springframework.org/schema/tx
          https://www.springframework.org/schema/tx/spring-tx.xsd">
  
      <!--1、扫描service下的包，使其注解生效-->
      <context:component-scan base-package="com.ghy.service" />
  
      <!--2、将业务类注入到Spring容器，可以用property配置，也可以用@Autowired注解-->
      <bean id="BookServiceImpl" class="com.ghy.service.BookServiceImpl">
          <property name="bookMapper" ref="bookMapper"/>
      </bean>
  
      <!--3、配置声明式事务-->
      <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
          <!--<constructor-arg ref="dataSource" />--><!--构造器注入也可以实现-->
          <property name="dataSource" ref="dataSource"/>
      </bean>
  
  
      <!--4、AOP事务支持-->
      <!--配置事务通知-->
      <tx:advice id="txAdvice" transaction-manager="transactionManager">
          <!--给哪些方法配置事务-->
          <tx:attributes>
  
              <!--配置事务的传播特性-->
              <tx:method name="add" propagation="REQUIRED"/>
              <tx:method name="delete" propagation="REQUIRED"/>
              <tx:method name="update" propagation="REQUIRED"/>
              <tx:method name="query" read-only="true"/>
              <tx:method name="*" propagation="REQUIRED"/>
          </tx:attributes>
      </tx:advice>
  
      <!--配置事务切入-->
      <aop:config>
          <aop:pointcut id="txPointCut" expression="execution(* com.ghy.dao.*.*(..))"/>
          <!--执行环绕-->
          <aop:advisor advice-ref="txAdvice" pointcut-ref="txPointCut"/>
      </aop:config>
  
  </beans>
  ```

  1、2是将Service接口实现类注入Spring容器，见 *6.6.3-3*

  3、4时用Spring自带的类注册事务，保证数据库的ACID特性

##### 2、Service接口

​	  业务类：Service层调用dao层的具体操作，所以service层中存在业务类，每个业务类对应一个dao层的接口，进行对应的调用

- com.ghy.service.BookService

  ```java
  public interface BookService {  
      // 增加一本书
      int addBook(Books books);
  
      // 删除一本书
      int deleteBook(int id);
  
      // 更新一本书
      int updateBook(Books books);
  
      // 查询一本书
      Books queryBook(int id);
  
      // 查询所有书
      List<Books> queryAllBook();
  }
  ```

  与dao层的BookMapper接口对应

##### 3、Service接口实现类

​	 实现Service层的接口，思路很简单，因为是调用dao层接口，所以类中有一个dao接口变量，各个操作都调用这个变量的方法

- com.ghy.service.BookServiceImpl

  ```java
  @Service
  public class BookServiceImpl implements BookService{
  
      // service层调用dao层：组合dao层
      @Autowired
      private BookMapper bookMapper;
  
      public void setBookMapper(BookMapper bookMapper) {
          this.bookMapper = bookMapper;
      }
  
      @Override
      public int addBook(Books books) {
          return bookMapper.addBook(books);
      }
  
      @Override
      public int deleteBook(int id) {
          return bookMapper.deleteBook(id);
      }
  
      @Override
      public int updateBook(Books books) {
          return bookMapper.updateBook(books);
      }
  
      @Override
      public Books queryBook(int id) {
          return bookMapper.queryBook(id);
      }
  
      @Override
      public List<Books> queryAllBook() {
          return bookMapper.queryAllBook();
      }
  }
  ```

  因为实现类也是普通的类，所以要注入Spring容器中，所以在Spring的Service层配置文件：Spring-service.xml中注册Bean，对于其中dao接口变量的赋值：

  - 方法一：在bean的property中直接设置
  - 方法二：在本实现类的dao接口变量属性上面加上@Autowired注解



#### 6.6.4 Controller层

Controller层是MVC架构的中间层，直接接收客户端的请求，将请求转发给Model层（Service、dao），得到Model层返回的数据后进行数据的返回和视图的跳转。

##### 1、Spring的Controller层配置文件

- Spring的Controller层配置文件：Spring-mvc.xml

  ```xml
  <?xml version="1.0" encoding="UTF8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns:context="http://www.springframework.org/schema/context"
         xmlns:mvc="http://www.springframework.org/schema/mvc"
         xsi:schemaLocation="http://www.springframework.org/schema/beans
          https://www.springframework.org/schema/beans/spring-beans.xsd
          http://www.springframework.org/schema/context
          https://www.springframework.org/schema/context/spring-context.xsd
          http://www.springframework.org/schema/mvc
          https://www.springframework.org/schema/mvc/spring-mvc.xsd">
  
      <!--1、扫描指定包下的注解-->
      <context:component-scan base-package="com.ghy.controller"/>
  
      <!--2、设置SpringMVC不处理静态资源-->
      <mvc:default-servlet-handler/>
  
      <!--3、添加 处理映射器、处理器适配器-->
      <mvc:annotation-driven/>
  
      <!--4、视图解析器-->
      <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver" id="internalResourceViewResolver">
          <!--前缀-->
          <property name="prefix" value="/WEB-INF/jsp/"/>
          <!--后缀-->
          <property name="suffix" value=".jsp"/>
      </bean>
  
  </beans>
  ```

  1、是扫描controller包，Controller类的注解生效

  2、是屏蔽html、jsp等静态资源

  3、是处理映射器、处理器适配器，接收客户端的url请求，根据其ur分别l匹配、执行对应的controller对象

  4、是视图解析器，实现controller对象执行结束后的视图跳转



##### 2、Servlet配置文件

​	 主要任务是注册Servlet，匹配客户端url请求

- Servlet配置文件：/web/WEB-INF/web.xml

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
           xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
           version="4.0">
  
      <!--1、注册DispatcherServlet-->
      <servlet>
          <servlet-name>springmvc</servlet-name>
          <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
          <!--关联一个springMVC配置文件：（格式）【servlet-name】-servlet.xml-->
          <init-param>
              <param-name>contextConfigLocation</param-name>
              <param-value>classpath:applicationContext.xml</param-value>
          </init-param>
          <!--启动级别1-->
          <load-on-startup>1</load-on-startup>
      </servlet>
  
      <!-- / 匹配所有请求（不包括.jsp）-->
      <!-- /* 匹配所有请求（包括.jsp）-->
      <servlet-mapping>
          <servlet-name>springmvc</servlet-name>
          <url-pattern>/</url-pattern>
      </servlet-mapping>
  
      <!--2、过滤器（乱码）-->
      <filter>
          <filter-name>encoding</filter-name>
          <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
          <init-param>
              <param-name>encoding</param-name>
              <param-value>utf-8</param-value>
          </init-param>
      </filter>
      <filter-mapping>
          <filter-name>encoding</filter-name>
          <url-pattern>/*</url-pattern>
      </filter-mapping>
  
      <!--3、Session设置-->
      <session-config>
          <!--Session失效时间，单位为分钟-->
          <session-timeout>15</session-timeout>
      </session-config>
  
  </web-app>
  ```

  SpringMVC中，只设少数的Servlet（上述代码只有1个），每个Servlet对应多个controller类，每个controller类的一个方法对应一个url地址，由一个个controller类及其方法实现最开始一个个Servlet类的工作。

  过滤器只是用Spring自带的过滤器解决乱码。其他设置非必需。



##### 3、功能业务

所有的Controller功能业务都用java目录下的contoller包的Controller类实现，先新建一个Controller类

- com.ghy.controller.BookController.java

  ```java
  @Controller
  @RequestMapping("/book")
  public class BookController {
      // 调用Service层
      @Autowired                     // 先byType自动注入
      @Qualifier("BookServiceImpl")  // byType失败时byName自动注入
      private BookService bookService;
  }
  ```

  Controller类中调用Service层接口实现

具体功能的实现见 *6.6.5*



#### 6.6.5 功能业务实现

一个功能从下往上实现

##### 1、DAO层

  先在接口中定义功能方法

- BookMapper.java

  ```java
  // 查询所有书
  List<Books> queryAllBook();
  ```

  再在该接口的实现mapper中进行具体的实现。

- BookMapper.xml

  ```java
  <select id="queryAllBook" resultType="Books">
      select * from ssmbuild.books
  </select>
  ```

##### 2、Service层

  Service层接口照搬DAO层接口的方法定义。

- BookService.java

  ```java
  // 查询所有书
  List<Books> queryAllBook();
  ```

  如果参数有sql操作用的注解可以去掉：`Books queryBook(@Param("bookID") int id);`

  再在Service层接口的实现类中调用dao层，实现该功能

- BookServiceImpl.java

  ```java
  @Override
  public List<Books> queryAllBook() {
      return bookMapper.queryAllBook();
  }
  ```

##### 3、Controller层

  在该实体类对应的Controller类中新建处理该功能请求的方法，

- **BookController.java**

  ```java
  @RequestMapping("/allbook")
  public String list(Model model){
      List<Books> list = bookService.queryAllBook();
      model.addAttribute("books", list);
      return "allBook";
  }
  ```

  @RequestMapping和前端url联系，在方法中调用Service层对象实现操作，获取数据，并进行转发或重定向

- **向前端传送数据**

  要传到前端的数据放到Model对象中并命名，前端中用${数据名}取用

  ```java
  model.addAttribute("books", list);
  ```

- **接收前端的数据**

  一般是`<form>`标签的submit的数据或超链接中地址附加的数据，接收方法有两种格式

  - 传统风格（`<form>`标签或超链接数据）

    @RequestMapping不变，方法参数中接收对应的参数，参数名要和前端中对应的`<input name="”>`相同

    ```java
    // 跳转到修改书籍页面
    @RequestMapping("/toUpdateBook")
    public String toUpdateBook(int id, Model model){
        Books book = bookService.queryBook(id);
        model.addAttribute("book",book);
        return "updateBook";
    }
    ```

    跳转到此页面时，参数会以这种形式写在地址后面，并被自动接收：`http://localhost:8080/book/toUpdateBook?id=2`

  - RestFul风格（超链接数据）

    修改@RequestMapping中的地址格式，手动的将参数定位到地址的某处，跳转时自动去相应位置取。

    方法中的形参要加@PathVariable注解，还可以在其中改名，不改时不用加`()`。

    地址中对应位置的参数名和@PathVariable对应，进行参数的读入。

    ```java
    // 删除书籍请求(RestFul风格)
    @RequestMapping("/deleteBook/{bookid}")
    public String deleteBook(@PathVariable("bookid") int id){
        bookService.deleteBook(id);
        return "redirect:/book/allbook";  // 重定向
    }
    ```

    跳转到此页面后的地址：`http://localhost:8080/book/toUpdateBook/2`，最后的`2`会根据设置传给id

- **转发和重定向**

  方法执行完后可以转发或重定向。

  ```java
  // 转发
  return "allBook";
  // 重定向
  return "redirect:/book/allbook";
  ```

  转发return地址不变，服务器根据视图解析器去找工程中的页面文件进行显示；
  重定向return "redirect:/"不走视图解析器，相当于自动的再访问一次，地址会变，服务器去找对应的Controller方法进行执行。



##### 4、View层

部分方法要有自己的前端界面，可以在web目录下进行实现。

因为WEB-INF目录重定向无法访问到，所以推荐把页面存到WEB-INF目录下保证安全。B

前端页面用前端的方式实现，如jsp：

allBook.jsp

```jsp
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>全部书籍</title>

    <%--使用BootStrap--%>
    <link href="https://cdn.bootcdn.net/ajax/libs/twitter-bootstrap/5.0.2/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
    <div class="container">
        <div class="row">
            <div class="col-md-8 column">
                <div class="page-header">
                    <h1>全部书籍展示</h1>
                </div>
            </div>
        </div>

        <div class="row">
            <div class="col-md-4 column">
                <a href="${pageContext.request.contextPath}/book/toAddBook">新增书籍</a>
            </div>
            <div class="col-md-4 column">
                <form action="${pageContext.request.contextPath}/book/queryBookByName" method="post">
                    <%--placeholder为提示信息--%>
                    <input type="text" name="queryBookName" class="form-control" placeholder="请输入要查询的书名">
                    <input type="submit" value="查询">
                </form>
            </div>
        </div>
    </div>

    <div class="row">
        <div class="col-md-8 column">
            <table class="table table-hover table-striped">
                <thead>
                    <th>书籍编号</th>
                    <th>书籍名称</th>
                    <th>书籍数量</th>
                    <th>书籍详情</th>
                    <th>操作</th>
                </thead>

                <tbody>
                    <c:forEach var="book" items="${books}">
                        <tr>
                            <td>${book.bookID}</td>
                            <td>${book.bookName}</td>
                            <td>${book.bookCounts}</td>
                            <td>${book.detail}</td>
                            <td>
                                <a href="${pageContext.request.contextPath}/book/toUpdateBook?id=${book.bookID}">修改</a>
                                &nbsp; | &nbsp;
                                <a href="${pageContext.request.contextPath}/book/deleteBook/${book.bookID}">删除</a>
                            </td>
                        </tr>
                    </c:forEach>
                </tbody>
            </table>
        </div>
    </div>
</body>
</html>
```



- 部分相关技术：
  - html、css、javascript
  - json、ajax
  - bootstrap、vue



- 接收后端数据

  前端用${数据名}取用后端发送的数据，数据为对象时可以直接用其内部属性${数据名.属性名}

  ```jsp
  <c:forEach var="book" items="${books}">
      <tr>
          <td>${book.bookID}</td>
          <td>${book.bookName}</td>
          <td>${book.bookCounts}</td>
          <td>${book.detail}</td>
      </tr>
  </c:forEach>
  ```

- 向后端传送数据

  **一般使用`<form>`标签**

  ```jsp
  <!--bootstrap官网 表单-->
  <form action="${pageContext.request.contextPath}/book/addBook" method="post">
      <div class="form-group">
          <label>书籍名称</label>
          <input type="text" name="bookName" class="form-control" required>
      </div>
      <div class="form-group">
          <label>书籍数量</label>
          <input type="text" name="bookCounts" class="form-control" required>
      </div>
      <div class="form-group">
          <label>书籍描述</label>
          <input type="text" name="detail" class="form-control" required>
      </div>
      <div class="form-group">
          <input type="submit" class="form-control" value="添加">
      </div>
  </form>
  ```

  action为要提交的url，数据会传到对应Controller类的对应方法

  method一般为post或get

  传统方法传送数据时，每个input标签的name属性要和后端对应

  input标签中加上required后，提交时该内容不能为空

  

  需要在表单中submit但不想显示在页面上的，可以用form表单的隐藏域：

  ```jsp
  <input type="hidden" name="bookID" value="${book.bookID}" required>
  ```

  

  **不需要用户输入，由超链接传送数据**

  一般是用超链接跳转，把数据放在地址后面

  ```jsp
  <!--传统风格-->
  <a href="${pageContext.request.contextPath}/book/toUpdateBook?id=${book.bookID}">修改</a>
  <!--RestFul风格-->
  <a href="${pageContext.request.contextPath}/book/deleteBook/${book.bookID}">删除</a>
  ```

  

#### 6.6.6 工程运行

##### 1、lib导包

要保证工程的Artifact的WEB-INF目录下有lib目录，并把所有的依赖代入进去

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211208164756376.png" alt="image-20211208164756376" style="zoom: 50%;" />

##### 2、配置Tomcat服务器

配置Tomcat

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211211202844330.png" alt="image-20211211202844330" style="zoom: 33%;" />

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211211103112274.png" alt="image-20211211103112274" style="zoom: 50%;" />



<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211211103135477.png" alt="image-20211211103135477" style="zoom:50%;" />



##### 3、运行工程

Tomcat部署完成后默认会打开index.jsp作为地址后缀为空时的页面。

也可以在web.xml中进行设置，选择作为首页的文件

```xml
<welcome-file-list>
    <welcome-file>需要打开的jsp</welcome-file>
</welcome-file-list>
```



#### 6.6.7 排错经验

```
1 数据库连接错误
1.1 c3p0数据库连接池无法连接数据库
	（A ResourcePool could not acquire a resource from its primary factory or source.）
    将db.properties的username改为其他关键字（如user），错误消失。
    原因可能是电脑里可能有其他隐藏的变量名字和这个重名，导致错误。
1.2 导入数据库配置文件
    <context:property-placeholder location="classpath:db.properties"/>，文件名前加classpath

2 bean不存在
2.1 检查配置文件中对应bean的注入
2.2 写Junit单元测试，测试bean的直接调用，能否正常运行
2.3 SpringMVC没有调用service层的bean
    applicationContext.xml要引入注册service层的bean的service层配置文件
    web.xml要绑定全局配置文件applicationContext.xml

3 重定向404
    保证地址完全正确，注意大小写
    重定向无法直接访问WEB-INF下的内容，要先重定向到对应的controller方法，由该方法转发
    重定向不走视图解析器，不用写comcat设置的基地址和项目地址，直接从真正影响controller的地址写起

    转发return地址不变，服务器去找工程中的页面文件进行显示；
    重定向return "redirect:/"地址会变，相当于自动的再访问一次，服务器去找对应的Controller方法

4 修改书籍失败
    要保证sql正常运行，id等需要但不该显示的，用传到前端页面的隐藏域中（可以用日志辅助查看sql的实际运行）
```



### 6.7 补充内容



#### 6.7.1 Ajax（分离前后端）

Ajax即Asynchronous Javascript And XML（异步JavaScript和XML）

是一种数据交互的方式，无需重新加载整个网页而能够更新部分网页，可以提高web应用的交互性 

**本质：后端不转发、重定向，只返回数据，把前端页面控制的主动权交给前端。实现前后端分离**

**前后端用Json格式相互传递数据**



1、后端

前端只负责返回数据，不再转发/重定向（二者必定刷新整个页面）

```java
@RestController
public class AjaxController {

    @RequestMapping("/a2")
    //@ResponseBody
    public List<Books> a2(){
        List<Books> books = new ArrayList<Books>();
        books.add(new Books(5,"胡桃", 3, "usa"));
        books.add(new Books(6,"希儿", 11, "seele"));
        books.add(new Books(7,"5ds", 55, "流天类星龙"));
        return books;
    }
}
```

@ResponseBody代表只返回数据，@RestController是让此类的所有方法只返回数据；

返回方式，return或response.getWriter().print();都可以



如果新书的内容是通过前端给的，可以用@RequestBody将之自动注入到Books对象中:

```java
@RestController
public class AjaxController {

    @RequestMapping("/a3")
    //@ResponseBody
    public List<Books> a2(@RequestBody Books newBook){
        List<Books> books = new ArrayList<Books>();
        books.add(newBook);
        return books;
    }
}
```

@RequestBody注解可以自动接收前端请求中的Json对象，并将之自动填充给注解的对象，对象的属性和Json的键对应

```javascript
// 前端传送的Json格式的数据
{"bookID":5, "bookName":"胡桃", "bookCounts":3, "detail":"usa"j
```

将一堆单个的属性自动整合成了一个对象



2、前端

用jQuery封装的Ajax获取数据，异步地更新部分网页

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
  <head>
    <title>首页</title>
    <script src="${pageContext.request.contextPath}/statics/js/jquery-3.6.0.js"></script>

    <script>
      function b(){
        $.post({
          url:"${pageContext.request.contextPath}/a2",
          data:{"name":$("#username").val()}, <!--向后端传送数据，没有时省略或填undefined-->
          success: function (data) {   <!--返回后执行函数，data为接收后端返回的数据-->
            var html="";
            for(let i = 0; i < data.length; i++){
              html += "<tr>" +
                      "<td>" + data[i].bookID + "</td>" +
                      "<td>" + data[i].bookName + "</td>" +
                      "<td>" + data[i].bookCounts + "</td>" +
                      "<td>" + data[i].detail + "</td>" +
                      "</tr>"
            }
            $("#bookInfo").html(html);  <!--更新指定html内容的源码-->
          },
          error:function (){

          }
        })
      }

    </script>
  </head>
  <body>

    <input type="button" value="获取书籍" onclick="b()">
    <table>
      <tr>
        <td>编号</td>
        <td>书名</td>
        <td>数量</td>
        <td>详情</td>
      </tr>
      <tbody id="bookInfo">
      </tbody>
    </table>
        
  </body>
</html>
```

把操作放到js函数中，并监听指定html内容的事件（onclick、onblur、keyup常用）,时间发生时执行，获取数据，动态更新指定html内容的源码

具体语法见前端笔记



3、注意点

前后端间以Json格式传送数据，前端向后端传要手写格式，后端向前端发送的数据会由Spring自动转换。

但是Spring是基于Jackson的，必须要导入这三个jar包，**并把他们加到项目的lib中**

```xml
<!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-databind -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.13.0</version>
</dependency>
<!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-core -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-core</artifactId>
    <version>2.13.0</version>
</dependency>
<!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-annotations -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-annotations</artifactId>
    <version>2.13.0</version>
</dependency>
```



4、具体应用举例

注册时注册信息的动态变化

如输入了邮箱后，能直接显示输入的是不是正确的邮箱地址；输入密码后能动态显示密码强度

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211212155937584.png" alt="image-20211212155937584" style="zoom:67%;" />



实现：

- 每个input后有一个字段（如`<span>`）用于显示这些额外信息
- 给对应的input加上js函数，监听其onblur事件，一旦输入完去输入别的东西，onblur事件就会发生，函数执行一次Ajax操作。
- Ajax读取input的内容，通过data属性传给后端controller对象对应方法。
- 该方法对内容进行处理（如正则判断邮箱格式，计算密码强度，查找数据库判断用户名是否已存在等），并将处理后的数据返回。
- Ajax用success属性对接收的数据进行处理，修改该input对应的字段的内容，实现反馈的显示。



#### 6.7.2 拦截器（权限检查）

类似与Servlet的过滤器Filter，SpringMVC自带

拦截器只会拦截访问控制器的方法



1、编写拦截器，实现HandlerInterceptor接口

com.ghy.config.MyInterceptor.java

```java
public class MyInterceptor implements HandlerInterceptor {

    // return true:放行，执行下个拦截器
    // return false:拦截方法，不让其继续执行，之后的拦截器无需运行了
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("=====处理前=====");
        return false;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        System.out.println("=====处理后=====");
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        System.out.println("=====通过拦截器=====");
    }
}
```

preHandle()方法用于拦截，postHandle()和afterCompletion()一般用于补充日志



2、在SpringMVC配置文件中将拦截器注入Spring容器

```xml
<!--5、拦截器-->
<mvc:interceptors>
    <mvc:interceptor>
        <mvc:mapping path="/**"/>
        <bean class="com.ghy.config.MyInterceptor"/>
    </mvc:interceptor>
</mvc:interceptors>
```

/*对应所有后缀只有一段的请求，如/a，无法对应 /a/1,

/**则完全地对应所有的请求

如此设置后，所有的Controller中的所有方法都会经过拦截器，运行顺序如下：

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211212165118958.png" alt="image-20211212165118958" style="zoom: 80%;" />



3、具体应用举例

应用：只有登录后才能进入用户界面，直接访问会被拦住

实现：

- 通过登录页面登录成功，跳转到登录成功页面的controller方法中，将用户信息（如用户名+密码）存到Session中
- 设定好拦截器的拦截对象页面（在配置文件中设置或拦截器类中手动判断当前页面:`getRequestURI`）
- 拦截器通过HttpServletResponse（方法参数就有）获取Session，检查Session的内容
  - 如果没问题，return true通过拦截器
  - 如果需要拦截，用HttpServletResponse重定向到报错页面等指定页面，return false
- 根据需求，给Session设置有效时间。
- 增加退出登录链接，其controller方法中删除Session中的用户信息

注意：严谨地设置拦截器的目标，不要误伤了不该拦截的页面



#### 6.7.3 文件上传和下载

1、导入jar包

```xml
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>4.0.1</version>
</dependency>

<dependency>
    <groupId>commons-fileupload</groupId>
    <artifactId>commons-fileupload</artifactId>
    <version>1.4</version>
</dependency>
```



2、在SpringMVC配置文件中将SpringMVC自带的文件传输类注入Spring容器

```xml
<!--6、文件传输配置-->
<bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
    <property name="defaultEncoding" value="utf-8"/>  <!--必须和对应jsp一致，默认ISO-8859-1-->
    <property name="maxUploadSize" value="10485760"/> <!--10M-->
    <property name="maxInMemorySize" value="40960"/>  <!--单位为字节-->
</bean>
```



3、前端页面

```jsp
<form action="${pageContext.request.contextPath}/upload2" enctype="multipart/form-data" method="post">
    <input type="file" name="file">
    <input type="submit" value="upload">
</form>
```

用表单上传文件。

下载文件可以直接在地址栏输入，也可以用超链接实现



4、文件上传的Controller类（两种）

```java
@RestController
public class FileController {

    @RequestMapping("/upload")
    public String fileUpload(@RequestParam("file") CommonsMultipartFile file, HttpServletRequest request) throws IOException {
        
        // 文件名
        String uploadFileName = file.getOriginalFilename();

        // 文件名为空，返回首页
        if("".equals(uploadFileName)){
            return "redirect:/index.jsp";
        }
        System.out.println("上传文件：" + uploadFileName);

        // 上传路径
        String path = request.getServletContext().getRealPath("/upload");
        // 路径不存在则新建
        File realPath = new File(path); // 上传时的url和实际保存到工程中的地址相同
        if(!realPath.exists()){
            realPath.mkdir();
        }
        System.out.println("上传文件保存地址：" + realPath);

        // 读取写出
        InputStream is = file.getInputStream(); //文件输入流
        OutputStream os = new FileOutputStream(new File(realPath, uploadFileName));

        int len = 0;
        byte[] buffer = new byte[1024];
        while((len=is.read(buffer)) != -1){
            os.write(buffer,0,len);
            os.flush();
        }

        os.close();
        is.close();
        return "redirect:/index.jsp";
    }

    @RequestMapping("/upload2")
    public String  fileUpload2(@RequestParam("file") CommonsMultipartFile file, HttpServletRequest request) throws IOException {

        //上传路径保存设置
        String path = request.getServletContext().getRealPath("/upload");
        File realPath = new File(path);
        if (!realPath.exists()){
            realPath.mkdir();
        }
        //上传文件地址
        System.out.println("上传文件保存地址："+realPath);

        //通过CommonsMultipartFile的方法直接写文件（注意这个时候）
        file.transferTo(new File(realPath +"/"+ file.getOriginalFilename()));

        return "redirect:/index.jsp";
    }

```

CommonsMultipartFile file是打包好的表单文件

文件上传：

- 文件名
- 上传文件的url- > path
- 保存文件的地址 -> realPath
- 文件读出、写入（方法一自己实现，方法二用现成方法transferTo()实现）

上传结果：

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211212212803926.png" alt="image-20211212212803926" style="zoom:50%;" />



5、文件下载的Controller类

```java
    @RequestMapping("/download/{fileName:.+}")
    public String  download(@PathVariable String fileName, HttpServletResponse response, HttpServletRequest request) throws IOException {

        //下载地址
        String path = request.getServletContext().getRealPath("/upload");

        // Response响应头
        response.reset();
        response.setCharacterEncoding("UTF-8");
        response.setContentType("multipart/form-data");
        response.setHeader("Content-Disposition", "attachment;fileName" + URLEncoder.encode(fileName,"UTF-8"));

        // 读取、写出文件
        File file = new File(path, fileName);
        InputStream is = new FileInputStream(file); //文件输入流
        OutputStream os = response.getOutputStream();
        System.out.println("下载文件：" + file);

        int len = 0;
        byte[] buffer = new byte[1024];
        while((len=is.read(buffer)) != -1){
            os.write(buffer,0,len);
            os.flush();
        }

        os.close();
        is.close();
        return "redirect:/index.jsp";
    }
}
```

文件下载：

- 下载文件的文件地址和文件名
  - 文件名参数的格式需要这么写，否则可能会忽略扩展名：@RequestMapping("/download/{fileName:.+}")
- 设置Response响应头
- 文件读出、写入



## 7 SpringBoot

一个开源的轻量级框架，微服务架构，打破之前所有内容都在一个工程的架构方式（开发人员不再需要定义样板化的配置），把每个功能元素独立出来，并根据需要进行动态组合。

节省了调用资源、每个功能元素的服务都是一个可替换、可独立升级的软件代码。

相比于SSM，用Maven项目的pom文件和注解代替了繁琐的xml配置文件



### 7.1 工程创建与运行



#### 7.1.1 工程创建



方法一：官网创建

网址：https://start.spring.io/

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211214173401736.png" alt="image-20211214173401736" style="zoom: 33%;" />

定制后下载工程压缩包，解压后导入IDEA



方法二：IDEA创建

本质上是内嵌了官网

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211214173624858.png" alt="image-20211214173624858" style="zoom:50%;" />

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211214173704628.png" alt="image-20211214173704628" style="zoom:50%;" />



#### 7.1.2 工程结构

1、入口类：java目录下有一个自动生成的类，类名同工程名

```java
// 程序主入口（本质是Spring的一个组件）
@SpringBootApplication
public class HelloSpringBootApplication {

	public static void main(String[] args) {
		SpringApplication.run(HelloSpringBootApplication.class, args);
	}

}
```

其是程序的主入口。

@SpringBootApplication的本质是Spring的一个@Component组件

规定：其他包要放在和此类的同级目录下：

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211214180343896.png" alt="image-20211214180343896" style="zoom:67%;" />



2、resources目录下有一个application.properties配置文件

3、工程的pom.xml

```xml
<!--继承该依赖管理、控制版本、打包等内容-->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>  
    <version>2.6.1</version>
    <relativePath/>
</parent>

<!--依赖-->
<dependencies>

    <!--web依赖，实现HTTP接口：内嵌Tomcat（默认）、dispatcherServlet、xml等，包含了Spring MVC-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <!--单元测试-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

</dependencies>

<build>
    <plugins>
        <!--插件-：把SpringBoot应用打成jar包直接运行-->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <excludes>
                    <exclude>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                    </exclude>
                </excludes>
            </configuration>
        </plugin>
    </plugins>
</build>
```

pom.xml中已经导入了定制的工程内容。

其中spring-boot-starter-web实现HTTP接口：内嵌了SpringMVC项目需要配置的Tomcat、dispatcherServlet、xml等内容，无需自建



#### 7.1.3 工程运行

工程运行的入口是入口类，运行后内嵌的Tomcat就会直接工作和部署。

也可以使用上述插件可以把工程打成jar包，直接运行jar包文件

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211214181009923.png" alt="image-20211214181009923" style="zoom:67%;" />





#### 7.1.4 修改banner

在resources目录下新建banner.txt文件，把内容放进去即可。

ASCII艺术字：https://www.bootschool.net/ascii

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211214201843621.png" alt="image-20211214201843621" style="zoom:50%;" />



### 7.2 自动装配原理

1、大量的核心依赖在父工程中，因此我们在引入一些依赖时，不需要指定版本

pom.xml

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.6.1</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
```



2、启动器

将所有的功能场景都设置成立一个个启动器，需要什么功能只要引入对应的启动器即可

默认的是spring-boot-starter，引入spring-boot-starter-web就会自动导入web环境的所有依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>
```

​		部分依赖：https://docs.spring.io/spring-boot/docs/current/reference/html/using.html#using.build-systems.starters



3、主函数：@SpringBootApplication注解

- 元注解1：@SpringBootConfiguration

  说明是SpringBoot的配置，本质是Spring的一个组件

  各层元注解：

  ```java
  @SpringBootConfiguration       // SpringBoot配置
  	@Configuration             // Spring 配置类
  		@Component             // Spring 组件
  ```

- 元注解2：@EnableAutoConfiguration 

  自动导入包

  ```java
  @EnableAutoConfiguration       // 自动配置
  	@AutoConfigurationPackage  // 自动配置包
  	@Import({AutoConfigurationImportSelector.class}) // 自动配置导入选择
  ```

  - @AutoConfigurationPackage

    元注解：`@Import({Registrar.class})`，Registrar是定义的注册表类，用来记录要导入包的信息

  - @Import({AutoConfigurationImportSelector.class})

    AutoConfigurationImportSelector类，**核心**，导入配置的自动选择器

    ```java
    // AutoConfigurationImportSelector类 
    
    // 方法：getAutoConfigurationEntry
    protected AutoConfigurationImportSelector.AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
        if (!this.isEnabled(annotationMetadata)) {
            return EMPTY_ENTRY;
        } else {
            AnnotationAttributes attributes = this.getAttributes(annotationMetadata);
            List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);
            configurations = this.removeDuplicates(configurations);
            Set<String> exclusions = this.getExclusions(annotationMetadata, attributes);
            this.checkExcludedClasses(configurations, exclusions);
            configurations.removeAll(exclusions);
            configurations = this.getConfigurationClassFilter().filter(configurations);
            this.fireAutoConfigurationImportEvents(configurations, exclusions);
            return new AutoConfigurationImportSelector.AutoConfigurationEntry(configurations, exclusions);
        }
    }
    
    ```

    其中的`List<String> configurations = this.getCandidateConfigurations();`获取了所有自动配置的实体

    该函数的定义：

    ```java
    // getCandidateConfigurations()方法
    protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
        List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
        Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
        return configurations;
    }
    ```

    其中的`loadFactoryNames()`方法调用了`getSpringFactoriesLoaderFactoryClass()`方法，

    后者选中了`@EnableAutoConfiguration`标注的类，也即选中了`@SpringBootApplication`的主函数所在类

    ```java
    // `getSpringFactoriesLoaderFactoryClass()方法
    protected Class<?> getSpringFactoriesLoaderFactoryClass() {
        return EnableAutoConfiguration.class;
    }
    ```

    而前者`loadFactoryNames()`根据获得的目标，进行实际配置的加载

    ```java
    public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
        ClassLoader classLoaderToUse = classLoader;
        if (classLoader == null) {
            classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
        }
    
        String factoryTypeName = factoryType.getName();
        return (List)loadSpringFactories(classLoaderToUse).getOrDefault(factoryTypeName, Collections.emptyList());
    }
    ```

    其中调用的loadSpringFactories()方法传入了`AutoConfigurationImportSelector`类的类加载器，进行了实际配置的加载，

    `getResources()`获取项目资源，并规定了导入项目资源jar包下的META-INF目录下的spring.factories文件

    `PropertiesLoaderUtils.loadProperties()`则将该文件的内容都读到了一个Properties对象中（用循环遍历所有目标文件）

    <img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211217145437018.png" alt="image-20211217145437018" style="zoom:80%;" />

    而所有的自动配置类，都在springboot的spring-boot-autoconfigure包下

    <img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211217150423132.png" alt="image-20211217150423132" style="zoom:67%;" />

    目标文件spring.factories里面是各种配置类，每个类都包含了对应的各种部件。

    如WebMvcAutoConfiguration里就有SpringMVC里需要的配置（如过滤器、视图解析器）。

    ![image-20211217150555225](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211217150555225.png)

    spring.factories里虽然所有的配置都有了，但不会都生效，因为有@ConditionalOnXXX注解，只有条件都满足才会生效

    <img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211217151035447.png" alt="image-20211217151035447" style="zoom:67%;" />

    

#### 总结

1、SpringBoot启动时，从对应包路径下的/MEF-INF/spring.factories获取配置

2、将满足条件（@ConditionalOnXX）的配置注入容器

3、整合了JavaEE，解决方案和自动配置的内容在spring-boot-autoconfigure包中

4、每个带有AutoConfiguration的类，就包含了对应场景需要的所有组件（用@Bean标注，自动注入）

​      需要的组件集合会以类名返回，注入容器，如：org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration



### 7.3 主启动类运行原理

1、工程启动时，新建SpringApplication类对象

- 判断项目是普通项目还是Web项目

- 查找所有的可用初始化器，存入initializers属性

- 查找所有的应用程序监听器，存入listeners属性

- （重点）设置main方法的定义类，即设定项目的主类



2、执行`SpringApplication.run()`

- headless系统属性设置
- 初始化、启动监听器
- 装配环境参数（如application.yml）
- 打印banner
- 配置上下文



### 7.4 配置文件和属性赋值

推荐使用application.yml或application.yaml



#### 7.4.1 配置文件写法

1、传统写法

格式：`name=value`

```properties
server.port=8081
```



2、yaml写法

yaml不是一种语言，是一种配置格式

格式：`name: value`（":"后有空格）

```yaml
server:
  port: 8081
```



- 优势是可以清晰的配置一个对象或数组，而properties文件只能定义键值对：

对象：

```yaml
#一般写法
user:
  name: foster
  id: 1

#行内写法（类似JavaScript）
user: {name: foster, id: 1}
```

数组格式：

```yaml
#一般写法
pets:
  - cat
  - dog
  - cow
  
#行内写法（类似JavaScript）
pets: [cat,dog,cow]
```



#### 7.4.2 属性赋值

**1、在属性或其set方法上加上`@Value()`注解，创建该对象时，用注解@Autowired注入**

```java
@Component
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Dog {

    @Value("dz")
    private String name;

    @Value("2")
    private Integer age;
}
```

```java
@Autowired
Dog dog;
```



**2、yml配置文件设置（推荐）**

在类定义中，用`@ConfigurationProperties(prefix = "")`进行设置，prefix的值对应yml文件中的配置名

```java
@Component
@Data
@AllArgsConstructor
@NoArgsConstructor
@ConfigurationProperties(prefix = "person1")
public class Person {
    private String name;
    private Integer age;
    private boolean happy;
    private Date birth;
    private Map<String, Object> maps;
    private List<Object> list;
    private Dog dog;
}
```

yml配置文件

```yaml
person1:
  name: foster
  age: 21
  happy: true
  birth: 2000/11/18
  map: {k1: v1, k2: v2}
  list:
    - game
    - code
    - yugioh
  dog:
    name: dj
    age: 3
```

使用时还是需要@Autowired注解

```java
@Autowired
Person p1;
```



高级用法

- 生成各种类型的随机数

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211217163113868.png" alt="image-20211217163113868" style="zoom:80%;" />

- 伪三元运算

```yaml
dog:
  name: ${person1.hello:hello}_1 # 如果存在person1.hello，则取其值；否则用后边的值hello
  age: 3
```



注：需要在pom.xml中导入依赖，但不导入也能正常运行

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```



**3、Properties配置文件设置**

类定义中，用`@PropertySource(value = "")`指定properties配置文件，之后在每个属性的@Value中，就可以使用配置文件中的内容

```java
@Component
@Data
@AllArgsConstructor
@NoArgsConstructor
@PropertySource(value = "classpath:person.properties")
public class Person {
    
    @Value("${name}")
    private String name;
}
```



对比：

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211217164124201.png" alt="image-20211217164124201" style="zoom:67%;" />



松散绑定：yml中的-命名和java类中的驼峰命名法，如first-name和firstName



#### 7.4.3 JSR303校验

导入spring-boot-starter-validation的依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

在java类上加上@Validated注解，在需要校验的属性上加上对应的校验注解

```Java
@Validated
public class Person {

    @Email
    private String name;
}
```

![img](https://img-blog.csdnimg.cn/20200913110853722.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyMzUyNzc3,size_16,color_FFFFFF,t_70)



#### 7.4.4 多环境配置

配置文件可以存在的位置，优先级1-4由高至低，默认为4

（file为项目文件目录，classpath为项目文件的resources目录） 

![image-20211217223336430](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211217223336430.png)

需要指定环境配置时，需要进行环境配置的切换

**1、properties配置文件**

以application-XXX.properties命名，不同的properties配置文件分别进行不同的配置

要切换时，再主配置文件application.properties中设置

```properties
# 选择激活配置文件
spring.profiles.active=XXX
```

XXX与application-XXX.properties中的XXX对应

**2、yml配置文件**

yml配置文件中，一个yml文件可定义多个环境配置

```yaml
server:
  port: 8081

spring:
  profiles:
    active: test  # 激活对应的环境配置

---
spring:
  config:
    activate:
      on-profile: dev

server:
  port: 8082
---
spring:
  config:
    activate:
      on-profile: test

server:
  port: 8083
---
```

最上边的是默认配置，之后可用`---`划分各环境，每个环境用<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211217230029243.png" alt="image-20211217230029243" style="zoom: 40%;" />命名（高版本，低版本springboot语法不同）。

如果需要切换成对应的环境配置，只要在默认配置中指定即可



#### 7.4.5 自动配置原理（重点）

配置文件中的内容是如何生效的

对于spring.factories中可被加载的每个XXXAutoConfiguration类，都有注解：

```
# 表明是配置类
@Configuration    

# 核心实现
@EnableConfigurationProperties({XXXProperties.class})  

# 控制本类是否有效
@ConditionalOnXXX  
```

其中@EnableConfigurationProperties({XXXProperties.class})指定了该配置类对应的实际功能类(一般是XXXProperties)

如在用于编码的HttpEncodingAutoConfiguration配置类中：

```java
@EnableConfigurationProperties({ServerProperties.class})
public class HttpEncodingAutoConfiguration {
    private final Encoding properties;

    public HttpEncodingAutoConfiguration(ServerProperties properties) {
        this.properties = properties.getServlet().getEncoding();
    }
}
```

指定了ServerProperties类，并用其对象对属性进行了初始化。在该类的其他方法中，便是用这个属性进行操作的

而在ServerProperties类中：

```java
@ConfigurationProperties(
    prefix = "server",
    ignoreUnknownFields = true
)
public class ServerProperties {
    private Integer port;
    private InetAddress address;
    @NestedConfigurationProperty
    private final ErrorProperties error = new ErrorProperties();
    private ServerProperties.ForwardHeadersStrategy forwardHeadersStrategy;
    ……
}
```

**用@ConfigurationProperties注解指定了前缀，这就对应了配置文件中的属性前缀，而该类的属性就是配置文件的属性**

如指定前缀为server，属性有port，就对应配置文件的：

```yaml
server:
	port: 8080
```



因此，想进行配置但不知道要配什么时，可以去spring.factories中找到对应的类，按上述步骤找到其对应的XXXProperties类，之后便可按其设定的前缀和属性，在配置文件中进行配置了



##### 总结

1、SpringBoot启动时会从spring.factories加载大量自动配置类（XXXAutoConfiguration）

2、查看自动配置类，如果我们需要的功能组件在里面已经有定义（有@Bean注入容器了），就不用再自己写了

3、实际注入容器时，组件中的属性需要设置，SpringBoot会自动地从配置文件（properties、yml等）中获取

4、如果需要在配置文件中设置某个功能的属性，可以去该功能的自动配置类（XXXAutoConfiguration）指定的功能类（XXXProperties）中，了解到其前缀和拥有的属性，便知道怎么写配置文件了

5、如果要使用条件不满足，未生效的自动配置类，只需导入对应的starter依赖即可



补：如何查看加载的自动配置类是否生效

在配置文件中打开相应日志：

```yaml
debug: true
```

运行后会输出对应的日志，其中：

| 日志模块              | 含义                 |
| --------------------- | -------------------- |
| Positive matches      | 生效的自动配置类     |
| Negative matches      | 没有生效的自动配置类 |
| Exclusions            |                      |
| Unconditional classes |                      |



### 7.5 静态资源获取



#### 1、自定义（不推荐）

```properties
# 定义匹配模式，url为什么时去获取静态资源
spring.mvc.static-path-pattern=/a/**

# 设定静态资源路径，去哪里获取静态资源
spring.resources.static-locations = classpath:/static,classpath:/public,classpath:/resources,classpath:/META-INF/resources
```

(yml格式同理)



#### 2、webjars

一般适合引入的第三方jar包资源（如JQuery），各种资源见https://www.webjars.org/

根据WebMvcAutoConfiguration自动配置类的方法：

```java
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    if (!this.resourceProperties.isAddMappings()) {
        
        // 有自定义的路径时，webjars和默认静态路径失效
        logger.debug("Default resource handling disabled");
    } else {
        
        // webjars方法
        this.addResourceHandler(registry, "/webjars/**", "classpath:/META-INF/resources/webjars/");
        
        // 默认的静态资源路径
        this.addResourceHandler(registry, this.mvcProperties.getStaticPathPattern(), (registration) -> {
            registration.addResourceLocations(this.resourceProperties.getStaticLocations());
            if (this.servletContext != null) {
                ServletContextResource resource = new ServletContextResource(this.servletContext, "/");
                registration.addResourceLocations(new Resource[]{resource});
            }

        });
    }
}
```

从`this.addResourceHandler(registry, "/webjars/**", "classpath:/META-INF/resources/webjars/");`得知，访问路径最后为：`/webjars/资源名` 时，会访问项目External Library的对应jar包的`/META-INF/resources/webjars/`目录下的同名资源



匹配模式：/webjars/**

静态资源路径：classpath:/META-INF/resources/webjars/



#### 3、默认的静态资源路径（推荐）

- 匹配模式：

  2中的代码段的`getStaticPathPattern()`使用了SpringBoot默认的静态资源路径，跳转后发现定义在WebMvcProperties类中：

```java
public class WebMvcProperties {
    ……
    private String staticPathPattern = "/**";
    ……
}
```

​		说明url中所有的静态资源请求都会被接受



- 静态资源路径：

  WebMvcAutoConfiguration自动配置类中有定义

```java
public class WebMvcAutoConfiguration{
    ……
	private final Resources resourceProperties;
    ……
}
```

​		其中有明确的默认设置：

```java
public static class Resources {
    private static final String[] CLASSPATH_RESOURCE_LOCATIONS = new String[]{"classpath:/META-INF/resources/", "classpath:/resources/", "classpath:/static/", "classpath:/public/"};
    ……
}
```

​		即：项目的`resources`目录下的`META-INF/resources`目录、`resources`目录、`static`目录、`public`目录

​		没有的可以自己新建，优先级和定义顺序一致。

​		因为匹配模式是"/**"，所以对任一一个资源请求，SpringBoot会按优先级从上述目录一个一个地找



### 7.6 web应用实现



#### 7.6.1 首页

<u>把名为index.html的文件放在默认或自定义的静态资源路径下即可。</u>只要存在首页文件，项目运行后会自动跳转到该页面

源码位于WebMvcAutoConfiguration类中

```java
@Bean
public WelcomePageHandlerMapping welcomePageHandlerMapping(ApplicationContext applicationContext, FormattingConversionService mvcConversionService, ResourceUrlProvider mvcResourceUrlProvider) {
    WelcomePageHandlerMapping welcomePageHandlerMapping = new WelcomePageHandlerMapping(new TemplateAvailabilityProviders(applicationContext), applicationContext, this.getWelcomePage(), this.mvcProperties.getStaticPathPattern());
    welcomePageHandlerMapping.setInterceptors(this.getInterceptors(mvcConversionService, mvcResourceUrlProvider));
    welcomePageHandlerMapping.setCorsConfigurations(this.getCorsConfigurations());
    return welcomePageHandlerMapping;
}

private Resource getWelcomePage() {
    String[] var1 = this.resourceProperties.getStaticLocations();  // 默认的静态资源路径
    int var2 = var1.length;

    for(int var3 = 0; var3 < var2; ++var3) {
        String location = var1[var3];
        Resource indexHtml = this.getIndexHtml(location);
        if (indexHtml != null) {
            return indexHtml;
        }
    }

    ServletContext servletContext = this.getServletContext();
    if (servletContext != null) {
        return this.getIndexHtml((Resource)(new ServletContextResource(servletContext, "/")));
    } else {
        return null;
    }
}
```

前者调用后者得到了路径。



#### 7.6.2 图标

将选好的图片改为 .ioc 为后缀，一般命名 favicon.ico，并将其放在 resources 文件夹下的 static 文件夹下（如果没有可以自己创建）。

- SpringBoot 2.2.0+ 版本之后

  在对应的页面文件中添加：

  ```html
  <link rel="icon" href="/favicon.ico">
  ```



- SpringBoot 2.2.0+ 版本之前

  关闭SpringBoot默认的图标：

  ```properties
  spring.mvc.favicon.enabled=false
  ```

  或

  ```yaml
  spring:
    mvc:
      favicon:
        enabled: false
  ```

  

如果没有显示可以清空浏览器缓存试试



#### 7.6.3 模版引擎（Thymeleaf）

SpringBoot推荐使用Thymeleaf，见前端笔记



#### 7.6.4 扩展SpringBoot自带的SpringMVC



在java下的config中自定义设置类，实现对应设置的接口。

自定义扩展SpringMVC时，实现WebMvcConfigurer接口：

```java
@Configuration
public class MyMvcConfig implements WebMvcConfigurer
```

不要在类上加`@EnableWebMvc`注解，因为MVC的自动配置类`WebMvcAutoConfiguration`有条件注解`@ConditionalOnMissingBean({WebMvcConfigurationSupport.class})`，存在该类时自动配置全部无效，而`@EnableWebMvc`引入了该类的子类。



**方法一**

要实现什么功能的扩展，就自己创建一个类，实现该功能的接口类，并注入到Spring容器中。

以视图解析器为例：

```java
@Configuration
public class MyMvcConfig implements WebMvcConfigurer {

    // 将自定义的视图解析器注入Spring容器，SpringBoot会将其自动地和其他视图解析器装配到一起
    @Bean
    public ViewResolver myViewResolver(){
        return new MyViewResolver();
    }

    // 自定义视图解析器
    public static class MyViewResolver implements ViewResolver{
        @Override
        public View resolveViewName(String viewName, Locale locale) throws Exception {
            return null;
        }
    }
}
```

SpringBoot自动处理自定义组件的策略：

​	对于有自定义的组件，会优先用自定义的；

​	如果自定义的组件和同类的自带组件可以同时存在，会自动的集合到一起



**方法二**

调用管理配置的登记类XXXRegistry，将自定义配置添加到登记的配置中

以视图解析器为例：

```java
import org.springframework.web.servlet.config.annotation.ViewControllerRegistry;

@Configuration
public class MyMvcConfig implements WebMvcConfigurer {

    // 视图跳转
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/ghy").setViewName("test");
    }
}
```

在视图解析器登记类ViewControllerRegistry中登记自定义的视图解析器（"/ghy" -> test.html）



#### 7.6.5 国际化

实现用页面的语言按钮切换语言

1、新建properties文件

**将同一个信息的不同语言版本放到properties文件中读取；**

i18n是internationalization的缩写；

在IDEA中，对于同一个功能的配置文件XX.properties，再新建其不同语言的配置文件（XX_语言后缀.properties）会自动合并；

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211219102640824.png" alt="image-20211219102640824" style="zoom:67%;" />

下载Resource Bundle Editor插件，实现可视化编辑properties bundle

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211219095427046.png" alt="image-20211219095427046" style="zoom: 50%;" />

2、编辑properties文件

之后打开任一个properties文件，选择Resource Bundle，便可对所有语言properties文件的同一属性进行分别的赋值

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211219103228816.png" alt="image-20211219103228816" style="zoom:50%;" />

3、登记到项目配置文件application.yml

要使用上述文件，需要将之登记到项目配置文件中：

```yaml
# application.yml
spring:
  messages:
    basename: i18n/login
```

4、在html中读取内容

在需要显示目标文字的地方，用thymeleaf读取properties文件，元素分类为消息 `#{}`

```html
<body class="text-center">
    <form class="form-signin">
        <img class="mb-4" th:src="@{/assets/brand/bootstrap-solid.svg}" alt="" width="72" height="72">
        <h1 class="h3 mb-3 font-weight-normal" th:text="#{login.tip}">Please sign in</h1>
        <input type="text" class="form-control" th:placeholder="#{login.username}" required autofocus>
        <input type="password" class="form-control" th:placeholder="#{login.password}" required>
        <div class="checkbox mb-3">
            <label>
                <input type="checkbox" value="remember-me"> [[#{login.remember}]]
            </label>
        </div>
        <button class="btn btn-lg btn-primary btn-block" type="submit">[[#{login.btn}]]</button>
        <p class="mt-5 mb-3 text-muted">&copy; 2017-2018</p>
        <a class="btn btn-sm" th:href="@{/index.html(l='zh_CN')}">中文</a>
        <a class="btn btn-sm" th:href="@{/index.html(l='en_US')}">English</a>
    </form>
</body>
```

这样，浏览器就会根据本身的语言设置，获取不同的语言信息（取决于客户端请求的请求头中的语言）

5、用按钮手动切换语言

给每个语言按钮加上超链接，跳转url仍为本界面，但增加不同的语言请求参数（Thymeleaf中参数写在url后的括号里）

```html
<a class="btn btn-sm" th:href="@{/index.html(l='zh_CN')}">中文</a>
<a class="btn btn-sm" th:href="@{/index.html(l='en_US')}">English</a>
```

在java下的config目录下自定义地区解析器，获取客户端请求，根据其语言请求返回不同的语言环境对象Locale

```java
// 自定义地区解析器
public class MyLocaleResolver implements LocaleResolver {

    // 解析请求
    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        // 获取语言请求参数
        String language = request.getParameter("l");

        // 如果有语言请求
        if(!StringUtils.isEmpty(language)){
            String[] split = language.split("_");
            return new Locale(split[0], split[1]);
        }

        // 否则用默认的
        return Locale.getDefault();
    }

    @Override
    public void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale) {

    }
}
```

将自定义的地区解析加到项目对SpringBoot自带的SpringMVC的扩展中，用@Bean注入容器，运行时会自动生效

```java
@Configuration
public class MyMvcConfig implements WebMvcConfigurer {
    @Bean
    public LocaleResolver localeResolver(){
        return new MyLocaleResolver();
    }
}
```



#### 7.6.6 登录

##### 登录功能

在登录请求处设置登录请求的url

```html
<form class="form-signin" th:action="@{/user/login}">
```

新建对应的Controller类，定义一个方法对应该url，需要获取的内容写在方法参数中。

进行实际的业务后，return转发或return"redirect:/"重定向   （登录最好用重定向，可以屏蔽登录信息参数）

```java
@Controller
public class LoginController {

    @RequestMapping("/user/login")
    public String login(@RequestParam("username") String username, @RequestParam("password") String password, Model model, HttpSession session){
        if("123456".equals(password)){
            session.setAttribute("loginUser", username);
            return "redirect:/main.html";  // 重定向
        }
        else{
            model.addAttribute("msg", "用户名或密码错误");
            return "index";
        }
    }
}
```

在视图解析器中要实现该重定向

```java
@Override
public void addViewControllers(ViewControllerRegistry registry) {
    registry.addViewController("/").setViewName("index");
    registry.addViewController("/index.html").setViewName("index");
    registry.addViewController("/main.html").setViewName("dashboard");  // 重定向登录界面
}
```



##### 拦截器

登录后的页面只有登录后才能访问到，因此需要拦截器拦截未登录的。

原理：登录后修改Session，拦截器检查Session



1、登录成功修改Session

需要在方法参数中加上`HttpSession session`

```java
@Controller
public class LoginController {

    @RequestMapping("/user/login")
    public String login(@RequestParam("username") String username, @RequestParam("password") String password, Model model, HttpSession session){
        if("123456".equals(password)){
            session.setAttribute("loginUser", username);
            return "redirect:/main.html";  // 重定向
        }
        ……
    }
}
```

2、拦截器检查Session

自定义拦截器类，实现拦截器接口

```java
public class LoginHandlerInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        Object loginUser = request.getSession().getAttribute("loginUser");
        if(loginUser == null){
            request.setAttribute("msg", "用户未登录");
            request.getRequestDispatcher("/index.html").forward(request, response);  // 如果未登录，转发回登录界面
            return false;
        }
        return true;
    }
}
```

3、部署拦截器

在自定义的SpringMVC扩展中增加自定义拦截器，设置拦截url（addPathPatterns）和不拦截的url（excludePathPatterns）。

不拦截的url中一般都要加上静态资源，否则css、js等静态资源也会被拦截，导致页面内容不正常。

```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(new LoginHandlerInterceptor()).addPathPatterns("/**").excludePathPatterns("/index.html", "/", "/user/login", "/assets/**");
}
```



#### 7.6.7 增删查改（CRUD）

共有操作：

- 后端：新建对应类的Controller，负责根据url请求，调用真正实现业务的代码，接收业务结果，并转发/重定向到目标页面

  ```java
  @Controller
  public class EmployeeController {
  
      @Autowired
      EmployeeDao employeeDao;
  
      @RequestMapping("/emps")
      public String list(Model model){
          Collection<Employee> employees = employeeDao.getEmployees();
          model.addAttribute("employees", employees);
          return "employee/list.html";
      }
      ……
  }
  ```

- 前端：新建对应的页面，先用thymeleaf插入公共片段，再进行具体页面设置。最后接收后端的数据并展示

  

##### 1、搜索

- 后端方法：

```java
@RequestMapping("/emps")
public String list(Model model){
    Collection<Employee> employees = employeeDao.getEmployees();
    model.addAttribute("employees", employees);
    return "employee/list.html";
}
```

- 前端展示：

  使用`th:each`

```html
<table class="table table-striped table-sm">
    <thead>
        <tr>
            <th>id</th>
            <th>lastName</th>
            <th>email</th>
            <th>gender</th>
            <th>department</th>
            <th>birth</th>
            <th>操作</th>
        </tr>
    </thead>
    <tbody>
        <tr th:each="employee:${employees}">
            <td>[[${employee.getId()}]]</td>
            <td>[[${employee.getLastName()}]]</td>
            <td>[[${employee.getEmail()}]]</td>
            <td>[[${employee.getGender()==0?'男':'女'}]]</td>
            <td>[[${employee.getDepartment().getDepartmentName()}]]</td>
            <td>[[${#dates.format(employee.getBirth(),'yyyy-MM-dd HH:mm:ss')}]]</td>
            <td>
                <a class="btn btn-sm btn-primary">编辑</a>
                <a class="btn btn-sm btn-danger">删除</a>
            </td>
        </tr>
    </tbody>
</table>
```



##### 2、增加

- 后端方法：

用get方法进入添加页面，在添加页面用post提交，进行实际的修改

```java
@GetMapping("/addEmployee")
public String toAddPage(Model model){
    model.addAttribute("departments", departmentDao.getDepartments());
    return "employee/add.html";
}

@PostMapping("/addEmployee")
public String addEmployee(Employee employee){
    employeeDao.addEmployee(employee);
    return "redirect:/emps";
}
```

- 前端展示：

  先在之前的页面增加添加内容的按钮（这种跳转必是get方式）

```html
<a class="btn btn-sm btn-success" th:href="@{/addEmployee}">添加员工</a>
```

​	   添加页面中，用form表格收集信息，设定方法为post

​	（form的input必须加上name，并且此name要和controller接收方法的参数对应；如果返回对象，要和该类的属性同名）

​	（该表格的提交内容为一个类，其department属性为一个对象，为此，我们让其显示department名，而实际传入的是department的id，SpringBoot也会将之装配到一个department对象中，但是没有department名，因此后端要手动加上department名）

```html
<form th:action="@{/addEmployee}" method="post">
    <div class="form-group">
        <label>LastName</label>
        <input type="text" name="lastName" class="form-control" placeholder="姓名">
    </div>
    <div class="form-group">
        <label>Email</label>
        <input type="email" name="email" class="form-control" placeholder="1362452972@qq.com">
    </div>
    <div class="form-group">
        <label>Gender</label><br>
        <div class="form-check form-check-inline">
            <input class="form-check-input" type="radio" name="gender" value="1">
            <label class="form-check-label">男</label>
        </div>
        <div class="form-check form-check-inline">
            <input class="form-check-input" type="radio" name="gender" value="0">
            <label class="form-check-label">女</label>
        </div>
    </div>
    <div class="form-group">
        <label>department</label>
        <select class="form-control" name="department.id"><!--department.id是该对象的一个属性-->
            <option th:each="department:${departments}" th:text="${department.getDepartmentName()}" th:value="${department.getId()}"></option>
        </select>
    </div>
    <div class="form-group">
        <label>Birth</label>
        <input type="text" name="birth" class="form-control" placeholder="2200/11/21">
    </div>
    <button type="submit" class="btn btn-primary">添加</button>
</form>
```



##### 3、修改

- 后端方法

  用get方法进入修改页面，在修改页面提交修改信息，进行实际的修改

  （进入修改页面时，用RestFul风格传送id，以在修改页面展示未修改的原数据）

```java
// 修改
@GetMapping("/employee/{id}")
public String toUpdatePage(@PathVariable("id")Integer id, Model model){
    model.addAttribute("employee",employeeDao.getEmployee(id));
    model.addAttribute("departments", departmentDao.getDepartments());
    return "employee/update";
}

@RequestMapping("/employeeUpdate")
public String update(Employee employee){
    Department department = employee.getDepartment();
    department.setDepartmentName(departmentDao.getDepartment(employee.getDepartment().getId()).getDepartmentName());
    employee.setDepartment(department);
    employeeDao.updateEmployee(employee);
    return "redirect:/emps";
}
```

- 前端展示

  先在之前的页面增加修改内容的按钮（这种跳转必是get方式）

```html
<a class="btn btn-sm btn-primary" th:href="@{'/employee/'+${employee.getId()}}">编辑</a>
```

修改页面中，用form表格收集信息，设定方法为post

（用thymeleaf的th:value展示原数据，checkbox、select等可以用thymeleaf表达式完成，时间要设置格式）

```html
<form th:action="@{/employeeUpdate}" method="post">
    <div class="form-group">
        <input type="hidden" name="id" class="form-control" th:value="${employee.getId()}">
    </div>
    <div class="form-group">
        <label>LastName</label>
        <input type="text" name="lastName" class="form-control" th:value="${employee.getLastName()}">
    </div>
    <div class="form-group">
        <label>Email</label>
        <input type="email" name="email" class="form-control" th:value="${employee.getEmail()}">
    </div>
    <div class="form-group">
        <label>Gender</label><br>
        <div class="form-check form-check-inline">
            <input class="form-check-input" type="radio" name="gender" value="0" th:checked="${employee.getGender()==0}">
            <label class="form-check-label">男</label>
        </div>
        <div class="form-check form-check-inline">
            <input class="form-check-input" type="radio" name="gender" value="1" th:checked="${employee.getGender()==1}">
            <label class="form-check-label">女</label>
        </div>
    </div>
    <div class="form-group">
        <label>department</label>
        <select class="form-control" name="department.id">
            <option th:each="department:${departments}" th:text="${department.getDepartmentName()}" th:value="${department.getId()}" th:selected="${employee.getDepartment().getId()==department.getId()}"></option>
        </select>
    </div>
    <div class="form-group">
        <label>Birth</label>
        <input type="text" name="birth" class="form-control" th:value="${#dates.format(employee.getBirth(),'yyyy-MM-dd HH:mm')}">
    </div>
    <button type="submit" class="btn btn-primary">修改</button>
</form>
```



##### 4、删除

- 后端方法

  后端接收带有参数的删除请求，调用方法实现业务后返回

```java
@GetMapping("/delete/{id}")
public String deleteEmployee(@PathVariable("id")Integer id, Model model){
    employeeDao.deleteEmployee(id);
    return "redirect:/emps";
}
```



-  前端展示

  添加删除的请求链接即可，要加上参数id

```html
<a class="btn btn-sm btn-danger" th:href="@{'/delete/'+${employee.getId()}}">删除</a>
```



#### 7.6.8 注销

- 后端方法

  在对应的Controller中自定义接收注销请求url的方法，删除session后返回首页

```java
@Controller
@RequestMapping("/user")
public class LoginController {
    
    ……

    @RequestMapping("/logout")
    public String logout(HttpSession session){
        session.invalidate();
        return "redirect:/index.html";
    }
}
```

- 前端展示

  加上注销链接即可

```html
<a class="nav-link" th:href="@{/user/logout}">退出登录</a>
```



#### 7.6.9 报错

在templates目录下加一个error目录，其中为以各个错误代码命名的html文件，应用报错时便会自动转到对应错误代码的页面

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211219194236444.png" alt="image-20211219194236444" style="zoom: 67%;" />







### 7.7 整合数据库

对于数据访问层，无论是关系型数据库（SQL）还是非关系型数据库（如Redis），SpringBoot底层都用Spring Data统一处理



#### 7.7.1 默认数据源和jdbc

##### 1、环境配置

创建工程时勾选jdbc和对应的数据库驱动

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211220104309747.png" alt="image-20211220104309747" style="zoom: 50%;" />

或手动导入对应依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>
```



##### 2、数据库连接配置

在项目配置文件（properties或yml中设置）：

```yml
spring:
  datasource:
    username: root
    password: '001118'
    url: jdbc:mysql://localhost:3306/mybatis?useSSL=true&useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC
    driver-class-name: com.mysql.cj.jdbc.Driver
```

**注：yml中，password的值要用单引号引起来，否则连接不上**



##### 3、默认数据源

hikari（class com.zaxxer.hikari.HikariDataSource），速度很快

使用：DataSource类的Bean已经注入了Spring容器，直接取用即可；

​			之后直接getConnection()即可建立连接，开始进行传统的jdbc操作（见笔记1.2.1）

```java
@Autowired
DataSource dataSource;

Connection conn = dataSource.getConnection();  // 建立连接
```



##### 4、JdbcTemplate

SpringBoot内置了jdbc的模板类JdbcTemplate，使用它可以代替传统的jdbc操作

```java
@Controller
public class jdbcController {

    @Autowired
    JdbcTemplate jdbcTemplate;

    // 查询数据库的所有信息
    @GetMapping("/users")
    @ResponseBody
    public List<Map<String, Object>> userList(){
        String sql = "select * from mybatis.user";
        List<Map<String, Object>> list = jdbcTemplate.queryForList(sql); // 无实体类时获取表数据
        return list;
    }

    @GetMapping("/add")
    public String addUser(){
        String sql = "insert into mybatis.user(id, name, pwd) values(5,'issac','abcdef')";
        jdbcTemplate.update(sql);
        return "redirect:/users";
    }

    @GetMapping("/delete/{id}")
    public String deleteUser(@PathVariable("id") int id){
        String sql = "delete from mybatis.user where id=?";
        jdbcTemplate.update(sql, id);
        return "redirect:/users";
    }

    @GetMapping("/update/{id}")
    public String updateUser(@PathVariable("id") int id){
        String sql = "update mybatis.user set name=?,pwd=? where id=" + id;
        Object[] objects = new Object[2];
        objects[0] = "lian";
        objects[1] = "1q2w3e";
        jdbcTemplate.update(sql, objects);
        return "redirect:/users";
    }
}
```

格式，先从Spring容器中获得JdbcTemplate对象，再用String写好sql语句，就可以:

用`jdbcTemplate.query(sql)`执行查询语句

用`jdbcTemplate.update(sql)`执行增、删、改语句，

还有一些代替方法，如：

`jdbcTemplate.queryForList(sql)`：查询结果类型为List<Map<String, Object>>

`jdbcTemplate.queryForMap(sql)`：查询结果类型为Map<String, Object>

`jdbcTemplate.execute(sql)`：执行所有sql语句



注：

1、事务自动commit

2、传参方法，可拼接sql的字符串（有sql注入风险），推荐在参数位置用`?`代替，参数在执行jdbcTemplate方法时以参数的形式传进去





#### 7.7.2 更换数据源（Druid）

以Druid为例

Druid是阿里巴巴开源平台的一个数据库连接池实现，结合了C3P0，DBCP，PROXOOL等数据库池的优先，并且自带日志监控

##### 1、导入依赖

```xml
<!-- https://mvnrepository.com/artifact/com.alibaba/druid -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.2.8</version>
</dependency>
<!-- 使用log4j日志功能时需要导入 -->
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.17</version>
</dependency>
```

或用starter

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.2.8</version>
</dependency>
```



##### 2、修改配置文件

```yaml
spring:
  datasource:
    username: root
    password: '001118'
    url: jdbc:mysql://localhost:3306/mybatis?useSSL=true&useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC
    driver-class-name: com.mysql.cj.jdbc.Driver
    type: com.alibaba.druid.pool.DruidDataSource
    
    #SpringBoot默认是不注入这些的，需要自己绑定
    #druid数据源专有配置
    initialSize: 5
    minIdle: 5
    maxActive: 20
    maxWait: 60000
    timeBetweenEvictionRunsMillis: 60000
    minEvictableIdleTimeMillis: 300000
    validationQuery: SELECT 1 FROM DUAL
    testWhileIdle: true
    testOnBorrow: false
    testOnReturn: false
    poolPreparedStatements: true

    #配置监控统计拦截的filters，stat：监控统计、log4j：日志记录、wall：防御sql注入
    #如果允许报错，java.lang.ClassNotFoundException: org.apache.Log4j.Properity
    #则导入log4j 依赖就行
    filters: stat,wall,log4j
    maxPoolPreparedStatementPerConnectionSize: 20
    useGlobalDataSourceStat: true
    connectionProperties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=500
```

加一个type属性，指定为对应数据源的DataSource，之后加上各种个性化配置



##### 3、自定义监控平台

在java下的config目录下新建一个专门的配置类，用`@Configuration`标注

创建并初始化一个`ServletRegistrationBean`类的bean可以实现监控平台；

创建并初始化一个`FilterRegistrationBean`类的bean可以实现对监控平台监控内容的过滤；

```java
@Configuration
public class DruidConfig {

    @ConfigurationProperties(prefix = "spring.datasource")
    @Bean
    public DataSource druidDataSource(){
        return new DruidDataSource();
    }

    // 后台监控功能
    @Bean
    public ServletRegistrationBean statViewServlet(){
        ServletRegistrationBean<StatViewServlet> bean = new ServletRegistrationBean<>(new StatViewServlet(),"/druid/*");

        // 增加设置
        HashMap<String, String> initParameters = new HashMap<>();

        // 监控页面登录
        initParameters.put("loginUsername", "admin");  
        initParameters.put("loginPassword", "123456");

        // 允许谁可以访问(不写是全都允许)
        initParameters.put("allow", "");

        // 禁止谁的访问
        initParameters.put("name", "192.168.11.123");

        // 设置初始化参数
        bean.setInitParameters(initParameters);
        return bean;
    }

    // 监控过滤器
    @Bean
    public FilterRegistrationBean webStatFilter(){
        FilterRegistrationBean bean = new FilterRegistrationBean();
        bean.setFilter(new WebStatFilter());
        
        // 增加设置
        Map<String, String> initParameters = new HashMap<>();
        
        // 设置不统计的内容
        initParameters.put("exclusions","*.js, *.css, /druid/*");
        bean.setInitParameters(initParameters);
        return bean;
    }
}

```

启动后，可通过设置的url的上一级访问监控平台登录页面（设置/druid/*的话，上一级就是/druid，后面的一级是监控平台的不同页面）

<img src="C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211220153203050.png" alt="image-20211220153203050" style="zoom:67%;" />

登录后可查看各种监控信息

![image-20211220153232218](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211220153232218.png)



![image-20211220153333390](C:\Users\Salieri\AppData\Roaming\Typora\typora-user-images\image-20211220153333390.png)



也可以用配置文件自定义监控平台

```properties
#配置监控统计拦截的filters，去掉后监控界面sql无法统计，'wall'用于防火墙
spring.datasource.druid.filters= stat,wall


#######监控配置
# WebStatFilter配置，说明请参考Druid Wiki，配置_配置WebStatFilter
spring.datasource.druid.web-stat-filter.enabled=true
spring.datasource.druid.web-stat-filter.url-pattern=/*
spring.datasource.druid.web-stat-filter.exclusions=/druid/*,*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico
spring.datasource.druid.web-stat-filter.session-stat-enable=true
spring.datasource.druid.web-stat-filter.session-stat-max-count=10
spring.datasource.druid.web-stat-filter.principal-session-name=session_name
spring.datasource.druid.web-stat-filter.principal-cookie-name=cookie_name
spring.datasource.druid.web-stat-filter.profile-enable=
# StatViewServlet配置，说明请参考Druid Wiki，配置_StatViewServlet配置默认false
spring.datasource.druid.stat-view-servlet.enabled=true
# 配置DruidStatViewServlet
spring.datasource.druid.stat-view-servlet.url-pattern=/druid/*
#  禁用HTML页面上的“Reset All”功能
spring.datasource.druid.stat-view-servlet.reset-enable=false
spring.datasource.druid.stat-view-servlet.login-username=admin #监控页面登录的用户名
spring.datasource.druid.stat-view-servlet.login-password=123456 #监控页面登录的密码
#IP白名单(没有配置或者为空，则允许所有访问)
spring.datasource.druid.stat-view-servlet.allow=127.0.0.1,192.168.0.119
#IP黑名单 (存在共同时，deny优先于allow)
spring.datasource.druid.stat-view-servlet.deny=
#Spring监控配置，说明请参考Druid Github Wiki，配置_Druid和Spring关联监控配置
spring.datasource.druid.aop-patterns= com.ghy.service.*

```

yml同理



#### 7.7.3 整合MyBatis

引入MyBatis，实现MVC项目的Model层（数据库+dao+service）和Controller层

##### 1、导入依赖

数据库的jdbc、mysql驱动、mybatis的starter：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>

<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.0</version>
</dependency>
```

spring-boot-starter-XX是Spring官方的，XX-spring-boot-starter是第三方的

##### 2、在项目配置文件进行配置

```yaml
spring:
  datasource:
    username: root
    password: '001118'
    url: jdbc:mysql://localhost:3306/mybatis?useSSL=true&useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC
    driver-class-name: com.mysql.cj.jdbc.Driver

# 整合MyBatis
mybatis:
  type-aliases-package: com.ghy.pojo                   # 别名
  mapper-locations: classpath:mybatis/mapper/*.xml     # 扫描mapper
  #开启驼峰命名
  configuration:
    map-underscore-to-camel-case: true
```

##### 3、编写实体类

com.ghy.pojo.User.java，与数据库的表对应

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class User implements Serializable {
    private int id;
    private String name;
    private String password;
}
```

##### 4、dao层的mapper接口与xml实现

**接口**：com.ghy.mapper.UserMapper.java

```java
@Mapper
@Repository
public interface UserMapper {
    List<User> queryUserList();
    User queryUserById(int id);
    int addUser(User user);
    int updateUser(User user);
    int deleteUser(int id);
}
```

`@Mapper`注解表明此类是mapper和对应的xml文件匹配上；

`@Repository`注解表明此接口是Spring的dao层组件，将该接口对象注入Spring容器



**xml实现**：resources/mybatis/mapper/UserMapper.xml

​		<u>用resultMap解决了数据库表的列名和实体类对应属性名不一致的问题</u>

​		使用传入对象的属性时，如果数据库表的列名和实体类对应属性名不一致，要使用实体类中的名字

```xml
<?xml version="1.0" encoding="UTF8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<!--namespace绑定一个对应的Dao/Mapper接口-->
<mapper namespace="com.ghy.mapper.UserMapper">
<resultMap id="UserMap" type="user">
    <result column="pwd" property="password"/>
</resultMap>

    <select id="queryUserList" resultMap="UserMap">
        select * from mybatis.user
    </select>

    <select id="queryUserById" resultMap="UserMap">
        select * from mybatis.user where id=#{id}
    </select>

    <insert id="addUser" parameterType="user">

        insert into mybatis.user(id, name, pwd) values (#{id}, #{name}, #{password})
    </insert>

    <update id="updateUser" parameterType="user">
        update mybatis.user set name=#{name},pwd=#{password} where id=#{id}
    </update>

    <delete id="deleteUser" parameterType="int">
        delete from mybatis.user where id=#{id}
    </delete>

</mapper>

```



##### 5、service层的service接口与实现类

接口：com.ghy.service.UserService

​		和dao层接口对应

```java
public interface UserService {
    List<User> queryUserList();
    User queryUserById(int id);
    int addUser(User user);
    int updateUser(User user);
    int deleteUser(int id);
}
```

实现类：com.ghy.service.UserServiceImpl

从Spring容器中获取dao接口，调用dao接口实现操作

```java
@Service
public class UserServiceImpl implements UserService{

    @Autowired
    private UserMapper userMapper;

    @Override
    public List<User> queryUserList() {
        List<User> userList = userMapper.queryUserList();
        return userList;
    }

    @Override
    public User queryUserById(int id) {
        User user = userMapper.queryUserById(id);
        return user;
    }

    @Override
    public int addUser(User user) {
        return userMapper.addUser(user);
    }

    @Override
    public int updateUser(User user) {
        return userMapper.updateUser(user);
    }

    @Override
    public int deleteUser(int id) {
        return userMapper.deleteUser(id);
    }
}
```

`@Service`注解表明此接口是Spring的service层组件，将该接口对象注入Spring容器，从容器获取service接口时，因为实现类是接口的子类，所以会自动注入实现类



##### 6、Controller层的controller类

com.ghy.controller.UserController.java

接收View层url请求，调用Model层的service层实现业务。

```java
@Controller
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/users")
    @ResponseBody
    public List<User> queryUserList(){
        List<User> userList = userService.queryUserList();
        return userList;
    }
    @GetMapping("/user")
    @ResponseBody
    public User queryUser(){
        User user = userService.queryUserById(5);
        return user;
    }
    @GetMapping("/add")
    public String addUser(){
        userService.addUser(new User(5,"希斯琳","12loj3"));
        return "redirect:/users";
    }
    @GetMapping("/update")
    public String updateUser(){
        userService.updateUser(new User(2,"西琳","honkai3"));
        return "redirect:/users";
    }
    @GetMapping("/delete")
    public String deleteUser(){
        userService.deleteUser(5);
        return "redirect:/users";
    }
}
```

@Controller注解表明此类是Spring的controller层组件，将此类对象注入Spring容器



##### 7、理解（重点）

流程：

- SpringBoot的web项目启动后，前端会自动打开初始页面，或用户自己输入地址跳到对应的页面，页面中有各种链接；
- 用户点击链接，就产生了url请求，SpringBoot自动使用Spring容器中的Controller对象进行匹配；
- 执行匹配到的Controller对象的对应方法，该方法调用容器中的Service接口（实际是接口的实现类）的对应方法；
- 执行Service接口的对应方法，该方法调用dao接口的对应方法;
- SpringBoot根据配置文件的设置扫描mapper，找到对应的mapper.xml;
- 根据mapper.xml的具体sql语句实现数据库的操作，返回结果；
- 结果一层层返回，返回到Controller层的Controller对象；
- Controller对象对数据进行操作后（如数据判断、将数据转入model类给前端等），用转发/重定向方式改变View层视图



看上去Service层只是承上启下，没有别的用，实际上Service层的实现类可以增加事务控制的功能

Service层接口类非必需，但推荐有，因为一个Service层接口可以用不同的Service实现类实现，更加灵活

（如果没有接口，切换不同的Service实现类要修改Controller对象中的变量，有接口的话，Controller对象中的属性设为Service接口即可，Spring容器实际注入了哪个实现类，就会自动注入进去）























## 附录

### 1、IDEA快捷键

Alt：多行操作

Ctrl+Alt+V / Alt+Enter：自动补齐返回值类型

Ctrl+O：覆写方法

Ctrl+I：实现接口中的方法

Ctrl+Shift+U：大小写转换

Ctrl+Shift+Z：取消撤销

Ctrl+B：进入源码

Alt＋Insert：生成构造方法、getter、setter

Ctrl+Y：删除当前行

Ctrl+Shift+J：将选中的行合并成一行

Ctrl+G：定位到某一行

Ctrl+Shift+向上箭头：将光标所在的代码块向上整体移动

Ctrl+Shift+向下箭头：将光标所在的代码块向下整体移动

Alt+Shift+向上箭头：将行向上移动

Alt+Shift+向下箭头：将行向下移动

shift+tab代码左移

Ctrl+F：查找

Ctrl+R：替换

Ctrl+Shift+F：全局查找

Ctrl+Shift+R：全局替换

Ctrl+Shift+N：打开指定文件

Ctrl+Shift+Enter：自动补齐 {} 或者 ;

Shift+Enter：在当前行的下方开始新行

Ctrl+Alt+Enter：在当前行的上方插入新行

Ctrl+Del：删除光标所在至单词结尾处的所有字符

Ctrl+D：复制本行到下一行

psvm：生成main()方法；

按住Alt下划多行：同时修改多行



### 2、Java语法

`Character.isLetter(ch)`：判断ch是不是字母

`Character.toLowerCase(ch)`：字母转小写

`Character.toUpperCase(ch)`：字母转大写

