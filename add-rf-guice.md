# [](#header-1)Add RequestFactory and Guice

> We'll replace vanilla GWT-RPC with RequestFactory. RequestFactory keeps track of objects that have been modified and sends only changes to the server, which results in very lightweight network payloads.   

> Guice provides Dependency Injection on the server side  

1. Java 7
1. GWT2.8.0beta1
1. GAE 1.9.42
1. Gradle
1. IntelliJ IDEA Ultimate 2016.3
1. GWTP 1.5.3 Model View Presenter architecture
1. Guice 3.0 Dependency Injection
1. **RequestFactory**

[Github version after completing this tutorial](https://github.com/EwartM/MyAppName/tree/8a1bb0cbfdc7518f2cf4573aedb5f3f89d81f944)  
Note: If you get RF errors with HTTP 500 status after cloning this repo please check that you have Annotation processing enabled as described below.

#### [](#header-2)Prerequisite

[Read the RequestFactory docs](http://www.gwtproject.org/doc/latest/DevGuideRequestFactory.html#using)

#### [](#header-2)Overview

At the end of this tutorial the project structure should look like this:
![]({{ site.baseurl }}/assets/images/Screen Shot 2017-03-08 at 6.52.48 PM.png)

#### [](#header-2)Add RequestFactory dependencies

Add RF libraries to build.gradle.  

```XML
group 'com.example'
version '1.0-SNAPSHOT'

apply plugin: 'war'

repositories {
    mavenCentral()
}
// Dependency Versions
ext{
    gwtpVer = '1.5.3'
    jbcryptVer = '0.3m'
    jodaVer = '2.9.1'
    guiceVer = '3.0'
    hibernateVer = '4.1.0.Final'
    sl4jVer = '1.6.1'
    gwtqueryVer = '1.4.3'
    requestFactoryVer = '2.8.0-beta1'
}
dependencies {
    compile group: 'com.gwtplatform', name: 'gwtp-mvp-client', version: gwtpVer
    compile group: 'org.mindrot', name: 'jbcrypt', version: jbcryptVer
    compile group: 'joda-time', name: 'joda-time', version: jodaVer
    compile group: 'com.google.inject', name: 'guice', version: guiceVer
    compile group: 'com.google.inject.extensions', name: 'guice-assistedinject', version: guiceVer
    compile group: 'com.google.inject.extensions', name: 'guice-servlet', version: guiceVer
    compile group: 'org.hibernate', name: 'hibernate-validator-annotation-processor', version: hibernateVer
    compile(group: 'org.hibernate', name: 'hibernate-validator', version: hibernateVer) {
        exclude(module: 'jaxb-api')
        exclude(module: 'jaxb-impl')
        exclude(module: 'slf4j-api')
    }
    compile(group: 'org.hibernate', name: 'hibernate-validator', version: hibernateVer, classifier:'sources') {
        exclude(module: 'jaxb-api')
        exclude(module: 'jaxb-impl')
        exclude(module: 'slf4j-api')
    }
    compile group: 'org.slf4j', name: 'slf4j-log4j12', version: sl4jVer
    compile group: 'com.googlecode.gwtquery', name: 'gwtquery', version: gwtqueryVer
    compile group: 'com.google.web.bindery', name: 'requestfactory-server', version: requestFactoryVer
    compile group: 'com.google.web.bindery', name: 'requestfactory-apt', version: requestFactoryVer


    testCompile 'junit:junit:4.11'
}

task explodedWar(type: Copy) {
    into "$buildDir/exploded"
    with war
}

war.dependsOn explodedWar
```

Do a Gradle > Tasks > Build > Build.  

Right-click the App module and 'Open Module Settings'. Go to Artifacts > App:war exploded.  
Click 'Fix' at the 'lib ...missing' warning.  
Open WEB-INF/lib and remove gwt-dev and gwt-user jars. Click 'OK'.    

Finally click the 'build project' button.  
![]({{ site.baseurl }}/assets/images/Screen Shot 2017-03-10 at 1.03.30 PM.png)


#### [](#header-2)Inherit RequestFactory

Inherit RequestFactory in the app.gwt.mxl  

```XML
<!DOCTYPE module PUBLIC "-//Google Inc.//DTD Google Web Toolkit 2.8.0//EN"
        "http://gwtproject.org/doctype/2.8.0/gwt-module.dtd">
<module rename-to="app">

    <!-- Inherit the core Web Toolkit stuff.                  -->
    <inherits name='com.google.gwt.user.User'/>
    <inherits name='com.google.gwt.inject.Inject' />
    <inherits name="com.google.gwt.i18n.I18N"/>

    <!-- GWTP -->
    <inherits name='com.gwtplatform.mvp.Mvp'/>
    <entry-point class='com.example.app.client.MyEntryPoint'/>
    <!-- GIN -->
    <extend-configuration-property name="gin.ginjector.modules"
                                   value="com.example.app.client.gin.ClientModule"/>
    <!-- Specify the paths for translatable code -->
    <source path="client"/>
    <source path="shared"/>

    <!-- RequestFactory -->
    <inherits name='com.google.web.bindery.requestfactory.RequestFactory' />

</module>
```

#### [](#header-2)'Users' Server POJO and Client Proxy  

With RequestFactory we create a POJO on the server and corresponding Proxy on the client

Create a New > Java class called 'User' in the com.example.app.server.domain folder  
```java
package com.example.app.server.domain;

public class User extends DomainObject {

    //No primitives allowed
    private String firstName;
    private String lastName;
    private String email = null;
    private String cleartextPassword;
    private Boolean loggedIn = false;

    public String getFirstName() {
        return firstName;
    }
    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }
    public String getLastName() {
        return lastName;
    }
    public void setLastName(String lastName) {
        this.lastName = lastName;
    }
    public String getEmail() {
        return email;
    }
    public void setEmail(String email) {
        this.email = email;
    }
    public String getCleartextPassword() {
        return cleartextPassword;
    }
    public void setCleartextPassword(String cleartextPassword) {
        this.cleartextPassword = cleartextPassword;
    }
    public Boolean isLoggedIn() {
        return loggedIn;
    }
    public void setLoggedIn(Boolean loggedIn) {
        this.loggedIn = loggedIn;
    }
}
```

Create a New > Java class called 'UserProxy' in the com.example.app.client.proxy folder  
```java
package com.example.app.client.proxy;

import com.example.app.server.domain.User;
import com.google.web.bindery.requestfactory.shared.EntityProxy;
import com.google.web.bindery.requestfactory.shared.ProxyFor;

@ProxyFor(value = User.class)
public interface UserProxy extends EntityProxy {

    String getFirstName();
    void setFirstName(String firstName);

    String getLastName();
    void setLastName(String lastName);

    String getEmail();
    void setEmail(String email);

    String getCleartextPassword();
    void setCleartextPassword(String cleartextPassword);

    Boolean isLoggedIn();
}
```

#### [](#header-2)Base class for domain objects

RequestFactory keeps track of object changes / versions. Every RF object needs to have an id & version so all our RF objects should extend the class below.  

Create a New > Java class called 'DomainObject' in the com.example.app.server.domain folder  
```java
package com.example.app.server.domain;

public class DomainObject {

    private Long id = null;

    private Integer version = 0;

    /**
     * Auto-increment version # whenever persisted
     */
    void onPersist()
    {
        this.version++;
    }

    public Long getId()
    {
        return id;
    }

    public void setId(Long id)
    {
        this.id = id;
    }

    public Integer getVersion()
    {
        return version;
    }

    public void setVersion(Integer version)
    {
        this.version = version;
    }

    // Equality based on ID field
    @Override
    public boolean equals(Object o)
    {
        if(o != null && (o instanceof DomainObject)) {
            return ((DomainObject)o).getId().compareTo(id) == 0;
        }
        return false;
    }

    // Required for checking equality
    @Override
    public int hashCode()
    {
        return ((Long)id).hashCode();
    }

}
```

#### [](#header-2)User service

We'll set up service for the client to interact with our User POJO. 

Create a New > Java class called 'UserService' in the com.example.app.server.service folder  
```java
package com.example.app.server.service;

import com.example.app.server.domain.User;

public class UserService {

    /**
     * PUBLIC
     * Check submitted credentials against DB
     */
    public User authenticate (String submittedEmail, String submittedPassword) {

        User user = null;

        String testEmail = "admin@example.com";
        String testPassword = "admin";

        if (testEmail != null) {
            if (submittedEmail.matches(testEmail) &&
                    submittedPassword.matches(testPassword)) {
                user = new User();
                user.setEmail(submittedEmail);
            }
        }

        return user;
    }
}
```
Don't worry, we will get to security later.  


#### [](#header-2)Wiring up RequestFactory

Create a New > Java class called **'MyRequestFactory'** in the com.example.app.shared.service folder  

Service methods like the ones we created above must be added into our RequestFactory class.   

```java
package com.example.app.shared.service;

import com.example.app.client.proxy.UserProxy;
import com.example.app.server.locator.DaoServiceLocator;
import com.example.app.server.service.UserService;
import com.google.web.bindery.requestfactory.shared.Request;
import com.google.web.bindery.requestfactory.shared.RequestContext;
import com.google.web.bindery.requestfactory.shared.RequestFactory;
import com.google.web.bindery.requestfactory.shared.Service;

public interface MyRequestFactory extends RequestFactory {

    //USER
    UserRequest userRequest();

    @Service(value = UserService.class, locator = DaoServiceLocator.class)
    public interface UserRequest extends RequestContext {
        Request<UserProxy> authenticate(String submittedEmail, String submittedPassword);
    }

}
```

We need a locator class to help Request Factory find our services. So create a New > Java class called **'DaoServiceLocator'** in the com.example.app.server.locator folder  
```java
package com.example.app.server.locator;

import javax.servlet.ServletContext;
import javax.servlet.http.HttpServletRequest;
import com.google.inject.Injector;
import com.google.web.bindery.requestfactory.server.RequestFactoryServlet;
import com.google.web.bindery.requestfactory.shared.ServiceLocator;

/**
 * Generic locator service that can be referenced in the @Service annotation
 * for any RequestFactory service stub
 */
public class DaoServiceLocator implements ServiceLocator {
    @SuppressWarnings("unchecked")
    @Override
    public Object getInstance(@SuppressWarnings("rawtypes") Class clazz) {
        HttpServletRequest request = RequestFactoryServlet.getThreadLocalRequest();
        ServletContext servletContext = request.getSession().getServletContext();
		/* during the GuiceServletContextListener the Injector is sticked to the ServletContext with the attribute name Injector.class.getName()*/
        Injector injector = (Injector) servletContext.getAttribute(Injector.class.getName());

        if (injector == null) {
            //throw new IllegalStateException(“No injector found. Must be set as attribute in the servlet context with the name ” + Injector.class.getName());
            System.out.println( "MYERROR: No injector found. Must be set as attribute in the servlet context with the name " + Injector.class.getName() );
        }

        // Resolve Binding with Guice
        return injector.getInstance(clazz);
    }
}
```


#### [](#header-2)Add the Guice files

Create a New > Java class called **'ServerModule'** with the content below in the com.example.app.server.guice folder  
```java
package com.example.app.server.guice;

import com.example.app.server.ServerBootstrapper;
import com.example.app.server.service.UserService;
import com.google.inject.AbstractModule;
import com.google.inject.Singleton;
import com.google.web.bindery.requestfactory.server.RequestFactoryServlet;

public class ServerModule extends AbstractModule {
    @Override
    protected void configure() {
        //Request factory
        bind(RequestFactoryServlet.class).in(Singleton.class);
        //Bootstrap
        bind(ServerBootstrapper.class).asEagerSingleton();
        //Services
        bind(UserService.class).in(Singleton.class);
    }
}
```

Create a New > Java class called **'MyServletModule'** with the content below in the com.example.app.server.guice folder  
```java
package com.example.app.server.guice;

import java.util.HashMap;
import java.util.Map;
import com.google.inject.servlet.ServletModule;
import com.google.web.bindery.requestfactory.server.RequestFactoryServlet;

public class MyServletModule extends ServletModule {

    @Override protected void configureServlets() {

        // NOTE: BOTH Request Factory AND Remote logging use the parameter below
        Map<String, String> params = new HashMap<String, String>();
        params.put("symbolMapsDirectory", "WEB-INF/deploy/app/symbolMaps/");

        // Request Factory (see parameters note above)
        serve("/gwtRequest").with( RequestFactoryServlet.class, params);

    }
}
```

Create a New > Java class called **'MyServletConfig'** with the content below in the com.example.app.server.guice folder  
```java
package com.example.app.server.guice;

import com.google.inject.Guice;
import com.google.inject.Injector;
import com.google.inject.servlet.GuiceServletContextListener;

public class MyServletConfig extends GuiceServletContextListener {
    @Override
    protected Injector getInjector() {
        return Guice.createInjector(
                new ServerModule(),
                new MyServletModule());
    }
}
```

Next, add a New > Java class called **'ServerBootstrapper'** with the content below in the com.example.app.server folder. We'll use this class later.    
```java
package com.example.app.server;

import com.google.inject.Inject;

public class ServerBootstrapper {

    @Inject
    ServerBootstrapper(
    ) {
        //TODO
    }

}
```

Finally, **configure the Guice servlet in App.web.WEB-INF.web.xml** 
```XML
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">
    <!-- Default page to serve -->
    <welcome-file-list>
        <welcome-file>app.html</welcome-file>
    </welcome-file-list>

    <!-- Configure Guice servlet, other servlets configured in MyServletModule -->
    <filter>
        <filter-name>guiceFilter</filter-name>
        <filter-class>com.google.inject.servlet.GuiceFilter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>guiceFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    <listener>
        <listener-class>com.example.app.server.guice.MyServletConfig</listener-class>
    </listener>

</web-app>
```

#### [](#header-2)Enable annotation processing for the project

RequestFactory requires that we enable annotation processing to avoid the dreaded 'the RequestFactory validation tool must be run' error.

![]({{ site.baseurl }}/assets/images/Screen Shot 2017-03-08 at 6.30.36 PM.png)

#### [](#header-2)Make a RequestFactory call from the client 

Add some textboxes to **client.ui.home.HomeView.ui.xml** 
```XML
<!DOCTYPE ui:UiBinder SYSTEM "http://dl.google.com/gwt/DTD/xhtml.ent">
<ui:UiBinder xmlns:ui="urn:ui:com.google.gwt.uibinder"
             xmlns:g="urn:import:com.google.gwt.user.client.ui">
    <g:HTMLPanel ui:field="main">
        <g:TextBox ui:field="txt_email"/>
        <g:TextBox ui:field="txt_password"/>
        <g:Button ui:field="btn_login" text="Sign in"/>
        <g:Label ui:field="lbl_result"/>
    </g:HTMLPanel>
</ui:UiBinder>
```

Make the following changes to **client.ui.home.HomeView**    
```java
package com.example.app.client.ui.home;

import com.example.app.client.proxy.UserProxy;
import com.example.app.shared.service.MyRequestFactory;
import com.google.gwt.event.dom.client.ClickEvent;
import com.google.gwt.event.dom.client.ClickHandler;
import com.google.gwt.uibinder.client.UiBinder;
import com.google.gwt.uibinder.client.UiField;
import com.google.gwt.user.client.Window;
import com.google.gwt.user.client.rpc.AsyncCallback;
import com.google.gwt.user.client.ui.*;
import com.google.inject.Inject;
import com.google.web.bindery.requestfactory.shared.Receiver;
import com.google.web.bindery.requestfactory.shared.ServerFailure;
import com.gwtplatform.mvp.client.ViewWithUiHandlers;

public class HomeView extends ViewWithUiHandlers<HomeUiHandlers> implements HomePresenter.MyView {
    interface Binder extends UiBinder<Widget, HomeView> {
    }

    private final MyRequestFactory requestFactory;

    @UiField
    HTMLPanel main;
    @UiField
    TextBox txt_email;
    @UiField
    TextBox txt_password;
    @UiField
    Button btn_login;
    @UiField
    Label lbl_result;

    @Inject
    HomeView(
            Binder uiBinder,
            MyRequestFactory requestFactory
        ) {
        initWidget(uiBinder.createAndBindUi(this));

        this.requestFactory = requestFactory;

        btn_login.addClickHandler(new ClickHandler() {
            public void onClick(ClickEvent event) {
                checkLogin();
            }
        });
    }

    private void checkLogin() {

        String email = txt_email.getText();
        String password = txt_password.getText();

        requestFactory.userRequest().authenticate(email,password).fire(
                new Receiver<UserProxy>() {
                    @Override
                    public void onSuccess(UserProxy user) {
                        if (user != null) {
                            lbl_result.setText("User logged in");
                        } else {
                            lbl_result.setText("Invalid credentials");
                        }
                    }
                    @Override
                    public void onFailure(ServerFailure error) {
                        Window.alert(error.getMessage() + error.getStackTraceString());
                    }
                });
    }

}
```

You should now be able to debug the application or upload it to App Engine and see the following:  
![]({{ site.baseurl }}/assets/images/Screen Shot 2017-03-08 at 8.03.39 PM.png)



[Previous - adding GWTP MVP architecture](add-gwtp-mvp-architecture)    
[Next - adding a Login presenter](add-login-presenter)
