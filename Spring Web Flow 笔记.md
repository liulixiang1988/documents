# Spring Web Flow 笔记

刘理想 2016-12-28

[TOC]

## 1. Introduction

Spring MVC 是个强大的框架，我们可以用它来配置和管理任意的应用流程。然而，有时我们需要多流程有更强的控制。

**Spring Web-Flow**通过定义明确的View和Transition来实现上述需求。它基于`Spring MVC`构建。下面我们来看看如何在应用中使用Web-Flow。

## 2. Project Set-Up

创建一个Maven应用。`pom.xml`如下所示：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.jcg.examples.springWebFlowExample</groupId>
  <artifactId>SpringWebFlowExample</artifactId>
  <packaging>war</packaging>
  <version>0.0.1-SNAPSHOT</version>
  <name>SpringWebFlowExample</name>
  <url>http://maven.apache.org</url>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>
     <dependency>
        <groupId>org.springframework.webflow</groupId>
        <artifactId>spring-webflow</artifactId>
        <version>2.4.2.RELEASE</version>
    </dependency>
  </dependencies>
  <build>
    <finalName>SpringWebFlowExample</finalName>
  </build>
</project>
```

这里我们的例子是一个简单的登录应用。第一次输入URL时首先跳转到登录页。

用户输入用户名、密码，然后点击登录按钮。

如果密码正确，视图跳转到成功视图，否则跳回到登录视图。

虽然这个非常简单场景是为了新手便于理解准备的，但是对于非常复杂的场景，Web-Flow也能处理。

## 3. Implementation

首先我们需要定义一个Pojo来存储登录需要用到的用户名和密码。

*LoginBean.java*

```java
package com.jcg.examples.bean;

import java.io.Serializable;

public class LoginBean implements Serializable
{
		/**
		 * 
		 */
		private static final long serialVersionUID = 1L;

		private String userName;

		private String password;

		public String getUserName()
		{
				return userName;
		}

		public void setUserName(String userName)
		{
				this.userName = userName;
		}

		public String getPassword()
		{
				return password;
		}

		public void setPassword(String password)
		{
				this.password = password;
		}

		@Override
		public String toString()
		{
				return "LoginBean [userName=" + userName + ", password=" + password + "]";
		}

}
```

`Service`文件用来认证用户。web-flow基于`validateUser`方法的输出，来决定渲染哪个视图。Service使用了注解，这样Spring Bean Factory就能在运行时识别它。

*LoginService.java*

```java
package com.jcg.examples.service;

import org.springframework.stereotype.Service;

import com.jcg.examples.bean.LoginBean;

@Service
public class LoginService
{
		public String validateUser(LoginBean loginBean)
		{
				String userName = loginBean.getUserName();
				String password = loginBean.getPassword();
				if(userName.equals("Chandan") && password.equals("TestPassword"))
				{
						return "true";
				}
				else
				{
						return "false";
				}
		}
		
}
```

现在我们需要定义flow。Flow是一系列为了完成某个任务的事件闭环。这个闭环包含了一系列事件，用户在这些视图间前后跳转。我们来看一下实现应用的xml：

*book-search-flow.xml*

```xml
<?xml version="1.0" encoding="UTF-8"?>
<flow xmlns="http://www.springframework.org/schema/webflow"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/webflow
http://www.springframework.org/schema/webflow/spring-webflow-2.4.xsd">

	<var name="loginBean" class="com.jcg.examples.bean.LoginBean" />
	
	<view-state id="displayLoginView" view="jsp/login.jsp" model="loginBean">
		<transition on="performLogin" to="performLoginAction" />
	</view-state>

	<action-state id="performLoginAction">
		<evaluate expression="loginService.validateUser(loginBean)" />

		<transition on="true" to="displaySuccess" />
		<transition on="false" to="displayError" />

	</action-state>
	
	<view-state id="displaySuccess" view="jsp/success.jsp" model="loginBean"/>

	<view-state id="displayError" view="jsp/failure.jsp" />
</flow>
```

Flow中的第一个视图是默认视图，因此，当输入视图的URL时，首先显示这个视图。当用户提交后，视图转移到`action`标签，用于决定下面该渲染什么视图。`action directive`（动作指令）反过来使用我们之前创建的Service Bean。

同事，视图页会有backing Bean，backing Bean定义在`model`属性上。View包含两个重要的变量用来告诉容器，已经发生的事件和目前应用的状态。这两个变量时`_eventId`和`_flowExecutionKey`。当为视图编写代码时，程序员不要忘了在视图代码中包含这些变量。

现在，flow已经有了，我们需要把它挂载到系统中去，以便于Spring容器能够发现它。

`flow-definition.xml`文件定义了`Flow-Executor`和`Flow-Registry`。 正如名字所表示的那样，`Flow-Executor`用来串接flow，同事它引用`Flow-Registry`用来决定流程的下一个动作。

`FlowHandlerMapping`负责为应用中所有的flow创建URL。

`FlowHandlerAdapter`封装实际流，并委托由Spring Flow Controller处理的特定流。我们将在spring configuration sheet中包含这个文件，这样，我们的web-flow就挂载到了Spring Container中。请求就被转给Flow Controller了。

*flow-definition.xml*

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:flow="http://www.springframework.org/schema/webflow-config"
	xsi:schemaLocation="http://www.springframework.org/schema/webflow-config
http://www.springframework.org/schema/webflow-config/spring-webflow-config-2.4.xsd
http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">

	<bean class="org.springframework.webflow.mvc.servlet.FlowHandlerMapping">
		<property name="flowRegistry" ref="bookSearchFlowRegistry" />
	</bean>

	<bean class="org.springframework.webflow.mvc.servlet.FlowHandlerAdapter">
		<property name="flowExecutor" ref="bookSearchFlowExecutor" />
	</bean>

	<flow:flow-executor id="bookSearchFlowExecutor" flow-registry="bookSearchFlowRegistry" />

	<flow:flow-registry id="bookSearchFlowRegistry">
		<flow:flow-location id="bookSearchFlow" path="/flows/book-search-flow.xml" />
	</flow:flow-registry>

</beans>
```

`flow-config.xml`为spring容器包含了一些基本信息，比如试图渲染、bean声明、注解扫描等。它还包含`flow-definition.xml`，用于加载其内容。

*spring-config.xml*

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:context="http://www.springframework.org/schema/context"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:flow="http://www.springframework.org/schema/webflow-config"
	xsi:schemaLocation="
	http://www.springframework.org/schema/webflow-config
	http://www.springframework.org/schema/webflow-config/spring-webflow-config-2.4.xsd
   http://www.springframework.org/schema/beans
   http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
   http://www.springframework.org/schema/context
   http://www.springframework.org/schema/context/spring-context-3.0.xsd">

	<context:component-scan base-package="com.jcg.examples" />

	<bean
		class="org.springframework.web.servlet.view.InternalResourceViewResolver">
		<property name="prefix" value="/jsp/" />
		<property name="suffix" value=".jsp" />
	</bean>
	
	<import resource="flow-definition.xml"/>
	
</beans>
```

`web.xml`跟其他spring mvc应用很像。它使用上面的xml，并且把所有的请求转给`DispatcherServlet`中。

*web.xml*

```xml
<!DOCTYPE web-app PUBLIC
 "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
 "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
  <display-name>Spring-Flow Web-Application Example</display-name>
  
  <servlet>
		<servlet-name>springFlowApplication</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>classpath://spring-config.xml</param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
	</servlet>
	
	 <servlet-mapping>
      <servlet-name>springFlowApplication</servlet-name>
      <url-pattern>/</url-pattern>
   </servlet-mapping>
</web-app>
```

下面是我们Flow的视图。其中包含了Spring Bean. input输入框的名字同Pojo中对应的名字相同。如果开发者想要名字不一样，他需要使用Spring tag library和`path`属性。

*login.jsp*

```jsp
<%@ page isELIgnored="false"%>
<html>
<body>
	<h2>Please Login</h2>

	<form method="post" action="${flowExecutionUrl}">

		<input type="hidden" name="_eventId" value="performLogin"> 
		<input type="hidden" name="_flowExecutionKey" value="${flowExecutionKey}" />

		<input type="text" name="userName" maxlength="40"><br> 
		<input type="password" name="password" maxlength="40">
		<input type="submit" value="Login" />

	</form>

</body>
</html>
```

下面是成功登录的视图。

*success.jsp*

```jsp
<%@ page language="java" contentType="text/html; charset=ISO-8859-1"
    pageEncoding="ISO-8859-1"%>
    <%@ page isELIgnored ="false" %>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=ISO-8859-1">
<title>Login Successful</title>
</head>
<body>
Welcome ${loginBean.userName}!!
</body>
</html>
```

当输入错误的凭据，用户会看到下面的视图。

*failure.jsp*

```jsp
<%@ page language="java" contentType="text/html; charset=ISO-8859-1"
    pageEncoding="ISO-8859-1"%>
    <%@ page isELIgnored ="false" %>
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=ISO-8859-1">
<title>Login Successful</title>
</head>
<body>
Invalid username or password. Please try again!
</body>
</html>
```

现在我们来部署并运行应用。效果如下：

*登录视图*

![Fig 1 : Login page](https://examples.javacodegeeks.com/wp-content/uploads/2016/05/Login-page.jpg)

*成功登录视图*

![Fig 2 : Successful login flow](https://examples.javacodegeeks.com/wp-content/uploads/2016/05/successful-login-flow.jpg)

*登录失败视图*

![Fig 3 : Failed Login Flow](https://examples.javacodegeeks.com/wp-content/uploads/2016/05/Failed-Login-Flow.jpg)

[点击此处，查看原文](https://examples.javacodegeeks.com/enterprise-java/spring/web-flow/spring-web-flow-tutorial/)