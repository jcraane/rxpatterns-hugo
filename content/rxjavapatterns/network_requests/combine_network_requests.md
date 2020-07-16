---
title: "Combine Network Requests/Responses"
date: 2020-07-03T11:35:28+02:00
draft: false
---
This pattern demonstrates retrieving data from an asynchronous datasource and then using the result of this invocation to retrieve two pieces of data from two other asynchronous data sources and then combining the result of both invocations in one response.
 
Please note that when one of the sources in the zipWith operator fails, no result is retured at all since zipWith needs both observables to emit a value.  
```java
public class CombineNetworkRequests {
    public static void main(String[] args) throws IOException {
        final MockWebServer mockWebServer = new MockWebServer();
        mockWebServer.enqueue(new MockResponse().setBody("{\"lat\": 52.2, \"lng\": 5.1}").setResponseCode(200)); // Response for getLastKnownLocation
        mockWebServer.enqueue(new MockResponse().setBody("{\"lat\": 52.55, \"lng\": 5.11}").setResponseCode(200)); // Response for resolve location
        mockWebServer.enqueue(new MockResponse().setBody("{\"temp\": 18.5}").setResponseCode(200)); // Response for weather
        mockWebServer.enqueue(new MockResponse().setBody("{\"name\": \"Utrecht\"}").setResponseCode(200)); // Response for location
        mockWebServer.start();

        final String baseUrl = mockWebServer.url("/").toString();
        final LocationApi locationApi = RetroFitFactory.getApi(baseUrl, Schedulers.immediate()).create(LocationApi.class);
        final WeatherApi weatherApi = RetroFitFactory.getApi(baseUrl, Schedulers.immediate()).create(WeatherApi.class);
        final PlacesApi placesApi = RetroFitFactory.getApi(baseUrl, Schedulers.immediate()).create(PlacesApi.class);

        getLastKnownLocationObservable(locationApi).switchIfEmpty(locationApi.resolveUserLocation())
                .flatMap(location -> weatherApi.getTemp(location.lat, location.lng)
                        .zipWith(placesApi.reverseGeocode(location.lat, location.lng), (Func2<Weather, City, Pair>) Pair::new))
                .subscribe(System.out::println);

        mockWebServer.shutdown();
    }

    private static Observable<Location> getLastKnownLocationObservable(final LocationApi locationApi) {
//        return Observable.empty();
        return locationApi.getLastKnownLocation();
    }
}
```
