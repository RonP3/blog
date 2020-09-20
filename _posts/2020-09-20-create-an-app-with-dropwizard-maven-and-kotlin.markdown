---
layout: post
title: "Create an app with Dropwizard, Maven and Kotlin"
date: 2020-09-20 14:33:01 +0300
categories: jekyll update
---

In this post, I am going to show you how to create a simple and basic Dropwizard app.  
This is going to be a technical tutorial.

At the end of this tutorial, your app will expose a RESTful API.  
The code is written in Kotlin, but you can do the same with Java.    

I will start with a short explanation about Dropwizard and Maven.

## Dropwizard 
[Dropwizard](https://www.dropwizard.io/) is a light Java framework for building RESTful web services.  
Dropwizard collects together stable libraries, such as Jersey, Jackson, and JDBI, into a lightweight package.

## Maven
[Maven](https://maven.apache.org/) is a build automation and management tool for Java.  
With Maven, it is easy to define the building of the software and the dependencies.  
POM is an XML file that describes a Maven project and its dependencies.  
</br>
Maven in Yiddish is "person with understanding, expert". This is the origin of the name for this tool.



## Creating a new project
I will use IntelliJ Community Edition IDE in this tutorial.
<br/>
<br/>
Click on File --> New --> Project --> Maven  
You are going to see the following screen:
![](/photos/drop0.png?raw=true)
Click on "Next". Now you are ready to define the POM file.

## Defining the POM file
As I described at the beginning of the tutorial, we have to define the dependencies of the project in the POM.xml file.  
Let's do it together.

#### Properties
In the properties tag, define the version of Dropwizard and Kotlin you want to use.  
You can also define Kotlin compiler configurations through properties.  
In this case, I defined languageVersion (you want to ensure source compatibility with the specified version of Kotlin) and jvmTarget (the target version of the generated JVM bytecode).
```xml
<properties>
	<dropwizard.version>2.0.13</dropwizard.version>
	<kotlin.version>1.4.10</kotlin.version>
	<kotlin.compiler.languageVersion>1.4</kotlin.compiler.languageVersion>
	<kotlin.compiler.jvmTarget>1.8</kotlin.compiler.jvmTarget>
</properties>
```

#### Dependencies
Your project needs dependencies in order to compile, build, run and test.
</br>
In the dependencies tag, we will define dropwizard-core and kotlin-stdlib-jdk8

```xml
<dependencies>
	<dependency>
		<groupId>io.dropwizard</groupId>
		<artifactId>dropwizard-core</artifactId>
		<version>${dropwizard.version}</version>
	</dependency>

	<dependency>
		<groupId>org.jetbrains.kotlin</groupId>
		<artifactId>kotlin-stdlib-jdk8</artifactId>
		<version>${kotlin.version}</version>
	</dependency>
</dependencies>
```

Broadly speaking, dropwizard-core module includes:
* Jetty - HTTP server.
* Jersey -  RESTful web framework.
* Jackson - JSON library for the JVM.
* Metrics - library for application metrics.

#### Compilation
First, we will specify the source directories in <sourceDirectory> and in <testSourceDirectory> in order to compile the source code.
</br>
The <plugin> section is responsible for the configuration of the Kotlin plugin in order to compile our source code. 

```xml
<build>
	<sourceDirectory>${project.basedir}/src/main/kotlin</sourceDirectory>
	<testSourceDirectory>${project.basedir}/src/test/kotlin</testSourceDirectory>
	<plugins>
		<plugin>
			<artifactId>kotlin-maven-plugin</artifactId>
			<groupId>org.jetbrains.kotlin</groupId>
			<version>${kotlin.version}</version>
			<executions>
				<execution>
					<id>compile</id>
					<goals>
						<goal>compile</goal>
					</goals>
				</execution>
				<execution>
					<id>test-compile</id>
					<goals>
						<goal>test-compile</goal>
					</goals>
				</execution>
			</executions>
		</plugin>
	</plugins>
</build>
```

## Project structure and configuration file
Here is my project structure, you can create the same structure.
</br>
![](/photos/drop0-1.png?raw=true)
</br>
We can define parameters in YAML configuration file which is deserialized to an instance of our configuration class.
</br>
Let's create both of them. Create my-app-config.yml in src/main/resources and add one parameter to this file as follow:
```xml
configTest: Ron
```

Now, we define our configuration class which extends the Dropwizard Configuration class.
</br>
Create MyAppConfig.kt in the configuration package. Here is the code:
```kotlin
package com.ronp.mydropwizardapp.configuration

import com.fasterxml.jackson.annotation.JsonProperty
import io.dropwizard.Configuration


class MyAppConfig(@JsonProperty("configTest") val configTest: String): Configuration()
```
We use the JsonProperty to allow Jackson to deserialize the properties (in our cast, configTest) from YAML file to our application's Configuration instance.	

## Creating an Application class
To run an instance of our Dropwizard RESTful server we have to implement our Application class.
</br>
Create a new MyApp class in mydropwizardapp and implement as following:

```kotlin
package com.ronp.mydropwizardapp

import com.ronp.mydropwizardapp.configuration.MyAppConfig
import com.ronp.mydropwizardapp.resources.HelloWorldResource
import io.dropwizard.Application
import io.dropwizard.setup.Environment

class MyApp: Application<MyAppConfig>() {
    companion object {
        @JvmStatic fun main(args: Array<String>) = MyApp().run(*args)
    }

    override fun run(myAppConfig: MyAppConfig, environment: Environment) {
        // we are going to implement this function soon
    }
}
```

## Creating and registering a resource class
Now we can add our first endpoint!
</br>
To do that we need to create a new Jersey REST resource.
</br>
In mydropwizardapp/resources create a new class - HelloWorldResource.kt

```kotlin
package com.ronp.mydropwizardapp.resources

import javax.ws.rs.GET
import javax.ws.rs.Path

@Path("/helloWorld")
class HelloWorldResource(private val property: String) {
    @GET
    fun helloWorld() = "Hello World $property :)"
}
```
As you can see, a resource is associated with a URI template, in our case it is "/helloWorld".
</br>
The @Path annotation tells Jersey that this resource is available at the URI "/helloWorld".
</br>
</br>
Now, we are ready to complete the MyApp code and register our resource to the application.
</br>
In the run method write the following code in order to register HelloWorldResource. 
```kotlin
package com.ronp.mydropwizardapp

import com.ronp.mydropwizardapp.configuration.MyAppConfig
import com.ronp.mydropwizardapp.resources.HelloWorldResource
import io.dropwizard.Application
import io.dropwizard.setup.Environment

class MyApp: Application<MyAppConfig>() {
    companion object {
        @JvmStatic fun main(args: Array<String>) = MyApp().run(*args)
    }

    override fun run(myAppConfig: MyAppConfig, environment: Environment) {
        environment.jersey().register(HelloWorldResource(myAppConfig.configTest))
    }
}
```
Notice that we are using the configuration YAML and pass the property to the resource (myAppConfig.configTest).

## Running the application
We are ready to run our application!
</br>
To do that you should edit the "Program arguments" in MyApp running configurations as follows:
</br>
![](/photos/drop6.PNG?raw=true)
</br>
It tells to the Dropwizard app to run as a server and specify the location of the configuration file.
</br>
</br>
Now you can click on run:
</br>
![](/photos/drop7.PNG?raw=true)
</br>
You would see the following info messages on the run tab: 
</br>
![](/photos/drop9.PNG?raw=true)
</br>
That means our resource was registered properly as a GET request on /helloWorld path.
</br>
</br>
![](/photos/drop8.PNG?raw=true)
</br>
That means our app is running on port 8080.
</br>
</br>
</br>
In order to call and test our resource you can write:
</br>
```curl HTTP://localhost:8080/helloWorld```
</br>
and you should see the following expected output:
</br>
![](/photos/drop10.PNG?raw=true)
</br>
Another option is to open your browser and type http://localhost:8080/helloWorld on the address line.
</br>
</br>
As you can see, we got the required output, including the config property. **Congratulations!**




Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]: https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
