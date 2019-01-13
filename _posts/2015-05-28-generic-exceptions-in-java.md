---
layout: post
title: "Generic Exceptions in Java"
description: ""
category:
tags: []
date: 2015-05-28
---

Quick post today. Skip the post completely if you are using Java 8.

Very often, when programming in Java, I miss blocks
(lambdas, anonymous functions, whatever your language calls them). We
can write this simple interface to fulfill this wish.

```java
public interface Executable {
  void execute();
}
```

I used it to create methods like this one:

```java
public static void whileOptimisticLockingExceptionOccursDo(Executable executable) {
  int counter = 0;
  while (true) {
    try {
      executable.execute();
    } catch (HibernateOptimisticLockingFailureException e) {
      counter++;
      if (counter < 5) {
        log.trace(
          format(
            "HibernateOptimisticLockingFailureException thrown, retrying for %s time", counter));
      }
      if (counter == 5) {
        log.error("HibernateOptimisticLockingFailureException happened 5 times in a row!", e);

        throw e;
      }
    }
  }
}
```

And when calling them method, it looked like:

```java
private void confirmPayment(final PaymentInfo paymentInfo) {
  whileOptimisticLockingExceptionOccursDo(new Executable() {
    @Override
    public void execute() {
      service.refreshEntity(paymentInfo);

      paymentInfo.makePaid(currentDateProvider.getCurrentDate());
    }
  });
```

This all kind of works, but it's not reusable much. Sometimes, we need
executable block to return value. Generics to the rescue:

```java
public interface Executable<T> {
  T execute();
}
```

And this is what I used for a long time. But recently I needed the
execute method to throw checked exceptions. Long story short, it appears
you can use generics for throws clause in Java:

```java
public interface Executable<T, X extends Exception> {
  T execute() throws X;
}
```

and you can use it like this:

```java
foo.doUntilTrueOrCrash(new Executable<Boolean, SomeException>() {
    @Override
    public Boolean execute() throws SomeException {
      // ...
    }
  });
```

And when `SomeException` is `RuntimeException`, you do not have to add
throws clause at all.

