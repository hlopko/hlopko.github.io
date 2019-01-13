---
layout: post
title: "Closures in Java"
description: ""
category:
tags: []
date: 2013-05-17
---

So right now we don't have real
[closures](http://en.wikipedia.org/wiki/Closure_(computer_science)) in
Java. We can access
variables from the surrounding context, but only if they are
final - they cannot be assigned a new value. This way
Java can simply copy the reference stored in the variable
to the newly created variable in inner context and use
only the newly created variable. Let me show you an example.
Imagine we need to do something in a new thread, let's call
it **worker**.

{% highlight java %}
    public T doInNewThread(final PieceOfWork work) {
        Thread t = new Thread(new Runnable() {
            @Override
            public void run() {
                work.doWork();
            }
        });
        t.start();
    }
{% endhighlight %}

All exceptions thrown in `doWork()` method are handled in
the context of the worker and *invisible* to the master
thread.  Now imagine the requirement is to rethrow all
exceptions thrown in the worker within the master thread. We
cannot use local variable, but we can use a setter.

{% highlight java %}
    public T doInNewThread(PieceOfWork work) {
        Thread t = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    work.doWork();
                } catch (RuntimeException e) {
                    setException(e);
                }
            }
        });
        t.start();

        if (exceptionThrow()) {
            throw getException();
        }
    }
{% endhighlight %}

This solves the problem, but introduces a field and at
least one method. What if we want to solve the problem
in the context of the `doInNewThread` method? Closures
to the rescue.

There is an old trick which I happily used for a while.
The idea is that while we cannot assign into the original
variable, we can change the type of the variable to an array.
The array instance will be final, but the content can be
changed.

{% highlight java %}
    public T doInNewThread(PieceOfWork work) {
        final RuntimeException[] result =
            new RuntimeException[1];

        Thread t = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    work.doWork();
                } catch (RuntimeException e) {
                    result[0] = e;
                }
            }
        });
        t.start();

        if (result[0] != null) {
            throw result[0];
        }
    }
{% endhighlight %}

This is pretty much how some closure compilers (e.g.
the one used by [Cog VM](http://www.mirandabanda.org/cog/)
and many others) work
behind the scenes (with a smart detection when a wrapper
is really needed and when a reference copy will do just
ok). The reason why I stopped doing this is that my
colleagues always stumbled upon the code and needed an
explanation. And there were right, the code is not
very intention revealing.

The improvement is this small class I put into our project
magically called `Closure` (which althrough magical is
not very accurate name).

{% highlight java %}
public class Closure<T> {

    private T value;
    private boolean valueSet = false;

    public void setValue(T value) {
        this.value = value;
        this.valueSet = true;
    }

    public T getValue() {
        return value;
    }

    public boolean isSet() {
        return valueSet;
    }
}
{% endhighlight %}

Using this class, our previous example can be refactored
into something a little bit nicer.

{% highlight java %}
    public T doInNewThread() {
        final Closure<RuntimeException> thrownException =
            new Closure<RuntimeException>();

        Thread t = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    work.doWork();
                } catch (RuntimeException e) {
                    thrownException.setValue(e);
                }
            }
        });
        t.start();

        if (thrownException.isSet()) {
            throw thrownException.getValue();
        }
    }
{% endhighlight %}
