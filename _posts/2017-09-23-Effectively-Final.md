---
published: true
layout: post
title: Effectively Final
---

It's best practice to not assign to a method parameter in the body of your method. You can find lots of opinions about
this but here's a good summary on [Stack Overflow](https://stackoverflow.com/questions/500508/why-should-i-use-the-keyword-final-on-a-method-parameter-in-java).

However, it's a bit tedious to have to write the modifier "final" before each parameter declaration and with the
addition of Java 8's ["effecitvely final"](http://javarevisited.blogspot.com/2015/03/what-is-effectively-final-variable-of.html#ixzz3WFtLyLuz) concept it seems reasonable to relax the requirement of having the final
modifier on the parameter and instead have the compiler enforce this for us. This is less typing and less clutter in your code.

## Explicitly final:

```java

public class ExplicitlyFinalExample {

    public void sendData(final String alpha, final String beta, final int gamma, final List<String> omega) {
        System.out.printf("sending %s, %s, %s, %s\n", alpha, beta, gamma, omega);
    }
}
```

## Effectively final:

```java

public class EffectivelyFinalExample {
    public void sendData(String alpha, String beta, int gamma, List<String> omega) {
        System.out.printf("sending %s, %s, %s, %s\n", alpha, beta, gamma, omega);
    }
}
```

In both snippets above, the method params are final. They are either final because they are explicitly declared
as being final or they are effectively final because they are never assigned to.

However, it's easy for the programmer to break this contract in the effectivley final example by assigning to one of the method params in the body of the method. As long as this method parameter isn't used in an Anonymous Class or within a lambda expression, it's still valid. 

**This is something we can detect with a Java Compiler Plugin and fail the build**

## Reporting an error when breaking the effectively final contract:

```java

public class BreakingEffectivelyFinalExample {
    public void sendData(String alpha, String beta, int gamma, List<String> omega) {
    
        if (gamma < 100) {
            gamma = 100; // <|--- plugin will generate a compilation error on this line
        }
    
        System.out.printf("sending %s, %s, %s, %s\n", alpha, beta, gamma, omega);
    }
}
```

### Compilation Error

```
BreakingEffectivelyFinalExample.java:[5,13] error: EFFECTIVELY_FINAL: Assignment to param in `gamma = 100`
```


# Plugin Details

There's a working copy of this plugin in [github](https://github.com/massfords/effectively-final) and published to Maven Central. 

Java 8 introduced the ability to write plugins for the Java Compiler. When properly configured, your instance of [Plugin](https://docs.oracle.com/javase/8/docs/jdk/api/javac/tree/com/sun/source/util/Plugin.html) will load that allows you to visit the compilation unit and in this case perform a little extra validation.

The plugin consists of the following components:

## Plugin Implementation
You need to write a class that implements the [Plugin](https://docs.oracle.com/javase/8/docs/jdk/api/javac/tree/com/sun/source/util/Plugin.html) interface. The plugin proper doesn't do anything other than advertise its name and when initialized add a TaskListener.

## TaskListener

The [TaskListener](https://docs.oracle.com/javase/8/docs/jdk/api/javac/tree/com/sun/source/util/TaskListener.html) receives callbacks to indicate when a task is starting or stopping. In our case we only care about the finished event of the analyze stage.

## Scanner / Visitor

The [TreePathScanner](https://docs.oracle.com/javase/8/docs/jdk/api/javac/tree/com/sun/source/util/TreePathScanner.html) allows you to traverse the compilation unit and visit each of the nodes in the [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree). In our case, we want to visit each of the methods to see if the parameters are assigned to in violation of the effectively final rule.

If we're visiting a method, we want to record any of its non-final params and if we have one or more non-final param then we want to traverse into the body and hit every assignment.

We're relying on javac to report any real errors and thus our logic here can be simpler.

For example, when visiting a method, we can ignore any context passed into us since any params in scope from enclosing methods must already be explicitly or effectively final.

Consider the following:

```java
void foo(int a, int b) {

     MyClass mc = new MyClass() {
         void differentMethod(int c, int d) {
             a = c + d; // <--- javac will catch for us
         }
     }
}
```

Therefore, when we encounter a method like we have here, we can disregard the param given to us and traverse into the body of the method with all of the non-final params from just this method.

It's tempting to not traverse if all of the params are final or it has no params but there could be something in the body like an inner class that has an assignment to one of its own params.

## ServiceLoader Configuration

The compiler is loaded via the java.util.ServiceLoader pattern so you need to have the fully qualified name of your plugin in `META-INF/services/com.sun.source.util.Plugin`.

# Maven Config Example

The [maven compiler plugin](https://maven.apache.org/plugins/maven-compiler-plugin/examples/pass-compiler-arguments.html) supports arguments to javac. The plugin's name needs to be passed as an arg as shown below. 

```xml

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.6.1</version>
    <configuration>
        <compilerArgs>
            <arg>-Xplugin:EffectivelyFinal</arg>
        </compilerArgs>
        <forceJavacCompilerUse>true</forceJavacCompilerUse>                
        </configuration>
    <dependencies>
        <dependency>
            <groupId>com.massfords</groupId>
            <artifactId>effectively-final</artifactId>
            <version>1.0</version>
        </dependency>
    </dependencies>
</plugin>

```