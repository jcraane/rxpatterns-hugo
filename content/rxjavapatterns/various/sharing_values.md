---
title: "Sharing Values"
date: 2020-07-16T11:35:28+02:00
draft: false
---
A common pattern is sharing the result of an Observable between multiple subsribers, for example a network call. For demonstration purposes the System::nanoTime is emitted. By using the combination of

1. [share](http://reactivex.io/RxJava/1.x/javadoc/rx/Observable.html#share--) multicasts the original observer as long as there is at least one subscriber.
2. [replay](http://reactivex.io/RxJava/1.x/javadoc/rx/Observable.html#replay--) returns a ConnectableObservable that shares a single subscription to the underlying Observable that will replay all of its items and notifications to any future Observer.
3. [autoConnect](http://reactivex.io/RxJava/1.x/javadoc/rx/observables/ConnectableObservable.html#autoConnect--) returns an Observable that automatically connects to this ConnectableObservable when the first Subscriber subscribes.

we are able to share the first emitted nanoTime value to all subscribers.  

```java
public class SharingValues {
    public static void main(String[] args) {
        final Observable<Long> obs = Observable.fromCallable(System::nanoTime).share().replay().autoConnect();

        obs.subscribe(System.out::println);
        obs.subscribe(System.out::println);
        obs.subscribe(System.out::println);
    }
}
```
