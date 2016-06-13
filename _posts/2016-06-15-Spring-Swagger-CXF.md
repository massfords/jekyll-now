---
published: false
layout: post
---
I seem to be in the shrinking minority of people that still use [Apache CXF](http://cxf.apache.org). I used this library in my SOAP / Web Services days and it seemed like a natural choice for my foray into REST. One problem though is that a lot of the latest cool libraries like [swagger](http://swagger.io) are focused on [Jersey](https://jersey.java.net) and ignore CXF in favor of detailed confg walk thrus for dropwizard/Jersey.

Here's how I got swagger working with CXF along with one gotcha.

## maven dependency

Add the latest version of swagger and CXF to your project's dependencies

        <dependency>
            <groupId>io.swagger</groupId>
            <artifactId>swagger-jaxrs</artifactId>
            <version>1.5.8</version>
        </dependency>
        <dependency>
        	<groupId>org.apache.cxf</groupId>
            <artifactId>cxf-rt-rs-service-description</artifactId>
            <version>3.1.6</version>
        </dependency?


## Spring jaxrs:server

Add the CXF Feature for Swagger to your jaxrs:server

    <jaxrs:server address="/rest" >
        <jaxrs:serviceBeans>
             <!-- your beans -->
        </jaxrs:serviceBeans>
        <jaxrs:providers>
        	<!-- your providers -->
        </jaxrs:providers>
        <jaxrs:features>
            <ref bean="swagger2Feature" />
        </jaxrs:features>
    </jaxrs:server>

## CXF Feature

Add the CXF Feature to your context. You can provide additional values to override the default settings in swagger. 

    <bean id="swagger2Feature" class="org.apache.cxf.jaxrs.swagger.Swagger2Feature">
        <property name="resourcePackage" value="com.example.services.api"/>
        <property name="basePath" value="/services/rest"/>
        <property name="scan" value="true"/>
        <property name="prettyPrint" value="true"/>
    </bean>
    
## One Important Detail

I found the most important difference with the recent version of swagger was using a different package name for my service interfaces from my implementations. I found that if I used the same package name for the implementation and the interface, the swagger CXF feature would find both during the discovery phase and it wouldn't render the swagger interface correctly.

This is meaningful if you use Java interfaces as artifacts and a separate implementation without the JAX-RS annotations. I'll post about this another time but suffice to say for now, I have a single Java interface with all of my JAX-RS annotations and it's in a separate package than the implementation because of this issue in CXF/Swagger with loading the API details.

