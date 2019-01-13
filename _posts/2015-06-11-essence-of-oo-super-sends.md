---
layout: single
title: "Essence of OO: Super Sends"
description: ""
category:
tags: []
date: 2015-06-11
---

In the last post I talked about what virtual methods are,
how the method lookup works, and how we can use them.
In this post, we will talk about something a bit more advanced,
super sends.

Super sends exist, because sometimes, you want to call a method from
superclass, not your overriden one. This is commonly used when you need
the behavior of both your specialized method and superclass method. Say
we work on a web application and we want to log how long did it take to
handle request (apologies to the OO design sensitive readers for this
contrived example).

```java
public class ProfilingHttpRequestHandler extends HttpRequestHandler {
  public HttpServletResponse handleRequest(HttpServletRequest request) {
    long start = System.currentTimeMillis();
    HttpServletResponse response = super.handleRequest(request);
    long end = System.currentTimeMillis();

    log.trace(String.format("Request took %l ms to handle", end - start));

    return response;
  }
}
```

In the last post, we talked about method lookup which works as follows:

1. take the object receiving the message (let's call it receiver)
2. look into receiver's class
3. try to find the method with the correct name
4. if you found such method, execute it and finish the lookup
5. if you didn't find the method, look into parent class of the current
   class and jump to step 3.

And now to the essence of super sends. All super send is doing, is that
it starts the method lookup in the superclass, not in the current class.
Here is a nice puzzle for you:

```java
class A {
  public int myGetClass() { return 1; }
}

class B extends A {
  public int myGetClass() { return 2; }

  public boolean testMyGetClass() {
    return myGetClass() == super.myGetClass();
  }

  public boolean testGetClass() {
    return getClass() == super.getClass();
  }
}

public class Main {
  public static void main(String[] args) {
    B b = new B();

    System.out.println(b.testMyGetClass());
    System.out.println(b.testGetClass());
  }
}
```

Try to tell the result without reading on, just for the fun.

First, the program prints `false`. It calls myGetClass() implemented in
B, which returns 2, and compares it to super.myGetClass(), which returns
1.

More interesting is the second one. Most of the masters degree students
in CS fail to tell the correct result. The program prints `true`. The
method `getClass` is defined on `Object` class. When we execute
`getClasss()`, the method lookup traverses the class **B**, doesn't find
the method, traverses class **A**, doesn't fint the method, and finally
traverses class **Object** and finds the method `getClass`. When we
execute `super.getClass()`, super causes the method lookup to start in
the superclass, therefvore traverses class **A**, doesn't find the
method, and traverses **Object** and finds the method. In both cases,
the method is the same, so is its return value (`B.class`), so the
result must be `true`.

So next time you will be challenged with super sends, you will know,
that all the super send is doing, is starting the method lookup in the
superclass, nothing more, nothing less. Happy sending! :)



