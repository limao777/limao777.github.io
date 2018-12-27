---
title: hibernate介绍与快速入门
date: 2017-10-30 10:11:35
category: java
tags:
- web
- 数据库
---

# 应用场景
一个数据库框架，使用hibernate后可以使用hql语句操作数据库，如果涉及到数据库迁移等则无需改代码，使开发人员更关注业务本身

# 快速入门
## hibernate和structs说明
hibernate可以用在j2se和j2ee中，而struts是web框架，一般用于j2ee项目
## 开发流程
### 创建一个项目

### 引入hibernate开发包
从网上下载

### 开发的三种方式
* 由Domain object -> mapping -> db（官方推荐）
* 由DB开始，用工具生成mapping和Domain object（使用较多）
* 由映射文件开始

### 使用第二种方式开发项目
* 创建表和序列(使用oracle)
```text
create table employee(
id number primary key,
name varchar2(64) not null,
email varchar(64) not null,
hiredate date not null
);
create sequence emp_seq(
start with 1,
increment by 1,
min value 1,
nomaxvalue,
nocycle,
nocache,
);
```
* 开发domain对象和对象映射文件，用于指定domain对象和表的映射关系，该文件取名有规范：domain对象.hbm.xml，一般放在和domain对象同一个文件夹下
```java
//
package com.lm.domain
import java.io.Serializable;

//domain对象那个名称就是对应首字母的大写
//domain对象，也叫javabean或pojo(plain ordinary java objects)
public class Employee implements Serializable{
	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;
	private Integer id;
	private String name;
	private String email;
	private java.util.Date hiredate;
	public Integer getId() {
		return id;
	}
	public void setId(Integer id) {
		this.id = id;
	}
	public String getName() {
		return name;
	}
	public void setName(String name) {
		this.name = name;
	}
	public String getEmail() {
		return email;
	}
	public void setEmail(String email) {
		this.email = email;
	}
	public java.util.Date getHiredate() {
		return hiredate;
	}
	public void setHiredate(java.util.Date hiredate) {
		this.hiredate = hiredate;
	}
}
```

```xml
<?xml version="1.0" encoding="utf-8"?>
<!--该文件要清楚地表述出 类 和 表 的对应关系-->
<!DOCTYPE hibernate-mapping PUBLIC
	"-//Hibernate/Hibernate Mapping DTD 3.0//EN"
	"http://hibernate.sourceforge.net/hibernate-mapping-3.0.dtd">
<!-- package : 表示该类在哪个包下 -->
<hibernate-mapping package="com.lm.domain">
<!-- name : 表示类名 table 表示 该类和哪个表映射 -->
	<class name="Employee" table="employee">
		<!-- id元素专门用于指定主键是如何生成,hibernate设计者认为，我们每一个表都应该有一个主键 -->
		<!-- name:表示类的哪个属性是主键 -->
		<id name="id" type="java.lang.Integer">
		<!-- 指定主键生成策略 -->
		<generator class="sequence">
		<param name="sequence">emp_seq</param>
		</generator>
		</id>
		<property name="name" type="java.lang.String">
		<column name="name" not-null="true"/>
		</property>
		<property name="email" type="java.lang.String">
		<column name="email" not-null="true"/>
		</property>
		<property name="hiredate" type="java.util.Date">
		<column name="hiredate" not-null="true"/>
		</property>
	</class>
</hibernate-mapping>
```

* 手动配置hibernate.cfg.xml文件（文件名一般不用改，但是也可以改成其他名字），该文件用于配置连接数据库的类型,driver、用户名、密码等，位于src根目录
```xml
<?xml version="1.0" encoding="utf-8"?>
<!DOCTYPE hibernate-configuration PUBLIC
	"-//Hibernate/Hibernate Configuration DTD 3.0//EN"
	"http://hibernate.sourceforge.net/hibernate-configuration-3.0.dtd">
<!-- 该文件用于配置连接数据的种类,用户名，密码,ul ,驱动.. 连接池,二级缓存.. 有点类似strus  struts-config.xml -->
<hibernate-configuration>
	<session-factory>
		<property name="connection.driver_class">oracle.jdbc.driver.OracleDriver</property>
		<property name="connection.url">jdbc:oracle:thin:@127.0.0.1:1521:orclhsp</property>
		<property name="connection.username">scott</property>
		<property name="connection.password">tiger</property>
		<!-- 配置显示hibernate生成的 sql ,特别说明，在开发阶段设为true利于调试，在使用项目则设为false-->
		<property name="show_sql">true</property>
		<!-- 配置数据库的方言/ -->
		<property name="dialect">org.hibernate.dialect.OracleDialect</property>
		<!-- 配置管理的对象映射文件 -->
		<mapping resource="com/hsp/domain/Employee.hbm.xml"/>
	</session-factory>
</hibernate-configuration>
```

* 测试文件
```java
package com.lm.view;

import java.sql.Date;

import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.Transaction;
import org.hibernate.cfg.Configuration;

import com.lm.domain.Employee;
import com.lm.utils.MySessionFactory;

public class TestMain {

	/**
	 * @param args
	 */
	public static void main(String[] args) {
		
		// TODO Auto-generated method stub
		SessionFactory sessionFactory= MySessionFactory.getSessionFactory();
		Session session=sessionFactory.openSession();
		//查询可以不使用事务
		Employee emp=(Employee) session.load(Employee.class, 3);
		System.out.println(emp.getName()+" "+emp.getEmail());
		session.close();
	}

	
	
	public static void delEmp() {
		// TODO Auto-generated method stub
		SessionFactory sessionFactory= MySessionFactory.getSessionFactory();
		Session session=sessionFactory.openSession();
		//删除一个雇员,先得到，再修改
		Transaction ts=session.beginTransaction();
		
		Employee emp=(Employee) session.load(Employee.class, 2);
		session.delete(emp);
		ts.commit();
	}

	//更新
	public static void updateEmp() {
		SessionFactory sessionFactory= MySessionFactory.getSessionFactory();
		Session session=sessionFactory.openSession();
		//修改一个雇员,先得到，再修改
		Transaction ts=session.beginTransaction();
		//load方法是用于获取 指定 主键的对象（记录.）
		Employee emp=(Employee)session.load(Employee.class, 1);
		emp.setName("小名");
		ts.commit();
	}
	
	//修改雇员
	

	//添加一个雇员
	private static void addEmpoyee() {
		//1.得到Configuration
		Configuration configuration= new Configuration().configure();
		//2.得到SessionFactory(会话工厂，这是一个重量级的类，因此要保证在一个应用程序中只能有一个)
		SessionFactory sessionFactory=configuration.buildSessionFactory();
		
		//3. 从SessionFactory中取出一个Session对象(它表示 和数据库的出一次会话)
		Session session=sessionFactory.openSession();
		//4. 开始一个事务
		Transaction transaction = session.beginTransaction();
		//保存一个对象到数据库(持久化一个对象)
		Employee emp=new Employee();
		emp.setEmail("kk@sohu.com");
		emp.setHiredate(new java.util.Date());
		emp.setName("shunping");
		//不要给id,因为它是自增的
		session.save(emp);//insert into employee (name,id,...) value(?,?,?)
		transaction.commit();
	}

}

```
