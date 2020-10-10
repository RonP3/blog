---
layout: post
title: "Authentication and Authorization in dropwizard app (Kotlin)"
date: 2020-10-09 16:34:01 +0300
---
**To think about a better introduction**  
This time I will talk about authentication and authorization and we will add them into a Dropwizard app.
<br />
<br />
### Authentication
We are using authentication to identify **who the user is**. Most of the times the authentication process accourd during the sign-in step.
<br />
<br />
There are several authentication schemes that is used by the HTTP authentication framework.  
In this tutorial, we will talk about the "Basic" authentication schemes for simplicity.  
An important note: In Basic scheme, the user ID and password are passed in base64-encoding over the network. HTTPS/TLS should be used if you are using this scheme. The reason for this is that base64 is reversible encoding, we will see it during this tutorial.
<br />
<br />
The Basic scheme flow:  
![](/photos/basicAuthDiagram1-0.png?raw=true)
<br />
1. The client try to access a server resource through HTTP GET.  
2. The server send a 401 Unauthorized response along with WWW-Authenticate header which defined the authentication method. In our case the authentication method is "Basic".  
3. The client can trigger HTTP GET including an Authorization header with the credentials, in order to authenticate.  
4. In this step there are 3 options:  
- The server will send HTTP 200 OK, which indicates that the authentication has succeded.
- The server will send again HTTP 401 Unauthorized response, which means that the credentials were wrong.
- The server will send HTTP 403 forbidden. In this case, the user authenticated successfully, but he has no the right permissions in order to access the resource.  
<br />
<br />
You can find the official HTTP/1.1 Authentication documentation in [RFC-7235](https://tools.ietf.org/html/rfc7235)

### Authorization
We are using authorization to manage users permissions in our app.  
For example, if we want to create an API for user deletion, probebly, we would like to give a permission for this API just for the admin of the system, and not for a regular user.  
The relevant error status code is 403 Forbidden. When 403 is being sent, authenticaion is possible, but the user has not the right permissions.
<br />
<br />

## How to implement Authentication and Authorization in Dropwizard app
I believe it is more easy to understand how all of these things are working through a real example, so lets start to implemet Authentication an Authorization from scratch in our Dropwizard app.  
I assume you know how to create a dropwizard app, if no, you can read my previous post [Create an app with Dropwizard, Maven and Kotlin](https://www.ronpaz.dev/jekyll/update/2020/09/20/create-an-app-with-dropwizard-maven-and-kotlin.html).

#### POM dependency
In order to use Dropwizard authentication, you should add the following dependency to POM file:
```xml
<dependency>
	<groupId>io.dropwizard</groupId>
	<artifactId>dropwizard-auth</artifactId>
	<version>${dropwizard.version}</version>
</dependency>
```

#### Defining User Roles
We will define 2 enums, the first one is for representing the user role. The second one is for representing group of roles.  
For example, the ADMIN group roles has all the permissions - GUEST, USER and ADMIN permissions.  
The GUEST user group roles has onle GUEST permissions.
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

#### Implement the Principle object
Once the user is authenticated, the application establishes the Principle. The Principle is an object whice represent the currently logged in user in the context of the applicatiom.  
One user can have multiple IDs, for example, if the user has several gmail accounts. But there is usually just one logged-in user per request and this is going to be the Principle.  
<br />
We will implement [java.security.Principal](https://docs.oracle.com/javase/8/docs/api/java/security/Principal.html) interface as follows:
```kotlin
package com.ronp.mydropwizardapp.auth

import java.security.Principal

data class MyAppUser(private val name: String, val groupRoles: GroupRoles): Principal {
    override fun getName() = name
}
```  

#### Creating a custom Authenticator
Now we can create our authenticator which implements [Interface Authenticator<C,P extends Principal>](https://javadoc.io/static/io.dropwizard/dropwizard-auth/1.3.9/io/dropwizard/auth/Authenticator.html)  
The authenticator gets user basic credentials as input and should validate them. If the authentiction successeded,the authenticate function should return a Principle object (user) in an Optional container. Otherwise, an empty Optional is returned, indicating that the credentials are invalid.  
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
Note: this is a simple authenticator implementation just for simplicity.

#### Creating a custom Authorizer
Our authorizer should implement the [Interface Authorizer< P extends Principal >](https://javadoc.io/static/io.dropwizard/dropwizard-auth/1.3.9/io/dropwizard/auth/Authorizer.html), which has a single method "authorize" that gets a Principle (user) and role, and responsible to decide if access is granted to this principle.
```kotlin
package com.ronp.mydropwizardapp.auth

import io.dropwizard.auth.Authorizer

class MyAppAuthorizer: Authorizer<MyAppUser> {
    override fun authorize(user: MyAppUser, role: String): Boolean {
        return user.groupRoles.roles.contains(Role.valueOf(role.toUpperCase()))
    }
}
```

#### Authenticator and Authorizer registering
We are ready to register the custom authenticator and authorizer into our Dropwizard app.

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
In this example we are using BasicCredentialAuthFilter which supply us an out-of-box authentication filter (implementation of ```AuthFilter```).  
It is possible to define a custom filter if needed. To do that you should use ```AuthDynamicFeature``` and implement the ```AuthFilter``` Dropwizrd class.


#### Secure our resources
There are 2 options to secure your resource:
1. Using one of ```@RolesAllowed```, ```@DenyAll```, ```@PermitAll``` annotations.  Each one of this annotations will trigger the authenticator and the authentication filter, in order to validate the credentials, and the authorizer, in order to check the permissions.
2. Using @Auth annotation on, in order to trigger the authenticator and the authentication filter.

**An important note:** Using ```@PermitAll``` and not set any of these annotations is not the same! If you don't set one of these annotations, the authenticator will not be triggered, so any user can access to this API.


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
