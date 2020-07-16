---
title: "Creating Pairs"
date: 2020-07-01T11:35:28+02:00
draft: false
---
This demo generates consecutive pairs from a sequence of numbers. This is done by combining the original sequence with the same sequence starting from the second element (the skip(1)). The zip function combines the elements from both streams in order and emits as many items as the source observable with the fewest items. See [Zip function](http://reactivex.io/documentation/operators/zip.html). 

```java
// Input:  1 - 2 - 3 - 4 - 5 - 6 - 7 - 8
// Output: [1, 2] - [2, 3]-  [3, 4]-  [4, 5] -  [5, 6] - [6, 7], [7, 8]
final Observable<Integer> numbers = Observable.from(Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8));

numbers
        .zipWith(numbers
                .skip(1), (first, second) -> Arrays.asList(first, second))
        .subscribe(System.out::println);
```
