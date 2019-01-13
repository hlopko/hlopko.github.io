---
layout: post
title: "Inline Subclass in Java"
description: ""
category: Java
tags: []
---

There are 3 languages I program in daily. Well, 4, but my
javascript code is not something I should publicly talk
about :) So there is Java, Smalltalk, and Ruby. Both
Smalltalk and Ruby are dynamic languages heavily using
blocks. Java doesn't have blocks. But luckily Java allows
you to create anonymous inline subclass easily (althrough it
looks ugly and verbose, but we're in Java so what do we
want).

The first example shows how iteration is done in Smalltalk
and Ruby. Instead of using `for` or `while` cycles, a method
is created. The method takes an argument, a command-like
object, which is applied to every element of the collection.
Althrough we're more used to **iterators** in Java, there
may be situations, where this approach fits the domain needs
more.

{% highlight java %}
public class Orders extends ArrayList<Order> {

    public void allOrdersDo(ForEach<Order> cmd) {
        for (Order order : this) {
            cmd.process(order);
        }
    }

    public static void main(String[] args) {
        Orders orders = new Orders();
        orders.allOrdersDo(new ForEach<Order>() {

            @Override
            public void process(Order order) {
                System.out.println(order);
            }
        });
    }
}

{% endhighlight %}

The Java syntax makes anonymous subclasses used less often
than they should be.  Following example shows how you can
remove duplication when handling exceptions, logging
information, and pretty much everytime something has to
be done before and/or after the arbitrary behavior.

{% highlight java %}
public class PaymentsService {

    private PaymentsWebService webService;

    public void createPayment(final Payment payment) {
        withHandledWSExceptionDo(new ThrowsWSException() {

            @Override
            public void execute() throws WSException {
                webService.createPayment(payment);
            }
        });
    }

    private void withHandledWSExceptionDo(ThrowsWS cmd) {
        try {
            cmd.execute();
        } catch (WSException e) {
            log.error(e);
        }
    }
}
{% endhighlight %}

The last example goes a few steps further and is inspired by
the amazing book
[Growing Object-Oriented Software Guided by Tests](http://www.growing-object-oriented-software.com/). 
Imagine we have builders used to create entities in tests.
Using inline subclass, we can override `toString()` method
of the entity in order to have a better test failure
message. Read the book if it sounds weird :)


{% highlight java %}
public class UserBuilder {
    public static UserBuilder anUser(final String name) {
        User user = new User() {
            @Override
            public String toString() {
                return name;
            }
        };
        return new UserBuilder(user);
    }
    
    private User user;

    private UserBuilder(User user) {
        this.user = user;
    }

    public User build() {
        return user;
    }
    //...
}
{% endhighlight %}

{% highlight java %}
public class FooTest {
    //...
    @Test
    public void testFoo() {
        User notSubscribedUser = 
                anUser("not subscribed user")
                    .notSubscribed()
                    .build();
        User subscribedUser = 
                anUser("subscribed user")
                    .subscribed()
                    .build();
        db.store(notSubscribedUser, subscribedUser);

        List<User> users = service.findAllSubscribedUsers();

        assertEquals(subscribedUser, users.get(0));
    }
}
{% endhighlight %}

And the failure message will read:

    expected:<subscribed user> but was:<not subscribed user>


