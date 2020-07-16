---
title: "Scheduling Network Requests"
date: 2020-07-16T11:35:28+02:00
draft: false
---
This pattern demonstrates scheduling periodic network requests which can be useful for keeping data up-to-date where web sockets or some other pusb based technology is not an option. To retrieve data from the network the switchMap operator is used instead of the flatMap operator. The difference between those operators is explained [here](https://medium.com/appunite-edu-collection/rxjava-flatmap-switchmap-and-concatmap-differences-examples-6d1f3ff88ee0).

To make sure the scheduler does not die when the Observable in switchMap emits an error, the onErrorResumeNext function is used to provide an alternative response in case of errors. 

```java
public class SchedulingExceptionDemo {
    public static void main(String[] args) throws IOException {
        final MockWebServer mockWebServer = new MockWebServer();
        for (int i = 0; i < 10; i++) {
            if (i == 2) {
                mockWebServer.enqueue(new MockResponse().setBody(DepartureTimes.getBody(i)).setResponseCode(500));
            } else {
                mockWebServer.enqueue(new MockResponse().setBody(DepartureTimes.getBody(i)).setResponseCode(200));
            }
        }
        mockWebServer.start();

        final DepartureTimesApi departureTimesApi = RetroFitFactory.getApi(mockWebServer.url("/").toString(), Schedulers.immediate()).create(DepartureTimesApi.class);

        final TestScheduler testScheduler = Schedulers.test();
        Observable.interval(1, TimeUnit.SECONDS, testScheduler)
                .doOnNext(i -> System.out.println("INTERVAL = " + i))
                .switchMap(i -> departureTimesApi.getDepartureTimes("ut") // switchMap only processes the latest event from the source (flatMap processes everything)
                        .doOnError(throwable -> System.out.println(throwable.getMessage())) // Log error
                        .onErrorResumeNext(Observable.empty())) // Return empty observable
                .subscribe(System.out::println);

        testScheduler.advanceTimeBy(1, TimeUnit.SECONDS);
        testScheduler.advanceTimeBy(1, TimeUnit.SECONDS);
        testScheduler.advanceTimeBy(1, TimeUnit.SECONDS);
        testScheduler.advanceTimeBy(1, TimeUnit.SECONDS);

        mockWebServer.shutdown();
    }
}
```
