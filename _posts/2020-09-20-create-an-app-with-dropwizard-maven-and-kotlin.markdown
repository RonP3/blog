---
layout: post
title: "Create an app with Dropwizard, Maven and Kotlin"
date: 2020-09-20 14:33:01 +0300
categories: jekyll update
---

In this post, I am going to show you how to create a simple and basic Dropwizard app. This is going to be a technical tutorial.

At the end of this tutorial, your app will expose a RESTful API.
The code is written in Kotlin, but you can do the same with Java.

I will start with a short explanation about Dropwizard and Maven.

## Dropwizard 
Dropwizard is a light Java framework for building RESTful web services. Dropwizard collects together stable libraries, such as Jersey, Jackson, and JDBI, into a lightweight package.

## Maven
Maven is a build automation and management tool for Java.
With Maven, it is easy to define the building of the software and the dependencies.
POM is an XML file that describes a Maven project and its dependencies.

Maven in Yiddish is "person with understanding, expert". This is the origin of the name for this tool.




## Creating a new project
------
I will use IntelliJ Community Edition IDE in this tutorial.
<br/>
<br/>
Click on File --> New --> Project --> Maven  
You are going to see the following screen:
![](/photos/drop0.png?raw=true)
Click on "Next". Now you are ready to define the POM file.

## Defining the POM file
As I described at the beginning of the tutorial, we have to define the dependencies of the project in the POM.xml file. Let's do it together.

#### Properties
In the properties tag, define the version of Dropwizard and Kotlin you want to use.
You can also define Kotlin compiler configurations through properties. In this case, I defined languageVersion (you want to ensure source compatibility with the specified version of Kotlin) and jvmTarget (the target version of the generated JVM bytecode).
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
![](/photos/drop0-1.png?raw=true)

We can define parameters in YAML configuration file which is deserialized to an instance of our configuration class. Let's create both of them.
Create my-app-config.yml in src/main/resources and add one parameter to this file as follow:
```xml
configTest: Ron
```

Now, we define our configuration class which extends the Dropwizard Configuration class. Create MyAppConfig.kt in the configuration package. Here is the code:
```kotlin
package com.ronp.mydropwizardapp.configuration

import com.fasterxml.jackson.annotation.JsonProperty
import io.dropwizard.Configuration


class MyAppConfig(@JsonProperty("configTest") val configTest: String): Configuration()
```
	

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]: https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
