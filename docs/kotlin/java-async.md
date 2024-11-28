# Async Programming in Java

## ExecutorService Pattern

```java
ExecutorService service = Executors.newFixedThreadPool(4);
HttpClient client = ...;
Future<String> future = service.submit(() -> client.get("https://google.com"));
String resp = future.get();
```

## CompletableFuture

```java
Supplier<String> fetchA = () -> {return client.get("https://google.com")} ;
Supplier<String> fetchB = () -> {return client.get("https://google.com")} ;
List<Supplier<String> suppliers = List.of(fetchA, fetchB);
List<CompletableFuture<String>> allFutures = new ArrayList();
for (Supplier<String> s : suppliers) {
    CompletableFuture<String> c = CompletableFuture.supplyAsync(s)
    allFutures.add(c);
}
for (CompletableFuture<String> c : allFutures) {
    c.get(); // can throw exception ; or use join() for no exception
}
```

## Task Pipelining

- via `thenApply()`

```java
CompletableFuture<String> getA = CompletableFuture.supplyAsync( () -> doWork() );
CompletableFuture<String> getB = getA.thenApply(
    it -> doWork()
);
```

```java
Collection<String> allResults = new ConcurrentLinkedDeque<>();

Supplier<String> fetchA = () -> {return client.get("https://google.com")} ;
Supplier<String> fetchB = () -> {return client.get("https://google.com")} ;
List<Supplier<String> suppliers = List.of(fetchA, fetchB);
List<CompletableFuture<String>> allFutures = new ArrayList();
for (Supplier<String> s : suppliers) {
    CompletableFuture<String> c = CompletableFuture.supplyAsync(s);
    allFutures.add(c);
}

List<CompletableFuture<String>> voidFutures = new ArrayList();
for (CompletableFuture<String> c : allFutures) {
    CompletableFuture<Void> voids = c.thenAccept(System.out::println);
    voidFutures.add(voids);
    c.thenAccept(allResults::add);
}

// app would die here, therefore we need the join()
for(CompletableFuture<void> v : voidFutures) {
    v.join();
}
```

## Async Composition

### Use the first that finishes `anyOf()` Use all `allOf()`

- `CompletableFuture.anyOf(a, b, c)` can be used
- due to `CompletableFuture<Object>` casting is needed

```java
Random random = new Random();

List<Supplier<Weather>> weatherTasks = buildWeatherTasks(random);
List<Supplier<Quotation>> quotationTasks = buildQuotationTasks(random);

CompletableFuture<Weather>[] weatherCFs = weatherTasks.stream()
    .map(CompletableFuture::supplyAsync)
    .toArray(CompletableFuture[]::new);

CompletableFuture<Weather> weatherCF =
    CompletableFuture.anyOf(weatherCFs)
        .thenApply(weather -> (Weather) weather);


CompletableFuture<Quotation>[] quotationCFS = quotationTasks.stream()
    .map(CompletableFuture::supplyAsync)
    .toArray(CompletableFuture[]::new);

CompletableFuture<Void> allOf = CompletableFuture.allOf(quotationCFS);

CompletableFuture<Quotation> bestQuotationCF = allOf.thenApply(
    v -> Arrays.stream(quotationCFS)
        .map(CompletableFuture::join)
        .min(Comparator.comparing(Quotation::amount))
        .orElseThrow()
);

CompletableFuture<Void> done =
bestQuotationCF.thenCompose(
    quotation ->
        weatherCF.thenApply(weather -> new TravelPage(quotation, weather)))
    .thenAccept(System.out::println);
done.join();
```
