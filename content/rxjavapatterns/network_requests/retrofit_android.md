---
title: "Retrofit Android"
date: 2020-07-02T11:35:28+02:00
draft: false
---
This patterns demos executing a [Retrofit](http://square.github.io/retrofit/) request within Android. To always execute the request in the background you can create the RxJavaCallAdapter by using the Schedulers.io() scheduler. For testing you can pass an immediate() or a test scheduler during creation of the API.

Make sure that when you touch any views in the subsribe function, you must use observeOn(AndroidSchedulers.mainThread()) because views can only be updated from the main thread. Also use an onError handler to avoid OnErrorNotImplemented exceptions.

The subscription is unsubscribed when the network request is finished. It is good practive however to unsubsribe in the proper life-cycle method, for example onPause(). For this you can use a CompositeSubscription and add any subscriptions that must be unsubscribed when the activity pauses. To unsubscribe using a CompositeSubscription call its clear() method. By using the clear() method, the CompositeSubscription can be re-used in contrast to the unsubscribe() method. 
```java
public class RetrofitTest {
    public static void main(String[] args) throws IOException {
        final MockWebServer mockWebServer = new MockWebServer();
        mockWebServer.enqueue(new MockResponse().setBody("{\"message\": \"Hallo\"}").setResponseCode(200));
        mockWebServer.play();
        final TestApi api = getApi(mockWebServer.getUrl("/").toString(), Schedulers.io());
        final Subscription subscription = api.getMessage()
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(System.out::println,
                        Throwable::printStackTrace);

//        when result is delivered, subscription is unsubscribed. In Android, best to unsubscribe in lifecycle methods since
        // the request does not always complete (for example on orientation change, back button etc.)
        System.out.println(subscription.isUnsubscribed());

        mockWebServer.shutdown();
    }

    public static TestApi getApi(final String baseUrl, final Scheduler scheduler) {
        return new Retrofit.Builder()
                .baseUrl(baseUrl)
                .addCallAdapterFactory(RxJavaCallAdapterFactory.createWithScheduler(scheduler))
                .addConverterFactory(GsonConverterFactory.create(new GsonBuilder().create()))
                .build().create(TestApi.class);
    }
}
```
