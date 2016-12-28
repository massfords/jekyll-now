---
published: true
layout: post
---
Please stop using XmlGregorianCalendar as the default type to model `xs:date` or `xs:dateTime` in a JAXB class. If you're on Java 8 or later then you should be using the new `java.time` classes like `LocalDate`, `LocalDateTime`, and `ZonedDateTime`. If you're on an earlier version of Java and are unable or unwilling to upgrade then you can map the `xs:date` and `xs:datetime` types to `joda-time` classes. 

## Why you shouldn't use XmlGregorianCalendar
This type lacks the semantics of what the underlying data type really is. The biggest distinction is whether the value is an `xs:date` or an `xs:dateTime`. A second distinction is whether the value is a local date/time versus one tied to a specific timezone offset. 

The `@XmlSchemaType` attribute conveys whether the underlying schema type is an `xs:date` or `xs:dateTime` but this isn't enforced in the generated code and in fact some popular frameworks like [Jackson](http://wiki.fasterxml.com/JacksonJAXBAnnotations) simply ignore the `@XmlSchemaType` annotation and marshal the value as a full ISO 8601 date and time which isn't great when you want a simple yyyy-mm-dd value.

Consider something like a person's birthday. This is probably best modeled as a `LocalDate`. We don't need a specific timezone offset nor do we need a specific time. **If someone asks you're birthday, you're unlikely to provide a time, it's just the date.**

It's also possible that our data model could have a `xs:dateTime` that's not tied to a specific timezone. Maybe it's a restaurant chain's offer that has drink or food specials on a specifc date from 4pm - 6pm. The fields for this object wouldn't need to include a timezone. We only want the local date and time.

Finally, in some cases we want the full date, time, and timezone. This could be an `XmlGregorianCalendar` but given that the new Java 8 types are immuatable then it makes sense to use these immutable types as opposed to the default Xml types.

## jaxws-maven-plugin

Here's an example with the `jaxws-maven-plugin`.

```xml
<plugin>
    <groupId>org.jvnet.jax-ws-commons</groupId>
    <artifactId>jaxws-maven-plugin</artifactId>
    <version>2.1</version>
    <executions>
        <execution>
            <goals>
                <goal>wsimport</goal>
            </goals>
            <configuration>
                <bindingDirectory>src/jaxb</bindingDirectory>
            </configuration>
        </execution>
    </executions>
</plugin>
```

## Bindings file

The jaxb-bindings file is in the bindings directory which in this example is `src/jaxb` as configured in the plugin above

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<jaxb:bindings
        xmlns:jaxb="http://java.sun.com/xml/ns/jaxb" xmlns:xs="http://www.w3.org/2001/XMLSchema"
        xmlns:xjc="http://java.sun.com/xml/ns/jaxb/xjc"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://java.sun.com/xml/ns/jaxb http://java.sun.com/xml/ns/jaxb/bindingschema_2_0.xsd"
        version="2.1">

    <jaxb:globalBindings>
        <xjc:javaType adapter="com.massfords.LocalDateAdapter"
                      name="java.time.LocalDate" xmlType="xs:date"/>
    </jaxb:globalBindings>

</jaxb:bindings>
```

## JAXB Adapter

The xml adapter is a trivial amount of code. You can follow this pattern for LocalDateTime and ZonedDateTime since both have good parse and toString methods.

```java
package com.massfords;

import javax.xml.bind.annotation.adapters.XmlAdapter;
import java.time.LocalDate;

public class LocalDateAdapter extends XmlAdapter<String,LocalDate> {
    @Override
    public LocalDate unmarshal(String v) throws Exception {
        return LocalDate.parse(v);
    }

    @Override
    public String marshal(LocalDate v) throws Exception {
        if (v != null) {
            return v.toString();
        } else {
            return null;
        }
    }
}
```
## What about other types?

You can follow the above templates and add a mapping and adapter for `xs:dateTime` to map them to a `LocalDateTime`. This is simply a copy/paste/replace `LocalDateTime` for 
`LocalDate`. However, if your application has actual `xs:dateTime` which refer to a specific instant in time that requires a timezone then my suggestion is to explicitly model this as its own extension of `xs:dateTime` which will allow you to add a mapping targeted at just these occurrences. The value of having an explicit type is that you'll always know when an `xs:dateTime` refers to a dateTime with a timezone offset as opposed to a timezone free `xs:dateTime`.

