# Mybatis
reference link: [Mybatis.org](https://mybatis.org/mybatis-3) 

## 1. Overview
- Mybatis is a first class persistence framework. avoids JDBC codes, supports SQL query and advaned ORM.
- - Persist your Java program data on DB
- Simple configuration mapping your Java Classes to your DB Tables
- save your time for writing painful JDBC code

### 1.1 Project Structure
- One Table --- One POJO Class --- One Mybatis Mapper
    - Each mapper contains all the query methods related to that class
- src 
    - pojo
        - User.java (Pojo class)
    - dao
        - UserMapper.java (Interface)
        - UserMapper.xml (mapper config)
- resources
    - mybatis-config.xml (global config)
    - db.properties    (property files)
    - dao
        - UserMapper.xml (mapper config)

- .xml config file can be placed in either src/ or resources/
	○ if placed in src/, then you need the following resource filter to include those .xml files

```xml
<build>
    <resources>
        <resource>
            <directory>src/main/java</directory>
            <includes>
                <include>**/*.xml</include>
            </includes>
        </resource>
        <resource>
            <directory>src/main/resources</directory>
            <includes>
                <include>**/*.xml</include>
                <include>**/*.properties</include>
            </includes>
        </resource>
    </resources>
<build>
```

### 1.2 Maven
```xml
<dependency>   
    <groupId>org.mybatis</groupId>         
    <artifactId>mybatis</artifactId>         
    <version>3.4.6</version> 
</dependency>
```

## 2. Lifecycle
### 2.1 SqlSessionFactoryBuilder
- build a session factory from a given config file
- after the factory constructed, the builder can be discarded. 
```java
String resource = "org/mybatis/example/mybatis-config.xml"; 
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory =  new SqlSessionFactoryBuilder().build(inputStream);
```

### 2.2 SqlSessionFactory
- Once created, the SqlSessionFactory should exist for the duration of your application execution.
- Use a Singleton pattern 
- Factory has a pool of threads for db connections

```java
// safe closing the session
try (SqlSession session = sqlSessionFactory.openSession()) {  
    BlogMapper mapper = session.getMapper(BlogMapper.class);  
    Blog blog = mapper.selectBlog(101); 
} catch (Exception e) {
    ...
} 
```

### 2.3 SqlSession
- Each thread has its own SqlSession. SqlSession is not thread safe.
- Do not keep SqlSession as a Class field or Static field 
- Upon receiving an HTTP request, you can open a SqlSession, then upon returning the response, you can close i
- Always close the SqlSession for recycling resources

### 2.4 Mapper Instances
- One SqlSession can create multiple Mapper Instances
- Mapper Instance ismdesigned to call the compiled SQL methods


## 3. Configuration
### 3.1 Global XML
#### 3.1.1 Environment
```xml
<environments default="development">
    <environment id="development">
        <transactionManager type="JDBC">
            <property name="..." value="..."/>     
        </transactionManager>     
        <dataSource type="POOLED">       
            <property name="driver" value="${driver}"/>       
            <property name="url" value="${url}"/>       
            <property name="username" value="${username}"/>  
            <property name="password" value="${password}"/>   
        </dataSource>   
    </environment> 
</environments>
```

- environment
	- each env has a transactionManager and a dataSource
- transactionManager
	- **JDBC**: with versioning and rollback
	- **MANAGED**: does nothing. no commits and rollback
- dataSource
	- **UNPOOLED**
	- **POOLED**: with thread pool
	- **JNDI**: TODO

#### 3.1.2 Properties
- load in `.properties` files and use them as variables
- file properties has the highest priority
```xml
<properties resource="org/mybatis/example/config.properties">
    <property name="username" value="dev_user"/>
    <property name="password" value="F2Fa3!33TYyg"/>
</properties>
```

#### 3.1.3 Mapper
- register all *Mapper.xml by xml, class, package

```xml
<mappers>
    // file path
    <mapper resource="com/sijie/dao/UserDao.xml">
    // class
    <mapper class="com.sijie.dao.UserDao">
    // scan package
    <package name="com.sijie.dao">
</mappers>
```

#### 3.1.4 TypeAliases
```xml
<typeAliases>   
    <typeAlias alias="User" type="com.sijie.pojo.User"/>
    // Mybatis will search from this package
    <package name="com.sijie.pojo">
</typeAliases>
```

```java
@Alias("User")
public Class User {
    String name;
    ...
}
```

- with type alias, you can type `User` instead of `com.sijie.pojo.User`

## 4. Logging
- settting.logImpl 

```xml
<settings>
    <setting name="logImpl" value="LOG4J">
<settings>
```

- logImpl
	- LOG4J
	- STDOUT_LOGGING

### 4.1 Log4j
#### 4.1.1 Configuration
- can specify logging level for specific Package / Class / Method. 
	- log4j.logger.<path to your target> = [TRACE | DEBUG | INFO | ERROR | ...]

```properties
# Root logger
log4j.rootLogger=DEBUG, console

# mybatis logging config
log4j.logger.com.sijie.dao=TRACE

# console output
log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.Target=System.out
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.consoleAppender.layout.ConversionPattern=[%t] %-5p %c %x - %m%n
```


#### 4.1.2 Level
- `ALL < DEBUG < INFO < WARN < ERROR < FATAL < OFF`

## 5. Mapping
### 5.1 XML
- Parameter Map

```xml
<select id="getUserById" paramerterType="map" resultType="com.sijie.pojo.User">
    select * from user where id = #{id} and name=#{username}
</select>
```

```java
Map<String, Object> map = new HashMap<String, Object>();
map.put("id", 123);
map.put("username", "Jason");
mapper.getUserById(map);
```

- Result Map

```xml
<!--column: DB, property: Class-->
<resultMap id="UserMap" type="com.sijie.pojo.User">
    <result column="name" property="username">
    <result column="pwd" property="password">
</resultMap>

<select id="getUserById" paramerterType="map" resultType="UserMap">
    select * from user where id = #{id} and name=#{username}
</select>
```

- `#{}` is better than `${}`, because of SQL Injection Vulnerability

### 5.2 Annotation
- CRUD implemented by Annotation Mapping

```java
public interface UserMapper{
    @Select("select * from user")
    List<User> getAllUsers();
    
    @Select("select * from user where id = #{uid}")
    User getUserById(@Param("uid") int id)；
    
    @Insert("insert into user(id, name, pwd) values (#{id}, #{name}, #{password})")
    int addUser(User user)
    
    @Update("update user set name=#{name},pwd=#{password} where id=#{id}")
    int updateUser(User user)
    
    @Delete("delete from user where id=#{uid}")
    int deleteUser(@Param("uid") int id);
}
```

## 6. Lombok
- reference link: Lombok Introduction (Chinese)
- Java code automation. Generate Trivial Java code such as Getter, Setter, Constructor, toString, ...
- Maven dependency & Intellij plugin
	- Intellij plugin is to avoid the error detection from Intellij
	- Lombok jar packages generate code during the compile time

### 6.1 Getter & Setter
- can annotate Class or Field.
- by default the generated method is PUBLIC.

```java
// PROTECTED Scope
@Getter(AccessLevel.PROTECTED) @Setter private Integer id;
@Getter @Setter private String name;
```

### 6.2 toString
TODO


## 7. Multiple Table Query
- apply <resultMap> to map the complex result set

```java
// example pojo Class
@Data
public class Course {
    private int id;
    private String name;
    
    private List<Student> students;
}

@Data
public class Student {
    private int id;
    private String name;
}

@Data
public class Enroll {
private int cid;
    private int sid;

    private Course course;
    private Student student;
}
```


```sql
// example DB Table schema
CREATE TABLE student (
   id INT PRIMARY KEY,
   name VARCHAR(30)
);

CREATE TABLE course (
   id INT PRIMARY KEY,
   name VARCHAR(30)
);

CREATE TABLE enroll (
   cid INT,
   sid INT,
PRIMARY KEY (cid, sid),
FOREIGN KEY (cid) REFERENCES course(id),
FOREIGN KEY (sid) REFERENCES student(id)
);
```

- `<association>`
	- one flat record: associate an object (Course) to the main object (Student)
	- each record represents one enrollment of one student takes one course

```xml
<select id="getAllEnrollments" resultMap="ExtendEnroll">
    select s.id sid, s.name sname, c.id cid, c.name cname
    from enroll E, course C, student S
    where E.cid = C.id and E.sid = S.id
</select>

<resultMap id="ExtendEnroll" type="com.sijie.pojo.Enroll">
    <result property="cid" column="cid" />
    <result property="sid" column="sid" />
    <association property="student" javaType="com.sijie.pojo.Student">
        <result property="id" column="sid" />
        <result property="name" column="sname" />
    </association>
    <association property="course" javaType="com.sijie.pojo.Course">
        <result property="id" column="cid" />
        <result property="name" column="cname" />
    </association>
</resultMap>
```

- `<collection>`
	- aggregated record: a main object (Course) is associated with mutiple other objects (Student) who are taking that course. 
	- each record has an array integrated inside. 
- the example shares the same Class Model and DB Schema
	- it returns all the enrollments of all courses with aggragated Students inside Course

```xml
<!--same class model and DB Schema-->
<select id="getAllCourses" resultMap="CourseStudent">
    select C.id cid, C.name cname, S.id sid, S.name sname
    from course as C, student as S, enroll as e
    where C.id = E.cid and S.id = E.sid 
</select>

<resultMap id="CourseStudent" type="com.sijie.pojo.Course">
    <result property="id" column="cid"/>
    <result property="name" column="cname"/>
    <collection property="students" ofType="com.sijie.pojo.Student">
        <result property="id" column="sid"/>
        <result property="name" column="sname"/>
    </collection>
</resultMap>
```

## 8. Cache
- cache DB data in memory, save time for I/O.
- First level Caching (local session caching)
	- cache data for the duration of a session, does not share between different sessions
- Second level Caching

### 8.1 First Level Caching
- Enabled by default
- All select results are cached
- All insert, update, delete would flush the cache
- LRU eviction policy
- Session local cache, not shared with other sessions 

```java
UserMapper mapper = sqlSession.getMapper(UserMapper.class);
User u1 = mapper.getUserById(1);
User u2 = mapper.getUserById(2);

System.out.println(u1 == u2);
// output TRUE, because the select result is cached 
sqlSession.clearCache();
User u3 = mapper.getUserById(3);
System.out.println(u1 == u3)
// output FALSE
```

### 8.2 Second level Caching
- Global Scope Caching, shared by all sessions. 
- Cache unit is the Namespace. 
    - each namespace has a second level cache space.
- Eable Second level caching by adding <cache> tag in *Mapper.xml

```xml
<!-- in mybatis-config.xml -->
<settings>
    ...
    <setting name="cahceEnabled" value="true">
</settings>

<!-- in *Mapper.xml -->
<cache 
    eviction="FIFO"   
    flushInterval="60000"  
    size="512"   
    readOnly="true"/>
<!-- or use the default second level cache config -->
<cache/>
```

#### 8.2.1 Implementation
- Each session has a local cache space. 
- Each namespace has a second level global cache space
- When the session closes or commits, first cache --> second cache
- The new session will reads data from the second cache first. 

### 8.3 Custom Cache
- we can use 3rd party cache implementation
- The 3rd party cache class must implements the org.apache.ibatis.cache.Cache interface

```java
public interface Cache {   
    String getId();   
    int getSize();   
    void putObject(Object key, Object value);   
    Object getObject(Object key);   
    boolean hasKey(Object key);   
    Object removeObject(Object key);   
    void clear(); 
}
```

- To configure your cache, simply add public JavaBeans properties to your Cache implementation, and pass properties via the cache Element, for example, the following would call a method called ... on your Cache implementation:
setCacheFile(String file)

```xml
<cache type="com.domain.something.MyCustomCache">   
    <property name="cacheFile" value="/tmp/my-custom-cache.tmp"/>
</cache>
```


