---
layout: post
title: "Authentication and Authorization in Dropwizard App (Kotlin)"
date: 2020-10-09 16:34:01 +0300
---
Do you know how to perform authentication and authorization in Dropwizard? Neither did I, but I had to learn it for a recent work project and I thought I would share what I have learned with you.
<br />
<br />
### Authentication
We are using authentication to identify **who the user is**. Most of the times the authentication process accours during the sign-in step.
<br />
<br />
There are several authentication schemes that is used by the HTTP authentication framework.  
In this post, we will talk about the "Basic" authentication schemes for simplicity.  
An important note: In Basic scheme, the user ID and password are passed in base64-encoding over the network. HTTPS/TLS should be used if you are using this scheme. The reason for this is that base64 is reversible encoding, as you willin this post.
<br />
<br />
The Basic scheme flow:  
![](/photos/basicAuthDiagram1-0.png?raw=true)
<br />
1. The client tries to access a server resource through HTTP GET.  
2. The server sends a 401 Unauthorized response along with WWW-Authenticate header, which defines the authentication method. In our case the authentication method is "Basic".  
3. The client can trigger HTTP GET including an Authorization header with the credentials, in order to authenticate.  
4. In this step there are 3 options:  
- The server will send HTTP 200 OK, which indicates that the authentication has succeeded.
- The server will send a HTTP 401 Unauthorized response again, which means that the credentials were wrong.
- The server will send HTTP 403 forbidden. In this case, the user authenticated successfully, but does not have the right permissions in order to access the resource.  
<br />
<br />
You can find the official HTTP/1.1 Authentication documentation in [RFC-7235](https://tools.ietf.org/html/rfc7235)

### Authorization
We use authorization to manage users permissions in our app.  
For example, if we want to create an API for user deletion, we would probably like to give a permission for this API just for the admin of the system, and not for a regular user.  
The relevant error status code is 403 Forbidden. When a 403 is sent, authentication is possible, but the user does not have the right permissions.
<br />
<br />

## How to Implement Authentication and Authorization in Dropwizard App
I believe it is easier to understand how all of these things work through a real example, so let's start to implement authentication and authorization from scratch in our Dropwizard app.  
I assume you know how to create a dropwizard app, if not, you can read my previous post [Create an app with Dropwizard, Maven and Kotlin](https://www.ronpaz.dev/jekyll/update/2020/09/20/create-an-app-with-dropwizard-maven-and-kotlin.html).

#### POM Dependency
In order to use Dropwizard authentication, you should add the following dependency to POM file:
```xml
<dependency>
	<groupId>io.dropwizard</groupId>
	<artifactId>dropwizard-auth</artifactId>
	<version>${dropwizard.version}</version>
</dependency>
```

#### Defining User Roles
We will define 2 enums, the first one represents the user role. The second one represents group roles.  
For example, the ADMIN group role has all the permissions - GUEST, USER and ADMIN permissions.  
The GUEST user group role has only GUEST permissions.
```kotlin
package com.ronp.mydropwizardapp.auth

enum class Role(val roleName: String) {
    GUEST(roleName = "Guest"),
    USER(roleName = "User"),
    ADMIN(roleName = "Admin");
}

enum class GroupRoles(val roles: Set<Role>) {
    GUEST(roles = setOf(Role.GUEST)),
    USER(roles = setOf(Role.GUEST, Role.USER)),
    ADMIN(roles = setOf(Role.GUEST, Role.USER, Role.ADMIN))
}
```

#### Implementing the Principle Object
Once the user is authenticated, the application establishes the Principle. The Principle is an object which represents the currently-logged-in user in the context of the application.  
One user can have multiple IDs, for example, if the user has several gmail accounts. However, there is usually just one logged-in user per request and this is going to be the Principle.  
<br />
We will implement a [java.security.Principal](https://docs.oracle.com/javase/8/docs/api/java/security/Principal.html) interface as follows:
```kotlin
package com.ronp.mydropwizardapp.auth

import java.security.Principal

data class MyAppUser(private val name: String, val groupRoles: GroupRoles): Principal {
    override fun getName() = name
}
```  

#### Creating a Custom Authenticator
Now we can create our authenticator which implements [Interface Authenticator<C,P extends Principal>](https://javadoc.io/static/io.dropwizard/dropwizard-auth/1.3.9/io/dropwizard/auth/Authenticator.html)  
The authenticator gets user basic credentials as input and should validate them. If the authentication succeeds,the authenticate function should return a Principle object (user) in an Optional container. Otherwise, an empty Optional is returned, indicating that the credentials are invalid.  
Here is the implementation:  
```kotlin
package com.ronp.mydropwizardapp.auth

import io.dropwizard.auth.Authenticator
import io.dropwizard.auth.basic.BasicCredentials
import java.util.*


class MyAppBasicAuthenticator : Authenticator<BasicCredentials, MyAppUser> {
    override fun authenticate(credentials: BasicCredentials): Optional<MyAppUser> {
        return if ("secret" == credentials.password) {
            val groupRoles = GroupRoles.valueOf(credentials.username.toUpperCase())
            val user = MyAppUser(name = credentials.username, groupRoles = groupRoles)
            Optional.of(user)
        } else {
            Optional.empty()
        }
    }
}
```
Note: this is a simple authenticator implementation just for demonstration.

#### Creating a Custom Authorizer
Our authorizer should implement the [Interface Authorizer< P extends Principal >](https://javadoc.io/static/io.dropwizard/dropwizard-auth/1.3.9/io/dropwizard/auth/Authorizer.html), which has a single method "authorize" that gets a Principle (user) and role, and is responsible for deciding if access is granted to this principle.
```kotlin
package com.ronp.mydropwizardapp.auth

import io.dropwizard.auth.Authorizer

class MyAppAuthorizer: Authorizer<MyAppUser> {
    override fun authorize(user: MyAppUser, role: String): Boolean {
        return user.groupRoles.roles.contains(Role.valueOf(role.toUpperCase()))
    }
}
```

#### Authenticator and Authorizer Registration
We are ready to register the custom authenticator and authorizer in our Dropwizard app.

```kotlin
package com.ronp.mydropwizardapp

import com.ronp.mydropwizardapp.auth.MyAppAuthorizer
import com.ronp.mydropwizardapp.auth.MyAppBasicAuthenticator
import com.ronp.mydropwizardapp.auth.MyAppUser
import com.ronp.mydropwizardapp.configuration.MyAppConfig
import com.ronp.mydropwizardapp.resources.HelloWorldResource
import io.dropwizard.Application
import io.dropwizard.auth.AuthDynamicFeature
import io.dropwizard.auth.AuthValueFactoryProvider
import io.dropwizard.auth.basic.BasicCredentialAuthFilter
import io.dropwizard.setup.Environment
import org.glassfish.jersey.server.filter.RolesAllowedDynamicFeature

class MyApp: Application<MyAppConfig>() {
    companion object {
        @JvmStatic fun main(args: Array<String>) = MyApp().run(*args)
    }

    override fun run(myAppConfig: MyAppConfig, environment: Environment) {
        environment.jersey().register(HelloWorldResource(myAppConfig.configTest))
        environment.jersey().register(AuthDynamicFeature(
                BasicCredentialAuthFilter
                        .Builder<MyAppUser>()
                        .setAuthenticator(MyAppBasicAuthenticator())
                        .setAuthorizer(MyAppAuthorizer())
                        .setRealm("Secret stuff")
                        .buildAuthFilter()))
        environment.jersey().register(RolesAllowedDynamicFeature::class.java)
        environment.jersey().register(AuthValueFactoryProvider.Binder(MyAppUser::class.java))
    }
}
```
```AuthDynamicFeature``` enables HTTP basic authentication.  
```RolesAllowedDynamicFeature``` enables HTTP authorization.  
```AuthValueFactoryProvider``` allows you to use @Auth to inject a custom Principal type into your resource.  
In this example we are using BasicCredentialAuthFilter which supplies us with an out-of-box authentication filter (implementation of ```AuthFilter```).  
It is possible to define a custom filter if needed. To do that you should use ```AuthDynamicFeature``` and implement the ```AuthFilter``` Dropwizard class.


#### Secure Our Resources
There are 2 options to secure your resource:
1. Use one of ```@RolesAllowed```, ```@DenyAll```, ```@PermitAll``` annotations.  Each one of this annotations will trigger the authenticator and the authentication filter, in order to validate the credentials, and the authorizer, in order to check the permissions.
2. Use @Auth annotation, in order to trigger the authenticator and the authentication filter.

**An important note:** Using ```@PermitAll``` and not setting any of these annotations is not the same! If you don't set one of these annotations, the authenticator will not be triggered, so any user can access this API.


```kotlin
package com.ronp.mydropwizardapp.resources

import com.ronp.mydropwizardapp.auth.MyAppUser
import io.dropwizard.auth.Auth
import javax.annotation.security.RolesAllowed
import javax.ws.rs.GET
import javax.ws.rs.Path

@Path("/")
class HelloWorldResource(private val property: String) {

    @RolesAllowed("Admin")
    @Path("/helloWorld")
    @GET
    fun helloWorld(@Auth user: MyAppUser) = "Hello World $property :)"
}
```

#### Testing our security
To complete
