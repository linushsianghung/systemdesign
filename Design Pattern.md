# Design Pattern
## Concepts
- [Dependency Injection](https://www.youtube.com/watch?v=QtDTfn8YxXg)
### SOLID Principles
- [The SOLID Principles of Object-Oriented Programming Explained in Plain English](https://www.freecodecamp.org/news/solid-principles-explained-in-plain-english/)
- [SOLID Design Patterns](https://www.youtube.com/watch?v=agkWYPUcLpg)

## Singleton
- [Java Singleton Design Pattern Best Practices with Examples](https://www.digitalocean.com/community/tutorials/java-singleton-design-pattern-best-practices-examples)
- [Singletons: Bill Pugh Solution or Enum](https://dzone.com/articles/singleton-bill-pugh-solution-or-enum)
### Implementation
Common Concepts:
- Private constructor to restrict instantiation of the class from other classes.
- Private static variable of the same class that is the only instance of the class.
- Public static method that returns the instance of the class, this is the global access point for the outer world to get the instance of the singleton class.

```java

public class EagerInitializedSingleton {}
public class StaticBlockSingleton {}

public class LazyInitializedSingleton {

    private static LazyInitializedSingleton instance;

    private LazyInitializedSingleton(){}

    // It provides thread-safety, but reducing the performance because of the cost of the synchronized method which only 
    // be required for the first few threads that might create separate instances
    public static LazyInitializedSingleton getInstance() {
        if (instance == null) {
            instance = new LazyInitializedSingleton();
        }
        return instance;
    }

    // Using Double-Checked Locking Principle
    public static LazyInitializedSingleton getInstanceUsingDoubleLocking() {
        if (instance == null) {
            synchronized (LazyInitializedSingleton.class) {
                if (instance == null) {
                    instance = new LazyInitializedSingleton();
                }
            }
        }
        return instance;
    }
}

public class BillPughSingleton {

    private BillPughSingleton(){}

    // The ClassLoader loads this static inner class SingletonHelper into JVM only once and is not loaded into memory 
    // until its getInstance() method is called.
    private static class SingletonHelper {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }

    public static BillPughSingleton getInstance() {
        return SingletonHelper.INSTANCE;
    }
}

/*** All the previous singleton implementation approaches can be destroyed by Reflection. ***/

// Java ensures that any Enum value is instantiated only once in JVM but the obvious drawback is that we cannot have 
// lazy loading in Enum lazy loading in Enum. 
// Although it overcomes the situation of Reflection, it still can be broken when using Serialization in distributed systems
public enum EnumSingleton {

    INSTANCE;

    public static void doSomething() {
        // do something
    }

    // Providing the implementation of readResolve() method to avoid the serialization issue
    protected Object readResolve() {
        return getInstance();
    }
}
```