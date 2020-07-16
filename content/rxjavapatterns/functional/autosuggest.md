---
title: "Autosuggest"
date: 2020-07-07T11:35:28+02:00
draft: false
---
The auto suggest demo, demonstrates a possible implementation for text based suggestion using rx. The following solution address the following, functional problems:

1. Only execute a search request if there are at least 3 characters typed in by the user. The filter operator is used for this.
2. Do not execute search requests when the user types in characters too fast. The debounce operator is used for this.
3. Make sure older search results are disgarded when the user has typed additional characters to further limit the search. switchMap is used instaed of flatMap.
 

```java
public class AutoSuggest {
    private static final int[] SLEEP_TIMES = new int[]{30, 30, 30, 30, 10, 10, 10, 30, 30, 10, 10, 30, 30};

    public static void main(String[] args) throws InterruptedException {
        final AutoSuggestService autoSuggestService = new AutoSuggestService();
        final PublishSubject<String> searchField = PublishSubject.create();

        searchField
                .filter(input -> input.length() > 2) // Only search when input contains 3 characters or more
                .debounce(25, TimeUnit.MILLISECONDS) // Do not search immediately when typing fast
                .switchMap(input -> autoSuggestService.suggest(input) // switchMap makes sure old requests are discard
                        .doOnError(System.out::println)
                        .onErrorResumeNext(throwable -> Observable.just(Collections.emptyList()))) // Do not die on onError
                .subscribe(System.out::println);

        final String input = "Amsterdam Kor";
        for (int i = 0; i < input.length(); i++) {
            searchField.onNext(input.substring(0, i));
            Thread.sleep(SLEEP_TIMES[i]);
        }
    }

    private static class AutoSuggestService {
        private final Map<String, List<String>> results = initSearchResults();

        private Map<String, List<String>> initSearchResults() {
            final Map<String, List<String>> values = new HashMap<>();
            values.put("", Arrays.asList("Amsterdam Kerkstraat", "Alkmaar Kerkstraat", "Amstelveen Kruisstraat", "Abcoude Kerkstraat", "Almere Kerkplein"));
            values.put("A", Arrays.asList("Amsterdam Kerkstraat", "Alkmaar Kerkstraat", "Amstelveen Kruisstraat", "Abcoude Kerkstraat", "Almere Kerkplein"));
            values.put("Am", Arrays.asList("Amsterdam Kerkstraat", "Amsterdam Kalverstraat", "Amstelveen Kruisstraat", "Amstelveen Kromweg"));
            values.put("Ams", Arrays.asList("Amsterdam Kerkstraat", "Amsterdam Kalverstraat", "Amstelveen Kruisstraat", "Amstelveen KraalWeg"));
            values.put("Amst", Arrays.asList("Amsterdam Kerkstraat", "Amsterdam Kalverstraat", "Amstelveen Kruisstraat", "Amstelveen KromWeg"));
            values.put("Amste", Arrays.asList("Amsterdam Kerkstraat", "Amsterdam Kalverstraat", "Amstelveen Kruisstraat", "Amstelveen Kromweg"));
            values.put("Amster", Arrays.asList("Amsterdam Kerkstraat", "Amsterdam Kalverstraat", "Amsterdam Kometensingel", "Amsterdam Elpermeer"));
            values.put("Amsterd", Arrays.asList("Amsterdam Kerkstraat", "Amsterdam Kalverstraat", "Amsterdam Kometensingel", "Amsterdam Elpermeer"));
            values.put("Amsterda", Arrays.asList("Amsterdam Kerkstraat", "Amsterdam Kalverstraat", "Amsterdam Kometensingel", "Amsterdam Elpermeer"));
            values.put("Amsterdam", Arrays.asList("Amsterdam Kerkstraat", "Amsterdam Kalverstraat", "Amsterdam Kometensingel", "Amsterdam Elpermeer"));
            values.put("Amsterdam ", Arrays.asList("Amsterdam Kerkstraat", "Amsterdam Kalverstraat", "Amsterdam Kometensingel", "Amsterdam Elpermeer"));
            values.put("Amsterdam K", Arrays.asList("Amsterdam Kerkstraat", "Amsterdam Kalverstraat", "Amsterdam Kometensingel", "Amsterdam Korenbloemstraat"));
            values.put("Amsterdam Ko", Arrays.asList("Amsterdam Korenbloemstraat", "Amsterdam Kompasstraat", "Amsterdam Kometensingel"));
            values.put("Amsterdam Kor", Arrays.asList("Amsterdam Korenbloemstraat", "Amsterdam Korte Leidsedwarsstraat", "Amsterdam Korte Keizersstraat"));
            return values;
        }


        public Observable<List<String>> suggest(final String input) {
            return Observable.fromCallable(() -> {
                System.out.println(String.format("Search --> %s", input));
                if (input.length() == 3) {
                    Thread.sleep(50);
                }
                if (input.length() == 7) {
                    throw new RuntimeException("Error");
                }
                return results.get(input);
            }).subscribeOn(Schedulers.io());
        }
    }
}
```
