# CAS 使用

刘理想 2016-12-27

[TOC]

## 1. 基本使用

### 1.1 安装JDK/Tomcat

很简单，跳过。不要忘记设置`JAVA_HOME`

### 1.2 给Tomcat配置SSL

1. 生成SSL证书：跳转到`JDK/bin`安装目录，运行:
```shell
keytool -genkey -alias cas-server-liulx -validity 7000 -keyalg RSA -keypass changeit -storepass changeit -keystore cas.keystore
```

2. 导出证书SSL：
```shell
keytool -export -alias cas-server-liulx -keypass changeit -file cas-server-liulx.crt -keystore cas.keystore -storepass changeit
```

3. 导入证书：
```shell
keytool -import -file cas-server-liulx.crt -alias cas-server-liulx -keypass changeit -keystore .\..\jre\lib\security\cacerts -storepass changeit
```

4. 配置Tomcat的`conf/server.xml`:找到`Connector`SSL一节，取消注释，并且更改为：
```xml
<Connector port="8443" protocol="HTTP/1.1" SSLEnabled="true"
           maxThreads="150" scheme="https" secure="true"
           clientAuth="false" sslProtocol="TLS" 
           keystoreFile="D:/src/ssl/cas.keystore"
           keystorePass="changeit"
           truststoreFile="C:/Program Files (x86)/Java/jre6/lib/security/cacerts"/>
```

### 1.3 部署CAS Server

打开下载的CAS Server压缩包，进入modules文件夹，解压出来`cas-server-webapp-3.2.1.war`，为了方便，重命名为`cas.war`，拷贝到tomcat的webapps中。

启动Tomcat，打开http://localhost:8080/cas/login 测试test/test

### 1.4 CASify HelloWorld Servlet 

在`WEB-INF`中创建`lib`文件夹，拷贝`casclient.jar`进去。更改`web.xml`，添加：

```xml
<filter>
	<filter-name>CAS Filter</filter-name>
  	<filter-class>edu.yale.its.tp.cas.client.filter.CASFilter</filter-class>
  	<init-param>
      <param-name>edu.yale.its.tp.cas.client.filter.loginUrl</param-name>
      <param-value>https://localhost:8443/cas/login</param-value>
  	</init-param>
  	<init-param>
      <param-name>edu.yale.its.tp.cas.client.filter.validateUrl</param-name>
      <param-value>https://localhost:8443/cas/serviceValidate</param-value>
  	</init-param>  
  	<init-param>
      <param-name>edu.yale.its.tp.cas.client.filter.serverName</param-name>
      <param-value>https://localhost:8080</param-value>
  	</init-param>  
</filter>
<filter-mapping>
	<filter-name>CAS Filter</filter-name>
  	<url-pattern>/servlet/HelloWorldExample</url-pattern>
</filter-mapping>
```

此时访问http://localhost:8080/servlets-examples/servlet/HelloWorldExample 会自动跳转到CAS登陆页