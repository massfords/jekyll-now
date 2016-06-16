---
published: false
---
Making a simple comma separated list of values is a lot simpler in Java 1.8. I used to do this with a few lines of code in a for loop (pre-dates for-each). 

```java
String result = "";
String delim = "";
for(int i=0; i<list.size(); i++) {
   result += delim;
   delim = ",";
   result += list.get(i);
}
```
I picked up the above technique years ago when I was first learning Java. Apache `commons-lang` provides a utility method for this in StringUtils but now this is baked into Java's String class:

```java
String result = String.join(",", List<String> list);
```

