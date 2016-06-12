---
published: true
layout: post
---
I've found myself settling on a multi-module maven structure for my web-apps as a matter of habit. The [Maven Book](https://books.sonatype.com/mvnref-book/reference/pom-relationships-sect-pom-best-practice.html) offers an introduction to the multi-module structure in its Best Practices section but it's quite dense and a lot to get through. The [Apache Maven tutorial](http://www.codetab.org/apache-maven-tutorial/maven-multi-module-project/) offers a gentler introduction. There's also some good SO questions on this topic which show some [strong opinions about complexity](http://stackoverflow.com/questions/15559041/maven-multi-module-benefits-over-simple-dependency) and raise some good points about [versioning](http://programmers.stackexchange.com/questions/194674/should-we-use-a-maven-multi-module-project-in-our-scenario). I arrived at this structure slowly over time so I thought I'd provide a few simple points on why I prefer this for my web-apps.

## Basic Structure

    service-groupA-api/
    service-groupA-impl/
    service-groupB-api/
    service-groupB-impl/
    myapp-web/
    myapp-web-ui/
    pom.xml
    
The directory structure above has 6 child modules in it plus the parent pom that brings them all together. See the [tutorial](http://www.codetab.org/apache-maven-tutorial/maven-multi-module-project/) for details on the relationship between the modules and their parent.

### *-api modules

I like defining my REST services with annotated Java interfaces and will put these interfaces and their request/response objects into api modules. The two modules above `service-groupA-api` and `service-groupB-api` represent a logical grouping of two sets of non-overlapping services. If there's any shared code between them then they might have a dependency on a `service-common-api` module (not shown).

Putting these service interfaces into ther own modules creates a nice encapsulation of the service interface. It reminds me of the good old days of WSDL when there was a single artifact to define your interface. This same artifact can be achieved with an annotated Java interface and this packaging puts it into its own module.

If you have a QA engineering team writing integration tests then having an api module that you can share with them helps them along in writing their tests since the interfaces could be invoked directly through Apache CXF's client REST proxy. These modules could also be shared with internal or external customers through an SDK since they only contain the interfaces for the services and not the implementations. 

Share these modules with your QA team or SDK group as follows:
```
       <dependency>
         <groupId>com.example</groupId>
         <artifactId>service-groupA</artifactId>
         <version>${your-version-number}</version>
       </dependency>
```


### *-impl modules

These are the implementations of the services. Again, the assumption here is that there's a logical grouping resulting in `service-groupA-impl` and `service-groupB-impl` but they could all be in the same module. Since these modules contain your implementation of the services, it's not something that you'd share outside the team.

### web-ui

I like to separate the actual UI files for the web-app to its own module and often times to its own top level module. I'm showing it as another child module here but I've found that the tools to build a web ui application like AngularJS or similar is very different than a Java application so it may make sense for you to kick this out to its own top level module.

One plus of having it as a separate module is that the UI guys never need to both building/checking out the whole web app. They can work locally with mock services and develop the entire web application UI independent of the server if they choose.

Whichever style you choose, this module will ultimately produce a WAR that contains just the HTML/JS and other supporting files for the web app UI. It's important that it's a WAR because we'll use the maven-war-plugin overlay feature in order to merge this into our web-app

### web module

Since this is a web app, the web module is what brings everything together as a web app. One thing to note is that you *never* put Java code into this module. The reason for that is that in a WAR type module in maven, the Java code is never visible outside of the module as a dependency and therefore there is no way of reusing it. 

The web module for most of my projects simply has the spring security files, web.xml, or other config files. This module has a dependency on all of the other modules needed for the web-app to start up in a proper servlet container.

## Problems

There are a number of potential problems with this setup and you should review these before taking this route.

### Build Cycles

Your build fails due to a cycle. Module A depends on module B which depends on module C which depends on module A. A->B->C->A equals cycle.

>Like a lot of things in Maven, if you hit a wall or it's hard to do then you're probably doing it wrong.

You wouldn't have this problem if all of the code were in the same module. However, that's really only hiding your problem. In reality, your code is failing Uncle Bob's [Acycilc Dependencies Principle](https://en.wikipedia.org/wiki/Acyclic_dependencies_principle).

You can fix this by moving some of the code to a new module which is then shared between the two. If this is difficult to do, then your code is more tangled then it should be and you should refactor before trying to move to a multi-module project.

Sadly, I've hit build cycles more than a few times. 

### Unnecessary Versions

Each time you cut a release you're creating a new version of each module. The parent module sets the version and all of the child modules use this same version. This is the easiest configuration so I won't explore others.

When you cut a release of the main module Parent, then the child modules Child1, Child2, and Child3 are each released as well with the parent's version. Over time, you may find that Child3 for example doesn't change at the same rate as the others. In fact, it's possible that Child3 hasn't changed in the last 10 versions. When looking at the git history of Child3, you may see that the only change in the last 10 versions was its pom.xml so it could point to the latest Parent version.

You can address this by doing one of the following:

#### Extract Utility Module

If the module in question has some value across other projects, then extract it to its own top level module and build it separately. Add it as a dependency back into your main module and in whatever other projects it's needed. Your parent module will thank you because it's less to build and your developers will thank you because it's less code to check out and maybe they'll get value in other projects.

#### Extract as a Supporting Lib

This is the exact same steps as the "Extract Utility Module" above except that you recognize that this module has no value outside of your project. You're only extracting it to a top level module because of the version number concerns. 

> Don't do this, it's silly

#### Ostrich Algorithm

Accept the multiple versions. Disk space is cheap.
