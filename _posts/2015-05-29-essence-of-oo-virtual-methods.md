---
layout: post
title: "Essence of OO: Virtual Methods"
description: ""
category:
tags: []
date: 2015-05-29
---

In this post I would like to show the most basic building block of
Object-Oriented programming - virtual methods. I know 99% of programmers
reading this know perfectly what virtual methods are and how to work
with them. This post is for the remaining 1% - many students of the
courses I teach belong to this category, and I hope this post will help
them build solid ground for the more advanced topic.

I non-OO languages, when you see a function application, you always know
which function will be executed. In C language, the program might
look like this:

```c
#include<stdio.h>

void sayHello() {
  printf("Hello!");
}

int main() {
  sayHello();
}
```

When you read the body of `main` function, you see calling of `sayHello`
function. You can be pretty sure the function being executed at runtime
is `sayHello` defined above `main`.

In OO languages, when you send a message to an object, you are not
always sure which method will be executed. Sure, in simple cases, such as
the following snippet written in Java, you *know*:

```java
class User {
  public void initialize() {
    System.out.println("Initializing user");
  }
}

public class Main {
  public static void main(String[] args) {
    User user = new User();
    user.initialize();
  }
}
```

The method `initialize` called from method `main` will surely print
"initializing user", the program will execute `initialize` method
defined on the `User` class.

No big surprises so far. Now let's change the code a bit. Imagine we
have a following code using User class defined in the previous snippet:

```java
public void callInitializeOnUser(User user) {
  user.initialize();
}
```

Do we know which method will be called? If there is no more code than
what we see, we do. But what if someone subclasses `User` class, and
overrides `initialize` method?

```java
class StubbornUser extends User {
  @Override
  public void initialize() {
    System.out.println("I don't want to be initialized");
  }
}
```

What will the following code print?

```java
public class Main {
  public static void main(String[] args) {
    User user = new User();
    User stubborn = new StubbornUser();

    callInitializeOnUser(user);
    callInitializeOnUser(stubborn);
  }
}
```

It will print:

```
Initializing user
I don't want to be initialized
```

And what we just saw is the virtual method in action. **In OO languages,
all messages sent to an object are looked up at runtime,
and which method will be executed depends on the runtime type of the
object.** When at runtime the stubborn user is passed in, the virtual
machine will call the initialize method of the stubborn user. When at
runtime the normal user is passed in, the virtual machine will call the
initialize method of the normal user.

The whole point of virtual methods is to give programs a little more
freedom, they don't have to know exactly which method to execute at the
compile time. It is enough they can decide at runtime. And as a result,
the programs can be reused more easily.

The simplest design pattern, which makes use of virtual methods is
`Template Method`. In short, we will create a superclass with a template
method. In this template method, we will call one or more methods. When
we subclass this superclass, we can override some of the methods to
modify the behavior of the template method, and at the same time we can
reuse all the existing logic.

```java
class EmailProcessor {
  // ...
  public void processEmails(ImapAccount imapAccount, Date date) {
    connectTo(imapAccount);

    List<EmailMessage> messages = downloadNewMessages(folder, date);
    for (EmailMessage message : messages) {
      processMessage(message);
    }

    disconnectFrom(imapAccount);
  }
  // ...
}
```

We see a code snippet from an desktop email client. It's purpose is to
download new emails from the imap server. The logic of `connectTo` and
`disconnectFrom` is very complicated and we are not interested in it.
Also, the `downloadNewMessages` is non-trivial and we don't want to
understand it.

Thanks to virtual methods and `template method` pattern, we can just
subclass `EmailProcessor` and override `processMessage` with our custom
logic:

```java
class LoggingEmailProcessor extends EmailProcessor {
  private Log log = LogFactory.getLog(getClass);

  protected void processMessage(EmailMessage message) {
    log.trace(message.toString);
  }
}
```

Now we can use this class in our code like this:

```java
new LoggingEmailProcessor().processEmails(folder, date);
```

Now to the final point of the post - the method lookup. The process of
looking up the method at runtime is simple:

1. take the object receiving the message (let's call it receiver)
2. look into receiver's class
3. try to find the method with the correct name
4. if you found such method, execute it and finish the lookup
5. if you didn't find the method, look into parent class of the current
   class and jump to step 3.

There is nothing more to it, it's that simple. Just look at what methods
are defined for each class in the parent hierarchy, and execute the
first one that matches.

Now it is your turn - try to explain what happens when executing the
last code snippet. Let this image help you :)

![Sequence diagram for the last code snippet](/images/virtual_methods_seq_diagram.png)





