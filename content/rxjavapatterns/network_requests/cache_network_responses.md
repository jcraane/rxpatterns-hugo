---
title: "Cache Network Responses"
date: 2020-07-16T11:35:28+02:00
draft: false
---
This pattern demonstrates a strategy for caching network data using reactive extensions. It is a common pattern to cache data a specific time before fetching it again from a network. The basic flow is as follows:

1. Fetch data from the in-memory store if it exists and is not stale.
2. Fetch data from the disk store if it exists and is not stale.
3. Fetch data from the network.
4. If succesfull, populate the in-memory and disk store.
5. If not succesfull, retrieve potential stale data from the disk cache if it exists (this disk cache acts as a fallback mechanism).
6. If no data exists in the disk cache, reply with an error.

The in-memory and disk cache may use different expiry policies. The in-memory cache may have a size- and time-based constraint while the disk cache only has a time-based constraint. The disk cache will survive application restarts while the in-memory cache may not.

When using the disk cache as a fallback mechanism in case of network errors, think about what to show to the user. Perhaps the timestamp of the data so the user knows the data is not up-to-date and may need to be refresed later.

Remember this is not the only way to handle this scenario. You might implement an [exponential backoff](http://www.rxpatterns.com/#sub-par-4-2) algorithm when retrieving data from the network before using the disk cache as a fallback mechanism.   
```java
public class CacheNetworkResponses {
    private static boolean simulateNetworkError;

    public static void main(String[] args) {
        final Cache memoryCache = new Cache("MEMORY"), diskCache = new Cache("DISK");

        final Observable<String> multipleSources = Observable.concat(
                memoryCache.getFromCache(false),
                diskCache.getFromCache(false),
                fromNetwork()
                        .doOnNext(memoryCache::putInCache)
                        .doOnNext(diskCache::putInCache)
                        .onErrorResumeNext(throwable ->
                                diskCache.getFromCache(true)
                                        .filter(Objects::nonNull)
                                        .switchIfEmpty(Observable.error(throwable)))
        )
                .take(1);

        multipleSources
                .subscribe(new DataSubscriber());

        multipleSources
                .subscribe(new DataSubscriber());

        memoryCache.invalidate();

        multipleSources
                .subscribe(new DataSubscriber());

        simulateNetworkError = true;
        diskCache.stale = true;

        multipleSources
                .subscribe(new DataSubscriber());

        simulateNetworkError = true;
        diskCache.cached = null;

        multipleSources
                .subscribe(new DataSubscriber());
    }

    static Observable<String> fromNetwork() {
        return Observable.fromCallable(() -> {
            System.out.println("getFrom NETWORK");
            if (simulateNetworkError) {
                throw new RuntimeException("Network error occurred");
            }
            return "value";
        });
    }

    private static class Cache {
        final String type;
        boolean stale;
        String cached;

        private Cache(final String type) {
            this.type = type;
        }

        Observable<String> getFromCache(final boolean ignoreStaleData) {
            return Observable.defer(() -> {
                System.out.println(String.format("getFrom [%s]", type));
                if (cached == null || (stale && !ignoreStaleData)) {
                    return Observable.empty();
                }

                return Observable.just(cached);
            });
        }

        void putInCache(final String value) {
            cached = value;
        }

        void invalidate() {
            cached = null;
        }
    }

    private static class DataSubscriber extends Subscriber<String> {
        @Override
        public void onCompleted() {

        }

        @Override
        public void onError(final Throwable e) {
            System.out.println("Cannot load data");
        }

        @Override
        public void onNext(final String s) {
            System.out.println(s);
        }
    }
}
```
