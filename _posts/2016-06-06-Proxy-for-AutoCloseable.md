---
published: true
layout: post
---
I was recently reminded of a [SO answer](http://stackoverflow.com/questions/8431755/can-java-ee-7s-core-interfaces-entitymanager-extend-autoclosable/18360526#18360526) I posted a while ago in response to a user asking about making Java 7's JPA core interfaces extend AutoCloseable. I thought I'd post a little bit of the code here for future reference.

## Motivation

The reason for wanting an AutoCloseable version of EntityManager is that you have one or more spots in your system where the injected EntityManager via the @PersistenceContext annotation isn't meeting your needs. If this doesn't apply to you, then you can stop reading here. I don't often find a need for the AuotCloseableEntityManager myself, but it was a fun exercise building one and I've used this pattern elsewhere with the Proxy class for other interfaces I needed to enhance.

## Intro to java.lang.reflect.Proxy

[Proxy](http://docs.oracle.com/javase/7/docs/api/java/lang/reflect/Proxy.html) class was introduced in Java 1.3 and is very useful for creating an implementation of an interface on the fly. If you find yourself needing to modify a few methods for an existing interface implementation or perhaps adapt an existing one like EntityManager to AutoCloseable, then this is your class.

First, a quick look at how we did this in the old days before Proxy was added to the language. Back then, if you wanted to change the behavior of an implementation or add new behavior, then you'd wrap your existing implementation in an wrapper class and delegate all of the calls to this delegate except the subset that you wanted to intercept and handle differently.

### Life Before Proxy

Our implementation for an AutoCloseableEntityManager in the old days would look like this:

```java
public class PrehistoricAutoCloseableEntityManagerImpl implements AutoCloseable, EntityManager {
    private final EntityManager em;

    public PrehistoricAutoCloseableEntityManagerImpl(EntityManager em) {
            this.em = em;
    }
        
    // ... implement all of the EM methods here as pass thru's to the em instance 
    // ... there are 30+ methods so it's a lot of rote code
        
    /**
     * Both EntityManager and AutoCloseable define a close method. Showing this
     * as an example of what all of the other delegating methods look like
     */
    @Override
    public void close() {
        em.close();
    }
}
```
    
This is a straightforward approach but it's tedious since in many cases there's a lot of code you need to write to implement the pass through. It's also something that you need to do at design time whereas you can create a proxy instance at runtime using the method below.

## Example in your code

The details for creating a Proxy below look scary, but here's a sneakpeek at the payoff...

```java
public MyManualJPAClass {
    private final AutoCloseableEntityManagerFactory emf;
        
    public MyManualJPAClass(EntityManagerFactory emf) throws Exception {
        this.emf = wrap(emf);
    }
    
    public void doSomething() throws Exception {
        try(AutoCloseableEntityManager em = emf.createEntityManager()) {
           EntityTransaction tx = em.getTransaction();
           tx.begin();
            
           // do a bunch of JPA stuff with your em
               
           tx.commit();
        }
    }
}
```


### Life After Proxy


    
#### Step 1: Create the AutoCloseableEntityManager

Create a new interface that extends both EntityManager and AutoCloseable so we can use the EntityManager within a try-with-resources block.

```java
public interface AutoCloseableEntityManager 
    extends EntityManager, AutoCloseable {
}
```

#### Step 2: Create an AutoCloseableEntityManagerFactory

We need this interface so the EntityManager type returned to us from the EntityManagerFactory will be AutoCloseable. The two methods in this signature are the same as defined in the EntityManagerFactory except they return the AutoCloseable instance. This is still compatible with the EntityManagerFactory interface and Java will let you use the @Override annotation since the AutoCloseableEntityManager is of type EntityManager 

```java
public interface AutoCloseableEntityManagerFactory 
    extends EntityManagerFactory {
    @Override
    AutoCloseableEntityManager createEntityManager();
    @Override
    AutoCloseableEntityManager createEntityManager(Map args);
}
```

#### Step 3: Wrap an intance of EntityManagerFactory to return AutoCloseableEntityManagerFactory

We need to use the Proxy.newProxyInstance method to construct our proxy. The arguments to this method are as follows:

- ClassLoader: used to define the proxy class
- Class[]: list of interfaces we want the proxy to implement
- InvocationHandler: handles all of the method invocations

Typically, passing the ClassLoader from your calling class is sufficient. This assumes that whatever ClassLoader loaded your class is sufficient to be the ClassLoader for your new Proxy instance:

```java
MyClass.class.getClassLoader()
```
   
The list of interfaces depends on the type of Proxy you're creating. In our case, we only need to pass a single interface here since we've already defined an interface that joins both parts of the problem we're trying to solve: an EntityManager that is AutoCloseable. Having this defined in a single interface is convenient.

```java
new Class[] {AutoCloseableEntityManager.class}
```
    
The InvocationHandler is the class that the proxy object will invoke every time someone invokes a method on the proxy. The general pattern here is to look for a method call that you want to handle directly and then do so or else just delegate the call to the object you're proxying. This is pretty much the same approach as we did with the wrapper class above but this is done at runtime instead of us writing lots of delegate calls.

The pattern I typically follow is to have a wrap(x) method where x is the type I want to wrap in a proxy and then I can define the new proxy behavior in this wrap method with an anonymous class implementation of the InvocationHandler.

Here's what it looks like in outline form:

```java
AutoCloseableEntityManagerFactory wrap(EntityManagerFactory emf) {
    return (AutoCloseableEntityManagerFactory) Proxy.newInstance(
        myClassLoader,
        new Class[] {AutoCloseableEntityManagerFactory.class},
        new InvocationHandler() {
            public Object invoke(Object proxy, Method method, Object[] args) {
                // for each method call, is it one I want to intercept?
                // if not, delegate the call with reflection
            }
        });
}
```

### Details on the InvocationHandler

The InvocationHandler created gets a shot at every method invoked. If it's one you want to override, then do something. If not, then use reflection to invoke the method on the object you're proxying which in this case is in scope as the argument to our method (emf).

In our case, we only want to override the create methods so we can return a wrapped version of the EntityManager from the underlying EntityManagerFactory so it'll be AutoCloseable for us.

There's a bit more complexity here with the reflection since we want to intercept methods to the createEntityManager calls but use the methods that are overridden which provide the AutoCloseable variant of the EntityManager.

> A quick note on the isCreateEntityManager checks below. Ideally, we wouldn't need any of 
> this code. We'd delegate the call to the proxied object, check to see if it's a 
> EntityManager, and then return a wrapped version of the EntityManager. Unfortunately, the 
> introduction of the AutoCloseableEntityManagerFactory above overrides the create calls and 
> thus we can't use the method we're passed in the invoke method to make the call but 
> instead need to figure out which method they're calling and use the appropriate method 
> from our new interface instead.

```java

@Override
public Object invoke(Object proxy, Method method, Object[] args) 
throws Throwable {
    Object retVal = null;
                
    // using the static Method fields here since they're from the
    // AutoCloseableEntityManagerFactory which is likely different
    // than the basic method arg passed in here

    if (isCreateEntityManager(method, args)) {
        retVal = createEntityManager.invoke(emf, args);
    } else if (isCreateEntityManagerMap(method, args)) {
        retVal = createEntityManagerMap.invoke(emf, args);
    }
    if (retVal != null) {
        return wrap((EntityManager)retVal);
    }
    try {
        return method.invoke(emf, args);
    } catch(InvocationTargetException e) {
        // Make sure you throw the cause of the excpetion and not e itself. 
        // You want the exception from the target, not the one wrapped by
        // InvocationTargetException
        throw e.getCause();
    }
}

private boolean isCreateEntityManagerMap(Method method, Object[] args) {
    return method.getName().equals(createEntityManagerMap.getName()) && 
        (args != null && args.length==1);
}

private boolean isCreateEntityManager(Method method, Object[] args) {
    return method.getName().equals(createEntityManager.getName()) && 
        (args == null || args.length==0);
}
            
private static final Method createEntityManager;
private static final Method createEntityManagerMap;
static {
    try {
        createEntityManager =
            EntityManagerFactory.class.getMethod("createEntityManager");
                createEntityManagerMap =
                    EntityManagerFactory.class.getMethod("createEntityManager", Map.class);
        } catch (NoSuchMethodException e) {
            throw new RuntimeException(e);
        }
}
            
public static AutoCloseableEntityManager wrap(final EntityManager em) {
    return (AutoCloseableEntityManager)
        Proxy.newProxyInstance(myClassLoader,
                new Class[]{AutoCloseableEntityManager.class}, 
                (proxy, method, args) -> {
                    if (method.getName().equals("close")) {
                        em.close();
                        return Void.TYPE;
                    }
                    try {
                        return method.invoke(em, args);
                    } catch (InvocationTargetException e) {
                        throw e.getCause();
                    }
    });
}
```
            
            
If the caller is attempting to create an EntityManager from the factory, then we want to intercept those calls and create one that we'll then wrap and return as an AutoCloseableEntityManager.
