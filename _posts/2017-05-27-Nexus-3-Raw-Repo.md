---
published: true
layout: post
title: Nexus 3 Raw Maven Repository
---

The [Nexus Book](https://books.sonatype.com/nexus-book/3.0/reference/raw.html#maven-site) has a section on deploying a maven site to the new "raw" repository
option in Nexus 3. The step by step guide there is great but I found that it didn't work
out of the box for me so here are a couple of things to look out for.

## Raw Repo

The Nexus Book defines a raw repository as:

>Nexus Repository Manager Pro and Nexus Repository Manager OSS include support for hosting, proxying and grouping static websites - the raw format. Hosted repositories with this format can be used to store and provide a Maven-generated website. Proxy repositories can subsequently proxy them in other servers. The raw format can also be used for other resources than HTML files exposed by straight HTTP-like browsable directory structures.

The screen below shows a newly created repo in raw format with the name of `site`.

![raw-repo.png]({{site.baseurl}}/assets/raw-repo.png)

## Distribution Management

The book provides an example for the `<distributionManagement>` element but it's an 
unparameterized URI which isn't a good choice for enterprises with multiple modules.
The docs explain that the URI can be parameterized which is a more reasonable default.

```
<distributionManagement>
    <site>
        <id>nexus</id>
        <url>dav:http://localhost:8081/repository/site/${project.groupId}/${project.artifactId}/</url>
    </site>
</distributionManagement>
```

Obviously you'd change the host and port to match your setup. Using the project's `groupId` and 
`artifactId` is a good disambiguator. If you want to maintain separate sites for different
versions then you can have the `version` appended as well.

**Unfortunately, you cannot define this snippet in a parent enterprise pom for all of your
modules in your enterprise.** There's [some discussion](https://issues.apache.org/jira/browse/MNG-2290) about this on the Maven JIRA site. 
The problem is that the `distributionManagement` url is evaluated by the parent as opposed
to the value being interpreted by the child module. Defining the above in a
common enterprise pom will result every module publishing to the same address which isn't
what you want.

Currently the only workaround is to define the above or some form of it in every child module
that publishes a site.

## Wagon Plugin

I got errors publishing to Sonatype Nexus 3.3.1-01 with the example defined in the Nexus 
Book but everything worked smoothly after upgrading the wagon plugin used for publishing.

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-site-plugin</artifactId>
    <version>3.6</version>
    <dependencies>
        <dependency>
            <groupId>org.apache.maven.wagon</groupId>
            <artifactId>wagon-webdav-jackrabbit</artifactId>
            <version>2.12</version>
        </dependency>
    </dependencies>
</plugin>

```

## Executing

Some useful commands for testing/deploying the maven site:

* `mvn site:run` : useful to build and stage the site locally and preview it from a browser before publishing
* `mvn site-deploy` : full build and deploy to the configured `site` in `distributionManagement`

