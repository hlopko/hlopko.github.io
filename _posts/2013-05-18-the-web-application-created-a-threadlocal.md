---
layout: post
title: "The web application created a ThreadLocal"
description: ""
date: 2013-05-18
---

Few months ago I spent some time finding and resolving
memory leaks in our Java web applications. The topic itself
deserves a dedicated post, but today I wanted to talk about
a very specific source of memory leak often present in Java
web applications - **ThreadLocal** variables.

The problem itself is already covered by others. Particularly
useful resource is [Tomcat Wiki](http://wiki.apache.org/tomcat/MemoryLeakProtection).
In short, each connection is handled by a thread from a
thread pool. These threads are reused after the work
is done. When an application sets a thread local
variable, and fails to remove it, the variable will
remain set until the thread dies. And in the case of
threads from thread pool, it can take some time,
usually until the server is shut down.

Why is this a source of memory leaks? When an application
is undeployed, we want all its instances and classes to be
garbage collected. All application classes are loaded
by a web app class loader. A Java class is collected, when
its loading class loader is collected. And a class loader is
collected, when all its classes can be collected.

If a thread local variable references a class, it prohibits
the collection of the class and transitively over its class
loader the collection of every class used by the application
(not loaded by the application server itself by its
parent class loader).

Our particular troublemaker was the [JasperReports](http://community.jaspersoft.com/project/jasperreports-library) library
used for PDF generation.

The solution was quite simple. If the JasperReports creates
thread local variable and fails to remove it, we have
to execute all JasperReports related code within another
thread. This can be done using many ways, e.g. [Java Futures](http://docs.oracle.com/javase/6/docs/api/java/util/concurrent/Future.html).
For our simple use case we used our own solution, `BlockingThreadDelegator`. You could see parts of it in the closures post.
Here is its full source with javadocs removed for
shortness.

{% highlight java %}
public abstract class BlockingThreadDelegator<T> {

    public T doInNewThread() throws Exception {
        final Closure<T> result = new Closure<T>();
        final Closure<Exception> thrownException =
            new Closure<Exception>();

        Thread t = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    result.setValue(doWork());
                } catch (Exception t) {
                    thrownException.setValue(t);
                }
            }
        });
        t.start();
        t.join();

        if (thrownException.isNull()) {
            return result.getValue();
        } else {
            throw thrownException.getValue();
        }
    }

    public abstract T doWork() throws Exception;
}
{% endhighlight %}

All our JasperReports code had to be wrapped by
this delegator.

{% highlight java %}
    new BlockingThreadDelegator<InputStream>() {
        @Override
        public InputStream doWork() throws Exception {
            return generateInvoice();
        }
    }.doInNewThread();
{% endhighlight %}
