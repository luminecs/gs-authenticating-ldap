Getting Started: Authenticating a user with LDAP
================================================

What you'll build
-----------------

This Getting Started guide will walk you through the process configuring an application to be secured by an LDAP server.

What you'll need
----------------

 - About 15 minutes
 - A favorite text editor or IDE
 - [JDK 6][jdk] or later
 - [Maven 3.0][mvn] or later

[jdk]: http://www.oracle.com/technetwork/java/javase/downloads/index.html
[mvn]: http://maven.apache.org/download.cgi

How to complete this guide
--------------------------

Like all Spring's [Getting Started guides](/getting-started), you can start from scratch and complete each step, or you can bypass basic setup steps that are already familiar to you. Either way, you end up with working code.

To **start from scratch**, move on to [Set up the project](#scratch).

To **skip the basics**, do the following:

 - [Download][zip] and unzip the source repository for this guide, or clone it using [git](/understanding/git):
`git clone https://github.com/springframework-meta/{@project-name}.git`
 - cd into `{@project-name}/initial`
 - Jump ahead to [Create a resource representation class](#initial).

**When you're finished**, you can check your results against the code in `{@project-name}/complete`.


<a name="scratch"></a>
Set up the project
------------------

First you set up a basic build script. You can use any build system you like when building apps with Spring, but the code you need to work with [Maven](https://maven.apache.org) and [Gradle](http://gradle.org) is included here. If you're not familiar with either, refer to [Getting Started with Maven](../gs-maven/README.md) or [Getting Started with Gradle](../gs-gradle/README.md).

### Create the directory structure

In a project directory of your choosing, create the following subdirectory structure; for example, with `mkdir -p src/main/java/hello` on *nix systems:

    └── src
        └── main
            └── java
                └── hello

### Create a Maven POM

`pom.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.springframework</groupId>
    <artifactId>gs-authenticating-ldap-initial</artifactId>
    <version>0.1.0</version>

    <parent>
        <groupId>org.springframework.bootstrap</groupId>
        <artifactId>spring-bootstrap-starters</artifactId>
        <version>0.5.0.BUILD-SNAPSHOT</version>
    </parent>

    <properties>
        <start-class>hello.Application</start-class>
        <spring-security.version>3.1.3.RELEASE</spring-security.version>
		<apacheds.version>1.5.5</apacheds.version>
     </properties>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.bootstrap</groupId>
            <artifactId>spring-bootstrap-web-starter</artifactId>
        </dependency>
        <dependency>
        	<groupId>org.springframework.security</groupId>
        	<artifactId>spring-security-ldap</artifactId>
        	<version>${spring.security.version}</version>
        </dependency>
        <dependency>
        	<groupId>org.springframework.security</groupId>
        	<artifactId>spring-security-web</artifactId>
        	<version>${spring.security.version}</version>
        </dependency>
        <dependency>
        	<groupId>org.springframework.security</groupId>
        	<artifactId>spring-security-config</artifactId>
        	<version>${spring.security.version}</version>
        </dependency>
        <dependency>
        	<groupId>org.apache.directory.server</groupId>
        	<artifactId>apacheds-server-jndi</artifactId>
        	<version>${apacheds.version}</version>
        </dependency>
    </dependencies>
    
    <!-- TODO: remove once bootstrap goes GA -->
    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>http://repo.springsource.org/milestone</url>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
        </repository>
        <repository>
            <id>spring-snapshots</id>
            <name>Spring Snapshots</name>
            <url>http://repo.springsource.org/snapshot</url>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
        </repository>
    </repositories>
    <pluginRepositories>
        <pluginRepository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>http://repo.springsource.org/milestone</url>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
        </pluginRepository>
        <pluginRepository>
            <id>spring-snapshots</id>
            <name>Spring Snapshots</name>
            <url>http://repo.springsource.org/snapshot</url>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
        </pluginRepository>
    </pluginRepositories>
</project>
```

TODO: mention that we're using Spring Bootstrap's [_starter POMs_](../gs-bootstrap-starter) here.

> Note to experienced Maven users who don't use an external parent project: You can take out the project later, it's just there to reduce the amount of code you have to write to get started.



<a name="initial"></a>
Creating a simple web controller
--------------------------------
In Spring, REST endpoints are just Spring MVC controllers. The following Spring MVC controller handles a `GET /` request by returning a simple message:

`src/main/java/hello/HomeController.java`
```java
package hello;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
public class HomeController {

	@RequestMapping("/")
	public @ResponseBody String index() {
		return "Welcome to the home page!";
	}
}
```
    
First of all, this entire class is marked up with `@Controller` so Spring MVC can pick it up and look for routes.

Next, the method has been tagged with `@RequestMapping` to flag the path and the REST action. In this case, `GET` is the default behavior, it will return back a very simple message indicating you are on the home page. 

Finally, `@ResponseBody` tells Spring MVC to write the text directly into the HTTP response body. That's because there aren't any views. Instead, when you visit the page, you'll get a very simple message in the browser. That's because the focus in this guide is on securing the page with LDAP.

Running an unsecured web application
-------------------------------------
Before you secure the web application, it's best to verify it works first. To do that, we need to wire up a web application configuration.

`src/main/java/hello/Application.java`
```java
package hello;

import org.springframework.bootstrap.SpringApplication;
import org.springframework.bootstrap.context.annotation.EnableAutoConfiguration;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;

@Configuration
@ComponentScan
@EnableWebMvc
@EnableAutoConfiguration
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
	
}
```
    
Using Spring Bootstrap, that's enough to launch our unsecured web application.

    mvn package && java -jar target/gs-authenticating-ldap-complete-0.1.0.jar

Open a browser and visit <http://localhost:8080/>, and you should see a simple message.

```
Welcome to the home page!
```

Setting up Spring Security
----------------------------
To configure Spring Security, you need an XML application context file. Let's create **application-context.xml**. To make it easier to read, it has been configured by default to use the `security` namespace.

`src/main/resources/application-context.xml`
```xml
<beans:beans xmlns="http://www.springframework.org/schema/security"
  xmlns:beans="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans 
           http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
           http://www.springframework.org/schema/security
           http://www.springframework.org/schema/security/spring-security-3.1.xsd">
     
	<ldap-server root="dc=springframework,dc=org" ldif="classpath:test-server.ldif" />

	<authentication-manager>
		<ldap-authentication-provider user-dn-pattern="uid={0},ou=people" 
												group-search-base="ou=groups"/>
	</authentication-manager>

	<http auto-config="true">
		<intercept-url pattern="/**" access="ROLE_DEVELOPERS" />
	</http>	

</beans:beans>
```

> **Note:** Unfortunately, at the time of writing, a Java-based version of Spring Security setup is still under development.

First of all, we need an LDAP server. While we could install a full-blown directory server to provide this, Spring Security's LDAP module includes an embedded one written in pure Java. It we eventually replace the embedded LDAP server with a real one, it's a one-line change to point to the real one.

Next, we need to declare `<authentication-manager>` as the component that will handle all authentication requests. In this setup, it contains an `<ldap-authentication-provider>`. It's configured to take a username, and insert it into `{0}` and look for `uid={0},ou=people,dc=springframework,dc=org`

Finally, we need the `<http>` component to declare a set of URL intercepts as well as some other automatic components such as form authentication.

Wiring an XML application context into our Java configuration
-------------------------------------------------------------
To wire in Spring Security, we need to update our configuration class and pull in the XML configuration we just defined.

`src/main/java/hello/Application.java`
```java
package hello;

import org.springframework.bootstrap.SpringApplication;
import org.springframework.bootstrap.context.annotation.EnableAutoConfiguration;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.ImportResource;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;

@Configuration
@ImportResource("classpath:application-context.xml")
@ComponentScan
@EnableWebMvc
@EnableAutoConfiguration
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
	
}
```

`@ImportResource` pulls **application-context.xml** into this application context configuration.

Setting up user data
--------------------

LDAP servers can exchange user data using LDIF files. We have it configured to pull in a file using the **ldif** parameter of the `<ldap-server>` element in **application-context.xml**.

`src/main/resources/test-server.ldif`
```ldif
dn: ou=groups,dc=springframework,dc=org
objectclass: top
objectclass: organizationalUnit
ou: groups

dn: ou=subgroups,ou=groups,dc=springframework,dc=org
objectclass: top
objectclass: organizationalUnit
ou: subgroups

dn: ou=people,dc=springframework,dc=org
objectclass: top
objectclass: organizationalUnit
ou: people

dn: ou=space cadets,dc=springframework,dc=org
objectclass: top
objectclass: organizationalUnit
ou: space cadets

dn: ou=\"quoted people\",dc=springframework,dc=org
objectclass: top
objectclass: organizationalUnit
ou: "quoted people"

dn: ou=otherpeople,dc=springframework,dc=org
objectclass: top
objectclass: organizationalUnit
ou: otherpeople

dn: uid=ben,ou=people,dc=springframework,dc=org
objectclass: top
objectclass: person
objectclass: organizationalPerson
objectclass: inetOrgPerson
cn: Ben Alex
sn: Alex
uid: ben
userPassword: {SHA}nFCebWjxfaLbHHG1Qk5UU4trbvQ=

dn: uid=bob,ou=people,dc=springframework,dc=org
objectclass: top
objectclass: person
objectclass: organizationalPerson
objectclass: inetOrgPerson
cn: Bob Hamilton
sn: Hamilton
uid: bob
userPassword: bobspassword

dn: uid=joe,ou=otherpeople,dc=springframework,dc=org
objectclass: top
objectclass: person
objectclass: organizationalPerson
objectclass: inetOrgPerson
cn: Joe Smeth
sn: Smeth
uid: joe
userPassword: joespassword

dn: cn=mouse\, jerry,ou=people,dc=springframework,dc=org
objectclass: top
objectclass: person
objectclass: organizationalPerson
objectclass: inetOrgPerson
cn: Mouse, Jerry
sn: Mouse
uid: jerry
userPassword: jerryspassword

dn: cn=slash/guy,ou=people,dc=springframework,dc=org
objectclass: top
objectclass: person
objectclass: organizationalPerson
objectclass: inetOrgPerson
cn: slash/guy
sn: Slash
uid: slashguy
userPassword: slashguyspassword

dn: cn=quote\"guy,ou=\"quoted people\",dc=springframework,dc=org
objectclass: top
objectclass: person
objectclass: organizationalPerson
objectclass: inetOrgPerson
cn: quote\"guy
sn: Quote
uid: quoteguy
userPassword: quoteguyspassword

dn: uid=space cadet,ou=space cadets,dc=springframework,dc=org
objectclass: top
objectclass: person
objectclass: organizationalPerson
objectclass: inetOrgPerson
cn: Space Cadet
sn: Cadet
uid: space cadet
userPassword: spacecadetspassword



dn: cn=developers,ou=groups,dc=springframework,dc=org
objectclass: top
objectclass: groupOfNames
cn: developers
ou: developer
uniqueMember: uid=ben,ou=people,dc=springframework,dc=org
uniqueMember: uid=bob,ou=people,dc=springframework,dc=org

dn: cn=managers,ou=groups,dc=springframework,dc=org
objectclass: top
objectclass: groupOfNames
cn: managers
ou: manager
uniqueMember: uid=ben,ou=people,dc=springframework,dc=org
uniqueMember: cn=mouse\, jerry,ou=people,dc=springframework,dc=org

dn: cn=submanagers,ou=subgroups,ou=groups,dc=springframework,dc=org
objectclass: top
objectclass: groupOfNames
cn: submanagers
ou: submanager
uniqueMember: uid=ben,ou=people,dc=springframework,dc=org
```
    
> **Note:** Using an LDIF file isn't standard configuration for a production system. However, it's very useful for testing purposes or guides.


Building and Running the Secured Web Application
------------------------------------------------
With Spring Security wired in, you can now run it in secured mode:

    mvn package && java -jar target/gs-authenticating-ldap-complete-0.1.0.jar

Great! If you visit the site at <http://localhost:8080>, you should get redirected to a login page provided by Spring Security.

Enter username **ben** and password **benspassword**. It should then let you in to see a very simple message in your browser.

```
Welcome to the home page!
```

Summary
-------
Congratulations! You have just written a web application and secured it with [Spring Security](http://static.springsource.org/spring-security/site/docs/3.2.x/reference/springsecurity-single.html). In this case, you used an [LDAP-based user store](http://static.springsource.org/spring-security/site/docs/3.2.x/reference/springsecurity-single.html#ldap).

[zip]: https://github.com/springframework-meta/gs-authenticating-ldap/archive/master.zip
