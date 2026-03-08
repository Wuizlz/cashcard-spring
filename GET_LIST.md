# GET List

## Changes from the Previous Lab

The following updates were made from the previous lab:

- Added more Cash Card data fixtures to `src/test/resources/data.sql`.
- Refactored `CashCardJsonTest.java` to incorporate the new data fixtures.
- Renamed `expected.json` to `single.json`.
- Added another data contract JSON file: `list.json`.
- Added imports to the test classes so they are already available.
- Added the `@DirtiesContext` annotation to the `CashCardApplicationTests` class.

This is only a summary. Each change can be expanded on in the later lab steps.

## Testing the New Data Contract

As in previous workshops, this step starts by writing a test for the expected data contract.

## Review the New Data Fixtures

Look at `src/test/resources/example/cashcard/list.json`. It contains this JSON array:

```json
[
  { "id": 99, "amount": 123.45 },
  { "id": 100, "amount": 1.0 },
  { "id": 101, "amount": 150.0 }
]
```

This is the new data contract for a list of Cash Cards. It should match the fixture data in `src/test/resources/data.sql`.

Now open `CashCardJsonTest.java`. The class-level `cashCards` variable is configured with this Java array:

```java
cashCards = Arrays.array(
     new CashCard(99L, 123.45),
     new CashCard(100L, 100.00),
     new CashCard(101L, 150.00));
```

One of these `CashCard` objects intentionally does not match the fixture data. That sets up a failing test so the behavior is easy to see.

## Add a Serialization Test for the Cash Card List

Add this test to `CashCardJsonTest.java`:

```java
@Test
void cashCardListSerializationTest() throws IOException {
   assertThat(jsonList.write(cashCards)).isStrictlyEqualToJson("list.json");
}
```

This test serializes `cashCards` into JSON and asserts that it matches `list.json`.

## Run the Tests

Use:

```sh
./gradlew test
```

## Expected Result

The test should fail:

```text
java.lang.AssertionError: JSON Comparison failure: [1].amount
Expected: 1.0
     got: 100.0
```

The failure points directly to the second `CashCard` in the array, at index `[1]`.

## Fix and Rerun the Tests

Update the incorrect test value:

```java
new CashCard(100L, 1.00),
```

After correcting the amount from `100.0` to `1.0`, rerun the tests:

```sh
./gradlew test
```

The tests should then pass:

```text
BUILD SUCCESSFUL in 7s
```

## Add a Deserialization Test

Now add a deserialization test:

```java
@Test
void cashCardListDeserializationTest() throws IOException {
   String expected="""
         [
            { "id": 99, "amount": 123.45 },
            { "id": 100, "amount": 100.00 },
            { "id": 101, "amount": 150.00 }
         ]
         """;
   assertThat(jsonList.parse(expected)).isEqualTo(cashCards);
}
```

This test also uses an intentionally incorrect value so the failure is obvious.

## Run the Tests Again

Use:

```sh
./gradlew test
```

## Expected Result

The test should fail with output like:

```text
expected:
  [CashCard[id=99, amount=123.45],
      CashCard[id=100, amount=1.0],
      CashCard[id=101, amount=150.0]]
 but was:
  [CashCard[id=99, amount=123.45],
      CashCard[id=100, amount=100.0],
      CashCard[id=101, amount=150.0]]
```

This time, the failure happens because the JSON string is deserialized and compared to `cashCards`, and the second amount still does not match.

## Fix the Expectation

Update the expected JSON string to:

```java
String expected="""
    [
        { "id": 99, "amount": 123.45 },
        { "id": 100, "amount": 1.00 },
        { "id": 101, "amount": 150.00 }
    ]
    """;
```

Then rerun the tests:

```sh
./gradlew test
```

The deserialization test should now pass:

```text
CashCardJsonTest > cashCardListDeserializationTest() PASSED
```

Now that the list data contract has been tested, the next step is to move on to the controller endpoint.

## Test for an Additional GET Endpoint

Write a failing test for a new `GET` endpoint that returns multiple `CashCard` objects.

## Add the New Test

In `CashCardApplicationTests.java`, add this test:

```java
@Test
void shouldReturnAllCashCardsWhenListIsRequested() {
    ResponseEntity<String> response = restTemplate.getForEntity("/cashcards", String.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
}
```

This sends a request to `/cashcards` and expects a successful response for the full list of cards.

Because the request is for the entire collection, no additional path parameter is needed.

## Run the Tests

Use:

```sh
./gradlew test
```

## Expected Failure

The test should fail because the controller does not yet implement a `GET /cashcards` endpoint.

At first glance, you might expect:

```text
404 NOT_FOUND
```

But the actual failure is:

```text
expected: 200 OK
 but was: 405 METHOD_NOT_ALLOWED
```

## Why the Failure Is `405 METHOD_NOT_ALLOWED`

The `405` happens because `/cashcards` already exists as a route, but only for `POST`.

Spring handles the request like this:

- A request arrives for `/cashcards`.
- There is no mapping for the HTTP `GET` verb at that route.
- There is already a mapping for the HTTP `POST` verb at that same route.
- Spring therefore reports `405 METHOD_NOT_ALLOWED` instead of `404 NOT_FOUND`.

So the route exists, but it does not support `GET` yet.

## Implement the GET Endpoint

To resolve the `405`, add a `GET` handler for `/cashcards` in the controller:

```java
@GetMapping()
private ResponseEntity<Iterable<CashCard>> findAll() {
   return ResponseEntity.ok(cashCardRepository.findAll());
}
```

## Understand the Handler Method

This uses Spring Data's built-in `CrudRepository.findAll()` method.

Because `CashCardRepository` extends `CrudRepository`, Spring Data automatically provides the implementation that returns all `CashCard` records from the database.

## Rerun the Tests

Use:

```sh
./gradlew test
```

## Expected Result

All tests should now pass, including the new list-endpoint test:

```text
BUILD SUCCESSFUL in 7s
```

## Enhance the List Test

So far, the test only verifies that the controller responds successfully to `GET /cashcards`.

Now expand the test so it also verifies that the correct data is returned.

## Update the Test Assertions

Enhance `shouldReturnAllCashCardsWhenListIsRequested()` to:

```java
@Test
void shouldReturnAllCashCardsWhenListIsRequested() {
    ResponseEntity<String> response = restTemplate.getForEntity("/cashcards", String.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);

    DocumentContext documentContext = JsonPath.parse(response.getBody());
    int cashCardCount = documentContext.read("$.length()");
    assertThat(cashCardCount).isEqualTo(3);

    JSONArray ids = documentContext.read("$..id");
    assertThat(ids).containsExactlyInAnyOrder(99, 100, 101);

    JSONArray amounts = documentContext.read("$..amount");
    assertThat(amounts).containsExactlyInAnyOrder(123.45, 100.0, 150.00);
}
```

## Understand the Test

`documentContext.read("$.length()")`

- Calculates the number of items in the returned JSON array.

`documentContext.read("$..id")`

- Collects all `id` values from the response.

`documentContext.read("$..amount")`

- Collects all `amount` values from the response.

These JsonPath expressions make it easy to validate list responses.

`assertThat(...).containsExactlyInAnyOrder(...)`

- This assertion checks that all expected values are present.
- The order does not matter.
- That is important because the database may return the `CashCard` rows in any order.

## Run the Tests

Use:

```sh
./gradlew test
```

## Expected Result

The test should fail with output like:

```text
Expecting actual:
  [123.45, 1.0, 150.0]
to contain exactly in any order:
  [123.45, 100.0, 150.0]
elements not found:
  [100.0]
and elements not expected:
  [1.0]
```

The failure clearly shows that the test expects the second cash card amount to be `100.0`, while the fixture data actually contains `1.0`.

## Correct the Test and Rerun

Update the expectation to:

```java
assertThat(amounts).containsExactlyInAnyOrder(123.45, 1.00, 150.00);
```

Then rerun the tests:

```sh
./gradlew test
```

The test should now pass:

```text
CashCardApplicationTests > shouldReturnAllCashCardsWhenListIsRequested() PASSED
BUILD SUCCESSFUL in 6s
```

## Test Interaction and `@DirtiesContext`

This step explains why `@DirtiesContext` is used in `CashCardApplicationTests`.

You will see two uses of the annotation in that class:

- One at the class level.
- One on `shouldCreateANewCashCard()`, commented out for now.

## Comment Out the Class-Level Annotation

Temporarily change the class definition to:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
//@DirtiesContext(classMode = ClassMode.AFTER_EACH_TEST_METHOD)
class CashCardApplicationTests {
```

## Run All Tests

Use:

```sh
./gradlew test
```

## Expected Result

The list test should now fail:

```text
org.opentest4j.AssertionFailedError:
expected: 3
 but was: 4
    ...
    at app//example.cashcard.CashCardApplicationTests.shouldReturnAllCashCardsWhenListIsRequested(CashCardApplicationTests.java:70)
```

## Why This Happens

Another test creates a new `CashCard`, and that extra row remains in the application state for the later list test.

So instead of returning 3 cash cards, the API now returns 4.

`@DirtiesContext` solves this by telling Spring to reset the application context so tests run with a clean state.

When the class-level annotation is removed, test interaction becomes visible.

## Learning Moment

`@DirtiesContext` is useful when tests change shared application state, but it should not be added indiscriminately.

In this lab, the reason for using it is clear: one test creates a new `CashCard`, so that test needs cleanup.

## Move `@DirtiesContext` to the Create Test

Leave the class-level annotation commented out, and uncomment the method-level annotation on the create test:

```java
//@DirtiesContext(classMode = ClassMode.AFTER_EACH_TEST_METHOD)
class CashCardApplicationTests {
   ...

    @Test
    @DirtiesContext
    void shouldCreateANewCashCard() {
   ...
```

## Run the Tests Again

Use:

```sh
./gradlew test
```

All tests should now pass again.

## Pagination

The next enhancement is paging the list endpoint.

There are 3 `CashCard` records in the database. This step starts by testing a request for a single-item page.

## Write the Pagination Test

Add this test to `CashCardApplicationTests`:

```java
@Test
void shouldReturnAPageOfCashCards() {
    ResponseEntity<String> response = restTemplate.getForEntity("/cashcards?page=0&size=1", String.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);

    DocumentContext documentContext = JsonPath.parse(response.getBody());
    JSONArray page = documentContext.read("$[*]");
    assertThat(page.size()).isEqualTo(1);
}
```

This request adds query parameters:

- `page=0`
- `size=1`

Those values will be handled later in the controller.

## Run the Tests

Use:

```sh
./gradlew test
```

## Expected Result

The new test should fail because the endpoint still returns the full list:

```text
expected: 1
but was: 3
```

## Implement Pagination in `CashCardController`

Add this method to `CashCardController`.

Do not delete the existing `findAll()` method yet.

```java
@GetMapping
private ResponseEntity<List<CashCard>> findAll(Pageable pageable) {
    Page<CashCard> page = cashCardRepository.findAll(
            PageRequest.of(
                    pageable.getPageNumber(),
                    pageable.getPageSize()
    ));
    return ResponseEntity.ok(page.getContent());
}
```

## Understand the Pagination Code

`findAll(Pageable pageable)`

- `Pageable` is another argument Spring Web can inject automatically.
- It captures request parameters such as `page=0` and `size=1`.

`PageRequest.of(pageable.getPageNumber(), pageable.getPageSize())`

- `PageRequest` is a standard Spring implementation of `Pageable`.
- It passes page and size information into repository methods that support paging.

## Try to Compile

Run the tests:

```sh
./gradlew test
```

At this point, the code should fail to compile:

```text
> Task :compileJava FAILED
exercises/src/main/java/example/cashcard/CashCardController.java:50: error: method findAll in interface CrudRepository<T,ID> cannot be applied to given types;
        Page<CashCard> page = cashCardRepository.findAll(
                                                ^
  required: no arguments
  found:    PageRequest
```

## Why It Fails

`CrudRepository` supports `findAll()`, but not the paging overload that accepts a `Pageable`.

The repository needs to support paging and sorting explicitly.

## Extend `PagingAndSortingRepository`

Update `CashCardRepository.java` to also extend `PagingAndSortingRepository`.

Add this import:

```java
import org.springframework.data.repository.PagingAndSortingRepository;
```

Then update the interface:

```java
interface CashCardRepository extends CrudRepository<CashCard, Long>, PagingAndSortingRepository<CashCard, Long> { ... }
```

Now the repository supports paging and sorting operations.

## Run the Tests Again

Use:

```sh
./gradlew test
```

## Next Failure: Ambiguous Mapping

The code now compiles, but Spring should fail to start with an error like:

```text
Failed to load ApplicationContext
java.lang.IllegalStateException: Failed to load ApplicationContext
...
Caused by: java.lang.IllegalStateException: Ambiguous mapping. Cannot map 'cashCardController' method
example.cashcard.CashCardController#findAll(Pageable)
to {GET [/cashcards]}: There is already 'cashCardController' bean method
example.cashcard.CashCardController#findAll() mapped.
```

## Why This Happens

The controller now has two methods mapped to the same endpoint:

- `GET /cashcards` with no parameters
- `GET /cashcards` with `Pageable`

Even though the Java method signatures are different, Spring sees them as the same route mapping.

That creates an ambiguous mapping during application startup.

## Remove the Old `findAll()` Method

Delete the old controller method:

```java
@GetMapping()
private ResponseEntity<Iterable<CashCard>> findAll() {
    return ResponseEntity.ok(cashCardRepository.findAll());
}
```

## Run the Tests One More Time

Use:

```sh
./gradlew test
```

The tests should now pass:

```text
BUILD SUCCESSFUL in 7s
```

The next step is to add sorting support.

## Sorting

The next improvement is to return `CashCard` results in an order that makes sense to humans.

In this step, the cards should be ordered by `amount` in descending order, with the highest amounts first.

## Write the Sorting Test

Add this test to `CashCardApplicationTests`:

```java
@Test
void shouldReturnASortedPageOfCashCards() {
    ResponseEntity<String> response = restTemplate.getForEntity("/cashcards?page=0&size=1&sort=amount,desc", String.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);

    DocumentContext documentContext = JsonPath.parse(response.getBody());
    JSONArray read = documentContext.read("$[*]");
    assertThat(read.size()).isEqualTo(1);

    double amount = documentContext.read("$[0].amount");
    assertThat(amount).isEqualTo(150.00);
}
```

## Understand the Test

The request URI contains both pagination and sorting parameters:

`/cashcards?page=0&size=1&sort=amount,desc`

- `page=0`: request the first page.
- `size=1`: request a page containing one item.
- `sort=amount,desc`: sort by `amount` in descending order.

The assertions expect that the single returned `CashCard` is the one with amount `150.00`.

## Run the Tests

Use:

```sh
./gradlew test
```

## Expected Failure

The test should fail like this:

```text
org.opentest4j.AssertionFailedError:
 expected: 150.0
  but was: 123.45
```

## Why It Fails

The controller currently supports paging, but it does not yet apply the sort information from the request.

That means the cards come back in the database's default order, which in this case happens to match insertion order.

This is exactly why explicit sorting matters: relying on default database order is not safe or portable.

## Implement Sorting in the Controller

Update the `PageRequest.of(...)` call in `CashCardController` to include sorting:

```java
PageRequest.of(
     pageable.getPageNumber(),
     pageable.getPageSize(),
     pageable.getSort()
));
```

`pageable.getSort()` extracts the sort information from the request URI and applies it to the page request.

## Run the Tests Again

Use:

```sh
./gradlew test
```

All tests should now pass.

## Learn by Breaking Things

To build confidence in the test, try changing the request from descending to ascending order:

```java
ResponseEntity<String> response = restTemplate.getForEntity("/cashcards?page=0&size=1&sort=amount,asc", String.class);
```

Now rerun the tests.

The test should fail because the first `CashCard` in ascending order should be the `$1.00` card:

```text
CashCardApplicationTests > shouldReturnASortedPageOfCashCards() FAILED
org.opentest4j.AssertionFailedError:
 expected: 150.0
  but was: 1.0
```

This confirms that the test is actually verifying sort behavior.

After the experiment, change the test back to descending order so it passes again.

## Paging and Sorting Defaults

The endpoint currently expects the client to provide several details:

- page index
- page size
- sort field
- sort direction

That is more than we want to require, so this step adds reasonable defaults.

## Write a Test with No Pagination or Sorting Parameters

Add this test to `CashCardApplicationTests`:

```java
@Test
void shouldReturnASortedPageOfCashCardsWithNoParametersAndUseDefaultValues() {
    ResponseEntity<String> response = restTemplate.getForEntity("/cashcards", String.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);

    DocumentContext documentContext = JsonPath.parse(response.getBody());
    JSONArray page = documentContext.read("$[*]");
    assertThat(page.size()).isEqualTo(3);

    JSONArray amounts = documentContext.read("$..amount");
    assertThat(amounts).containsExactly(1.00, 123.45, 150.00);
}
```

This test expects the following defaults:

- Sort by `amount` ascending.
- Use a page size larger than `3`, so all fixture rows are returned.

## Run the Tests

Use:

```sh
./gradlew test
```

## Expected Failure

The test should fail with output like:

```text
Actual and expected have the same elements but not in the same order, at index 0 actual element was:
  123.45
whereas expected element was:
  1.0
```

This tells us:

- All three `CashCard` records were returned, because `page.size()` was `3`.
- The list was not sorted as expected, because the amounts are not in ascending order.

## Make the Test Pass

Update the controller by changing the `PageRequest.of(...)` call to:

```java
PageRequest.of(
        pageable.getPageNumber(),
        pageable.getPageSize(),
        pageable.getSortOr(Sort.by(Sort.Direction.ASC, "amount"))
));
```

## Understand the Implementation

`getSortOr(...)` provides a default sort when the request does not specify one.

The defaults come from two places:

- Spring provides default values for page and size automatically.
- Your code provides the default sort order.

Spring’s defaults are:

- page `0`
- size `20`

That explains why all three fixture records are returned even when the request does not include paging parameters.

Your code supplies the default sort:

```java
Sort.by(Sort.Direction.ASC, "amount")
```

This means the response will be sorted by `amount` ascending whenever the client does not specify a sort.

## Run the Tests Again

Use:

```sh
./gradlew test
```

Everything should now pass:

```text
BUILD SUCCESSFUL in 7s
```

## Summary

In this lab, a "GET many" endpoint was implemented with both pagination and sorting.

These changes accomplish two important goals:

- They ensure that data returned from the server is predictable and understandable.
- They protect both the client and the server from being overwhelmed by too much data in a single response.

The page size puts an upper bound on how much data can be returned at once, while sorting makes the response order explicit instead of relying on database defaults.
