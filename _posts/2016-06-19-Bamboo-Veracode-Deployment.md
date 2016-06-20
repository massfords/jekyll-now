---
published: false
layout: post
title: Deploying to Veracode with Bamboo
---
[Veracode](http://www.veracode.com) provides static and dynamic analysis of your application code. There are a couple of [scripts](https://github.com/OnLineStrategies/veracode-scripts/blob/master/submitToVeracode.py) out there for automating the deployment to their scanning service but I didn't see anything for providing push button deployments via [Bamboo](https://www.atlassian.com/software/bamboo) so here we go.

## Stage the Veracode API Jar in your Maven Repo

This step assumes that you have access to the Veracode customer portal and that you're running an internal maven repo like Nexus or Artifactory. Sorry for not providing a link, I got the JAR from our security tester.

```
mvn deploy:deploy-file 
-Dfile=VeracodeJavaAPI-16.3.3.0.jar 
-DrepositoryId=YOUR-REPO-ID 
-Durl=YOUR-REPO-URL 
-DgroupId=com.veracode 
-DartifactId=VeracodeJavaAPI 
-Dversion=16.3.3.0
```

The coordinate you pick is entirely up to you. I went with `com.veracode:VeracodeJavaAPI:16.3.3.0` We'll be using this coordinate to download the JAR for use in our project.

## Create a generic Maven POM to do the build

I'll review each of the steps for the build below and then show the whole pom at the end.

### Download the Veracode API Jar

Add the dependency that was deployed to your local maven repo with the proper coordinate as below:

```xml
        <dependency>
            <groupId>com.veracode</groupId>
            <artifactId>VeracodeJavaAPI</artifactId>
            <version>16.3.3.0</version>
        </dependency>
```

Use the `maven-dependency-plugin` to download the jar and place it in the target directory so we can use it to run the veracode deployment of your artifact.

```xml
<plugin>
    <artifactId>maven-dependency-plugin</artifactId>
    <executions>
        <execution>
            <id>copy-internal-dependencies</id>
            <phase>initialize</phase>
            <goals>
                <goal>copy-dependencies</goal>
            </goals>
            <configuration>
                <outputDirectory>${project.build.directory}</outputDirectory>
                <stripVersion>true</stripVersion>
                <stripClassifier>true</stripClassifier>
                <includeArtifactIds>VeracodeJavaAPI</includeArtifactIds>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### Download the module that you're testing

This is designed to be a generic POM that it can be used to submit any artifact for analysis. As such, the dependency declared on this artifact is done with properties that can be specified externally from the POM. We'll eventually do this with Bamboo variables but for now, it's shown as simple maven properties.

```xml
<dependency>
    <groupId>${dashD.groupId}</groupId>
    <artifactId>${dashD.artifactId}</artifactId>
    <version>${dashD.version}</version>
    <type>${dashD.type}</type>
</dependency>
```

I've prefaced the properties with `dashD` as a marker that they're designed to be specified as `-D` parameters on the mvn build line.

Another execution is added to the `maven-dependency-plugin` to download this module and place the artifact in the target directory that will be uploaded to Veracode.

```xml
<plugin>
    <artifactId>maven-dependency-plugin</artifactId>
    <executions>
        <execution>
            <id>copy-target-dependencies</id>
            <phase>initialize</phase>
            <goals>
                <goal>copy-dependencies</goal>
            </goals>
            <configuration>
                <outputDirectory>${dashD.veracode.filepath}</outputDirectory>
                <stripVersion>true</stripVersion>
                <stripClassifier>true</stripClassifier>
                <includeArtifactIds>${dashD.artifactId}</includeArtifactIds>
            </configuration>
        </execution>
    </executions>
</plugin>

```

Notice that the `outputDirectory` above is also parameterized. This is that same property we use below when we invoke the Veracode API upload process.

### Hybrid Maven / Ant Properties

The maven example from Veracode used the `maven-antrun-plugin` so I kept followed that example here. However, one tricky thing with this plugin is propagating properties to it from Maven and also allowing for these properties to be overidden. 

The pattern for this is to define the properties outside of the `maven-antrun-plugin` and then map those properties again in the `maven-antrun-plugin`.

#### Top Level Maven Properties

```xml
<properties>
    <dashD.veracode.action>UploadAndScan</dashD.veracode.action>
    <dashD.veracode.createprofile>false</dashD.veracode.createprofile>
    <dashD.veracode.criticality>VeryHigh</dashD.veracode.criticality>
    <dashD.veracode.filepath>target/veracode</dashD.veracode.filepath>
    <dashD.veracode.appname>???</dashD.veracode.appname>
    <dashD.veracode.vuser>???</dashD.veracode.vuser>
    <dashD.veracode.vpassword>???</dashD.veracode.vpassword>
</properties>
```

The properties above are a child of your `<project>` element. I've defined these with reasonable defaults except for the ones that will be specific to your organization. Like the properties used elsewhere, these are prefixed with `dashD` as a signal that they're meant to be passed on the maven command line.

#### Mapping them to Ant Properties

The plugin definition below invokes ant to simply dump the properties available to the ant processor as a way of ensuring that the POM is configured correctly. Note how there are mappings for the Top Level Maven Properties to bring them into the ant task.

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-antrun-plugin</artifactId>
    <version>1.8</version>
    <executions>
        <execution>
            <phase>package</phase>
            <configuration>
                <target name="UploadAndScan" description="Turns on debug symbols, logging.
                                                          Cleans, builds, uploads binaries.
                                                          Starts scan">
                    <property name="veracode.action" value="${dashD.veracode.action}"/>
                    <property name="veracode.createprofile" value="${dashD.veracode.createprofile}"/>
                    <property name="veracode.criticality" value="${dashD.veracode.criticality}"/>
                    <property name="veracode.appname" value="${dashD.veracode.appname}"/>
                    <property name="veracode.filepath" value="${dashD.veracode.filepath}"/>
                    <property name="veracode.vuser" value="${dashD.veracode.vuser}"/>
                    <property name="veracode.vpassword" value="${dashD.veracode.vpassword}"/>

                    <echoproperties />

                </target>
            </configuration>
            <goals>
                <goal>run</goal>
            </goals>
        </execution>
    </executions>
</plugin>

You may notice something odd with the echoproperties output. If you override one of the default properties with a -D command line override then you'll see that Maven Top Level Properties always appear to have their default values while the antrun properties will correctly see the overidden value.

For Example: try passing -DdashD.veracode.appname=FooBar and you'll see the following output from echoproperties:

```
dashD.veracode.appname=???
veracode.appname=FooBar
```

### Invoking the Vercode Java API

Add the following steps to your `maven-antrun-plugin` in order to invoke the Veracode Java API:

```xml
<!-- Create a timestamp value to use for the build id -->
<tstamp>
    <format property="current.time" pattern="yyyyMMdd-kmmssS" />
</tstamp>
<!-- Log all output to veracode.log file -->
<record name="veracode_${current.time}.log" loglevel="verbose" append="false"/>
<!-- Call the Veracode API to upload and scan -->
<java jar="${project.build.directory}/VeracodeJavaAPI.jar" fork="true">
    <arg line=" -action ${veracode.action}
                   -vuser ${veracode.vuser}
                   -vpassword ${veracode.vpassword}
                   -criticality ${veracode.criticality}
                   -createprofile ${veracode.createprofile}
                   -version ${current.time}
                   -appname ${veracode.appname}
                   -filepath ${veracode.filepath}"/>
</java>

```

### Putting it all together

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.massfords</groupId>
    <artifactId>veracode</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <properties>
        <dashD.veracode.action>UploadAndScan</dashD.veracode.action>
        <dashD.veracode.createprofile>false</dashD.veracode.createprofile>
        <dashD.veracode.criticality>VeryHigh</dashD.veracode.criticality>
        <dashD.veracode.appname>???</dashD.veracode.appname>
        <dashD.veracode.filepath>target/veracode</dashD.veracode.filepath>
        <dashD.veracode.vuser>super</dashD.veracode.vuser>
        <dashD.veracode.vpassword>password</dashD.veracode.vpassword>
    </properties>

    <dependencies>
        <dependency>
            <groupId>com.veracode</groupId>
            <artifactId>VeracodeJavaAPI</artifactId>
            <version>16.3.3.0</version>
        </dependency>
        <dependency>
            <groupId>${dashD.groupId}</groupId>
            <artifactId>${dashD.artifactId}</artifactId>
            <version>${dashD.version}</version>
            <type>${dashD.type}</type>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <artifactId>maven-dependency-plugin</artifactId>
                <executions>
                    <execution>
                        <id>copy-internal-dependencies</id>
                        <phase>initialize</phase>
                        <goals>
                            <goal>copy-dependencies</goal>
                        </goals>
                        <configuration>
                            <outputDirectory>${project.build.directory}</outputDirectory>
                            <stripVersion>true</stripVersion>
                            <stripClassifier>true</stripClassifier>
                            <includeArtifactIds>VeracodeJavaAPI</includeArtifactIds>
                        </configuration>
                    </execution>
                    <execution>
                        <id>copy-target-dependencies</id>
                        <phase>initialize</phase>
                        <goals>
                            <goal>copy-dependencies</goal>
                        </goals>
                        <configuration>
                            <outputDirectory>${dashD.veracode.filepath}</outputDirectory>
                            <stripVersion>true</stripVersion>
                            <stripClassifier>true</stripClassifier>
                            <includeArtifactIds>${dashD.artifactId}</includeArtifactIds>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-antrun-plugin</artifactId>
                <version>1.8</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <configuration>
                            <target name="UploadAndScan" 
                                           description="Turns on debug symbols, logging.
														Cleans, builds, uploads binaries.
                                                        Starts scan">
                                <property name="veracode.action" value="${dashD.veracode.action}"/>
                                <property name="veracode.createprofile"
                                          value="${dashD.veracode.createprofile}"/>
                                <property name="veracode.criticality"
                                          value="${dashD.veracode.criticality}"/>
                                <property name="veracode.appname" 
                                          value="${dashD.veracode.appname}"/>
                                <property name="veracode.filepath"
                                          value="${dashD.veracode.filepath}"/>
                                <property name="veracode.vuser" 
                                          value="${dashD.veracode.vuser}"/>
                                <property name="veracode.vpassword"
                                          value="${dashD.veracode.vpassword}"/>

                                <echoproperties />

                                <!-- Create a timestamp value to use for the build id -->
                                <tstamp>
                                    <format property="current.time" pattern="yyyyMMdd-kmmssS" />
                                </tstamp>
                                <!-- Log all output to veracode.log file -->
                                <record name="veracode_${current.time}.log" 
                                        loglevel="verbose" append="false"/>
                                <!-- Call the Veracode API to upload and scan -->
                                <java jar="${project.build.directory}/VeracodeJavaAPI.jar"
                                      fork="true">
                                    <arg line=" -action ${veracode.action}
                                                   -vuser ${veracode.vuser}
                                                   -vpassword ${veracode.vpassword}
                                                   -criticality ${veracode.criticality}
                                                   -createprofile ${veracode.createprofile}
                                                   -version ${current.time}
                                                   -appname ${veracode.appname}
                                                   -filepath ${veracode.filepath}"/>
                                </java>
                            </target>
                        </configuration>
                        <goals>
                            <goal>run</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>

```

## Creating a Bamboo Plan with Properties

Add a Bamboo Plan to your existing project group for the Veracode submission. I make this a separate plan in order to allow the security group the opportunity to pick whatever version they want to submit for test throughout the project's lifecycle.

You can easily add a plan like this to other projects as well. All you need to do is to change the overridden variables accordingly.

### Define Bamboo Variables

![bamboo-vars.png]({{site.baseurl}}/_posts/bamboo-vars.png)

Note that if you include "password" in the variable name then Bamboo will mask its value.


### Define the Job

![maven-task.png]({{site.baseurl}}/_posts/maven-task.png)

The job consists of a source repo task that pulls the pom.xml in from your repo and then executes the Maven task. We'll run clean package and then have one -D param for each of the variables. These variables are defined in the Bamboo.

Note: using the source repository task is a simple way of kicking this job off but you may want to replace that something that pulls the pom.xml from your repo instead. This would remove the undesirable connection to the source in order to avoid kicking off every Veracode related job in the event that you tweak one of the settings in the shared POM.


## Running a Customized Plan

Bamboo allows you to run a customized plan. This is an easy way to changing one of the baked in variables w/o having to mess with the properties view or otherwise edit the plan.

![override-var.png]({{site.baseurl}}/_posts/override-var.png)

The dialog above shows overriding the version in order to pull a different version than the one specified in the plan.


