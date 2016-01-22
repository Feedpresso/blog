---
layout: post
title:  Why you should use RxJava in Android a short introduction to RxJava
date:   2016-01-22
author: Tadas Šubonis
tags:
- RxJava
- Android
- introduction
- tutorial
---


In most of the Android applications, you are reacting to user interactions
(clicks, swipes and etc.) and you will be doing something in the background
(networking).

Orchestrating all of this is a hard thing and could quickly become unmanageable
code mess.


For example, it isn't so easy, to send a request to a database over network
and after that completes start fetching his messages and his preferences
at the same time, and after all of that is complete - show a welcome message.

This is a case where [RxJava](https://github.com/ReactiveX/RxJava) (ReactiveX) excels - orchestrating multiple
actions that happen due to certain events in the system.

Using RxJava you will be able to forget Callbacks and hellish global
state management.

## Quick start in Android studio
To get libraries that you
are most likely to need in your project, in your *build.gradle* insert
these lines:

```
compile 'io.reactivex:rxjava:1.1.0'
compile 'io.reactivex:rxjava-async-util:0.21.0'

compile 'io.reactivex:rxandroid:1.1.0'

compile 'com.jakewharton.rxbinding:rxbinding:0.3.0'

compile 'com.trello:rxlifecycle:0.4.0'
compile 'com.trello:rxlifecycle-components:0.4.0'
```

These will include:

 * [RxJava](https://github.com/ReactiveX/RxJava) - a core ReactiveX library
 for Java.
 * [RxAndroid](https://github.com/ReactiveX/RxAndroid) - RxJava extensions
 for Android that will help you with Android threading and Loopers.
 * [RxBinding](https://github.com/JakeWharton/RxBinding) - this will provide
 bindings between RxJava and Android UI elements likes Buttons and TextViews
 * [RxJavaAsyncUtil](https://github.com/ReactiveX/RxJavaAsyncUtil) - helps you
 to glue code between [Callable](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Callable.html)s and
  [Future](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Future.html)s.

## Example
I think, it's the best to start with an example:
```java
Observable.just("1", "2")
        .subscribe(new Action1<String>() {
            @Override
            public void call(String s) {
                System.out.println(s);
            }
        });
```

Here we create an  [Observable](http://reactivex.io/documentation/observable.html)
that will be emit two items *1* and *2*.

Afterwards, we subscribe to those actions and we receive an item
it will be printed out.

## Some details
*Observable* is something, that you can subscribe to to listen for the items
that the observable will emit. They can be constructed in many different
ways. However, they usually don't begin emitting items, until you subscribe
to them.

After you subscribe to an observable, you get a [Subscription](http://reactivex.io/RxJava/javadoc/rx/Subscription.html).
The subscription will listen for the items from observable until it
marks itself as completed or, otherwise, it will continue indefinitely
(very rare case).

Furthermore, all of these actions are going to be executed on the main
thread.

## Expanded example
```java
Observable.from(fetchHttpNetworkContentFuture())
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(new Action1<String>() {
            @Override
            public void call(String s) {
                System.out.println(s);
            }
        }, new Action1<Throwable>() {
            @Override
            public void call(Throwable throwable) {
                throwable.printStackTrace();
            }
        });
```

Here we can see a few new things:

1. ```.subscribeOn(Schedulers.io())``` - it will make the Observable to do its
waiting and computations on a ThreadPool that's dedicated for I/O (*Schedulers.io()*).
1. ```.observeOn(AndroidSchedulers.mainThread())``` - makes subscriber action
to execute its result on Android's main thread. This is needed if one wants
to change anything on Android UI.
1. A second argument to ```.subscribe()``` - introduces an error handler for
subscription in case something goes wrong. It's something that should be
present almost **always**.

## Managing complicated flow.

Remember the complicated flow, we described initially?

Here is how it would look with RxJava:

```java
Observable.fromCallable(createNewUser())
        .subscribeOn(Schedulers.io())
        .flatMap(new Func1<User, Observable<Pair<Settings, List<Message>>>>() {
            @Override
            public Observable<Pair<Settings, List<Message>>> call(User user) {
                return Observable.zip(
                        Observable.from(fetchUserSettings(user)),
                        Observable.from(fetchUserMessages(user))
                        , new Func2<Settings, List<Message>, Pair<Settings, List<Message>>>() {
                            @Override
                            public Pair<Settings, List<Message>> call(Settings settings, List<Message> messages) {
                                return Pair.create(settings, messages);
                            }
                        });
            }
        })
        .doOnNext(new Action1<Pair<Settings, List<Message>>>() {
            @Override
            public void call(Pair<Settings, List<Message>> pair) {
                System.out.println("Received settings" + pair.first);
            }
        })
        .flatMap(new Func1<Pair<Settings, List<Message>>, Observable<Message>>() {
            @Override
            public Observable<Message> call(Pair<Settings, List<Message>> settingsListPair) {
                return Observable.from(settingsListPair.second);
            }
        })
        .subscribe(new Action1<Message>() {
            @Override
            public void call(Message message) {
                System.out.println("New message " + message);
            }
        });
```

This will create a new user (```createNewUser()```) and *when*
it is created and result is returned, it continues to fetch
user messages (```fetchUserMessages()```) and user settings (```fetchUserSettings```)
*at the same time*. It will wait until both actions are completed
and will return a combined result (```Pair.create()```).

Keep in mind, that all of this is happening in the background on a separate
thread.

After that, it will print out settings that were received. Finally,
the list of messages is transformed into another observable that will start
emitting messages themselves instead of an entire list, and each of the
messages are printed.

## Functional approach
RxJava will be much easier if programmer is familiar with functional
programming concepts such as *map* and *zip*. Also, they both
share lots of similarities of how one would construct a generic
logic flow.

## How to create a custom observable?
If codes becomes heavily based on RxJava (as for example as
  [Feedpresso](http://www.feedpresso.com) is), you will find yourself
often in a position where you need to write custom observables so
they would fit your flow.

An example of that:

```java
public Observable<String> customObservable() {
    return rx.Observable.create(new rx.Observable.OnSubscribe<String>() {
        @Override
        public void call(final Subscriber<? super String> subscriber) {
            // Execute in a background
            Scheduler.Worker inner = Schedulers.io().createWorker();
            subscriber.add(inner);

            inner.schedule(new Action0() {

                @Override
                public void call() {
                    try {
                        String fancyText = getJson();
                        subscriber.onNext(fancyText);
                    } catch (Exception e) {
                        subscriber.onError(e);
                    } finally {
                      subscriber.onCompleted();
                    }
                }

            });
        }
    });
}
```

or a simpler version that doesn't enforce execution of action
on a specific thread:

```java
Observable<String> observable = Observable.create(
    new Observable.OnSubscribe<String>() {
        @Override
        public void call(Subscriber<? super String> subscriber) {
            subscriber.onNext("Hi");
            subscriber.onCompleted();
        }
    }
);
```

It is important to note three methods here:

1. ```onNext(v)``` - send a new value to a subscriber
1. ```onError(e)``` - notify observer about an error that has occurred
1. ```onCompleted()``` - let subscriber know that it should unsubscribe as
there won't be any more content from this observable.

Furthermore, it's might be handy to rely on RxJavaAsyncUtil.

## Integration with other libraries
As RxJava is becoming more and more popular and becoming de-facto way
of doing asynchronous programming on Android, more and more libraries
are providing deep integration or reliance on it.

To name a few:

* [Retrofit](https://github.com/square/retrofit) - "Type-safe HTTP client for Android and Java"
* [SqlBrite](https://github.com/square/sqlbrite) - "A lightweight wrapper around SQLiteOpenHelper which introduces reactive stream semantics to SQL operations."
* [StorIO](https://github.com/pushtorefresh/storio) - "Beautiful API for SQLiteDatabase and ContentResolver"

All of them, can make life a lot easier when working with HTTP requests
and databases.


## Interactive with Android UI
This intro, wouldn't be complete without an example, how to use
native Android UI elements.

```java
TextView finalText;
EditText editText;
Button button;
...

    RxView.clicks(button)
            .subscribe(new Action1<Void>() {
                @Override
                public void call(Void aVoid) {
                    System.out.println("Click");
                }
            });

    RxTextView.textChanges(editText)
            .subscribe(new Action1<CharSequence>() {
                @Override
                public void call(CharSequence charSequence) {
                    finalText.setText(charSequence);
                }
            });
...
```

Obviously, it's easy to just rely on ```setOnClickListener```
but RxBinding might suit your needs better in a long run as
it allows you to plugin UI into a general RxJava flow.

## Tips
Over a time, we have noticed a few things that should be followed while
working with RxJava.

### Always use error handler
Skipping error handler like here

```java
.subscribe(new Action1<Void>() {
    @Override
    public void call(Void aVoid) {
        System.out.println("Click");
    }
});
```

is highly advised to avoid. As an exception thrown in observer or
in one of the actions will most likely kill your entire application.

Having a generic handler, would be even better:

```java
.subscribe(..., myErrorHandler);
```

### Extract actions methods

Having lots of inner classes might not look so readable after a while
(especially if you are not using RetroLambda).

So a code like this:

```java
.doOnNext(new Action1<Pair<Settings, List<Message>>>() {
    @Override
    public void call(Pair<Settings, List<Message>> pair) {
        System.out.println("Received settings" + pair.first);
    }
})
```

could look a lot better if refactored into this:

```java
.doOnNext(logSettings())

@NonNull
private Action1<Pair<Settings, List<Message>>> logSettings() {
    return new Action1<Pair<Settings, List<Message>>>() {
        @Override
        public void call(Pair<Settings, List<Message>> pair) {
            System.out.println("Received settings" + pair.first);
        }
    };
}

```


### Use custom classes or tuples

There will be many occasions where one value depends on the first one
(User and user settings) and you would like to get both of them
using two asynchronous requests.

In such cases we would suggest using [JavaTuples](http://www.javatuples.org/) .

Example:

```java
Observable.fromCallable(createNewUser())
        .subscribeOn(Schedulers.io())
        .flatMap(new Func1<User, Observable<Pair<User, Settings>>>() {
            @Override
            public Observable<Pair<User, Settings>> call(final User user) {
                return Observable.from(fetchUserSettings(user))
                        .map(new Func1<Settings, Pair<User, Settings>>() {
                            @Override
                            public Pair<User, Settings> call(Settings o) {
                                return Pair.create(user, o);
                            }
                        });

            }
        });
```


### Lifecycle management
It is going to be often a case when a background process (subscription) is
going to survive longer than the Activity or Fragment that contains it. But
what if you don't care about the result if user leaves the actvity?

[RxLifecycle](https://github.com/trello/RxLifecycle) project will help you here.

Wrap your observable like this (taken from docs),
 it will unsubscribe when it is being
destroyed:

```java
public class MyActivity extends RxActivity {
    @Override
    public void onResume() {
        super.onResume();
        myObservable
            .compose(bindToLifecycle())
            .subscribe();
    }
}
```

## Conclusion
This is far from complete guide about RxJava usage on Android but hopefully
it gave a few arguments in favour of RxJava compared to regular AsyncTask and
guide
