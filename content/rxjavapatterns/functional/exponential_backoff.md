---
title: "Exponential Backoff"
date: 2020-07-16T11:35:28+02:00
draft: false
---
 When scheduling repeated work [Exponential backoff](https://en.wikipedia.org/wiki/Exponential_backoff) can be used to retry a failed request. When the scheduling interval is high (for example 1 hour), this makes sure the request is retried relatively fast so you do not have to wait before the next interval occurs.

Exponential backoff can be implemented using the onErrorResumeNext function and then comparing the actual number of invocations with a specified maximum. Each next invocation is delayed exponentially more then the previous invocation. 

```java
public class ExponentialBackOff {
    private static final String RESPONSE = "[\n" +
            "  {\"name\": \"'s-Hertogenbosch\"},\n" +
            "  {\"name\": \"Oss\"},\n" +
            "  {\"name\": \"Maastricht\"},\n" +
            "  {\"name\": \"Utrecht Centraal\"},\n" +
            "  {\"name\": \"Groningen\"}\n" +
            "]";
    private static StationsApi stationsApi;

    public static void main(String[] args) throws IOException, InterruptedException {
        final TestScheduler testScheduler = Schedulers.test();
//        final Scheduler testScheduler = Schedulers.immediate();
        final int maxFails = 3;
        final int iterations = 5 + maxFails;
        final MockWebServer mockWebServer = prepareResponses(iterations, maxFails);
        stationsApi = RetroFitFactory.getApi(mockWebServer.url("/").toString(), testScheduler).create(StationsApi.class);

        // Demo
        final Subscription subsription = Observable.interval(1, TimeUnit.HOURS, testScheduler)
                .doOnNext(hour -> System.out.println("------------ " + hour + " hour(s) ------------"))
                .switchMap(i -> updateStations(4, 1, testScheduler)
                        .doOnError(throwable -> System.out.println(throwable.getMessage()))
                        .onErrorResumeNext(throwable -> Observable.empty()))
                .subscribe(System.out::println);

        for (int i = 0; i < iterations; i++) {
            testScheduler.advanceTimeBy(1, TimeUnit.HOURS);
        }
        // Demo

        subsription.unsubscribe();
        mockWebServer.shutdown();
    }

    private static MockWebServer prepareResponses(final int iterations, final int maxFailes) throws IOException {
        final MockWebServer mockWebServer = new MockWebServer();
        for (int i = 0; i < iterations + maxFailes; i++) {
            if (i >= 2 && i < 2 + maxFailes) {
                mockWebServer.enqueue(new MockResponse().setResponseCode(500));
            } else {
                mockWebServer.enqueue(new MockResponse().setBody(RESPONSE).setResponseCode(200));
            }
        }
        mockWebServer.start();
        return mockWebServer;
    }

    private static Observable<List<Station>> updateStations(final int maxRetries, final int numRetries, final Scheduler scheduler) {
        System.out.println("Retrieve stations");
        final long delay = (long) Math.pow(2, numRetries) * 1000;
        return stationsApi.retrieveStations()
                .doOnError(throwable -> {
                    System.out.println(String.format("Error. Retrying again in %d milliseconds", delay));
                })
                .onErrorResumeNext(throwable -> {
                    if (numRetries < maxRetries) {
                        System.out.println(String.format("Retrying %d time", numRetries));
                        return Observable.just(1).delay(delay, TimeUnit.MILLISECONDS, scheduler)
                                .switchMap(i -> updateStations(maxRetries, numRetries + 1, scheduler));
                    }

                    return Observable.error(throwable);
                });
    }

    private interface StationsApi {

        @GET("/stations")
        Observable<List<Station>> retrieveStations();
    }

    private static class Station {
        String name;

        @Override
        public String toString() {
            final StringBuilder sb = new StringBuilder("Station{");
            sb.append("name='").append(name).append('\'');
            sb.append('}');
            return sb.toString();
        }
    }
}
```
