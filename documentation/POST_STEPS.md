# Test the HTTP POST Endpoint

As in the earlier labs, this step starts with a test that defines success before the endpoint is implemented.

## Add a Test for the POST Endpoint

Edit `src/test/java/example/cashcard/CashCardApplicationTests.java` and add this test method:

```java
@Test
void shouldCreateANewCashCard() {
   CashCard newCashCard = new CashCard(null, 250.00);
   ResponseEntity<Void> createResponse = restTemplate.postForEntity("/cashcards", newCashCard, Void.class);
   assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
}
```

## Understand the Test

`CashCard newCashCard = new CashCard(null, 250.00);`

- The database will create and manage the `CashCard.id` value.
- You should not provide an ID when creating a new cash card.

`restTemplate.postForEntity("/cashcards", newCashCard, Void.class);`

- This works like `restTemplate.getForEntity`, but it also sends `newCashCard` as the request body.
- The expected response body is empty, so the response type is `Void`.

## Run the Tests

Use:

```sh
./gradlew test
```

## What to Expect

The test should fail:

```text
CashCardApplicationTests > shouldCreateANewCashCard() FAILED
   org.opentest4j.AssertionFailedError:
   expected: 200 OK
   but was: 404 NOT_FOUND
```

## Why It Fails

The `POST /cashcards` endpoint has not been implemented yet, so the application returns `404 NOT_FOUND`.

This is the expected failure in a test-first workflow.

## Next Step

Add the `POST /cashcards` endpoint so this test passes.

## Add the POST Endpoint

The `POST` endpoint is similar to the `GET` endpoint in `CashCardController`, but it uses Spring Web's `@PostMapping` annotation.

At this stage, add the endpoint without accepting any `CashCard` data yet.

### Update the Controller

Edit `src/main/java/example/cashcard/CashCardController.java` and add the import:

```java
import org.springframework.web.bind.annotation.PostMapping;
```

Then add this method:

```java
@PostMapping
private ResponseEntity<Void> createCashCard() {
   return null;
}
```

### Why This Passes

Even though the method does not create anything yet, returning `null` here allows Spring Web to generate an HTTP status of `200 OK` for the request in this lab step.

### Run the Tests

Use:

```sh
./gradlew test
```

### Expected Result

At this point, the tests should pass:

```text
BUILD SUCCESSFUL in 7s
```

### What This Means

The test passes because the endpoint now exists and responds successfully, but it still does not use the submitted `CashCard` data or create a new record.

This is why the result is not very satisfying yet, and why the next step should improve the tests.

## Testing Based on Semantic Correctness

We want the Cash Card API to behave as semantically correctly as possible so API consumers are not surprised by its behavior.

For `POST`, the HTTP semantics in RFC 9110 say that when a resource is created, the server should respond with `201 Created` and include a `Location` header that identifies the newly created resource.

## Update the POST Test

Edit `src/test/java/example/cashcard/CashCardApplicationTests.java` and add this import:

```java
import java.net.URI;
```

Then update `shouldCreateANewCashCard()` to:

```java
@Test
void shouldCreateANewCashCard() {
   CashCard newCashCard = new CashCard(null, 250.00);
   ResponseEntity<Void> createResponse = restTemplate.postForEntity("/cashcards", newCashCard, Void.class);
   assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);

   URI locationOfNewCashCard = createResponse.getHeaders().getLocation();
   ResponseEntity<String> getResponse = restTemplate.getForEntity(locationOfNewCashCard, String.class);
   assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
}
```

## Understand the Test Updates

`assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);`

- The expected response status is now `201 CREATED`.
- This is the semantically correct response when a `POST` request creates a new resource.

`URI locationOfNewCashCard = createResponse.getHeaders().getLocation();`

- The response should include a `Location` header.
- That header should identify the newly created `CashCard`.
- `URI` is the correct type here because a URL is only one kind of URI.

`ResponseEntity<String> getResponse = restTemplate.getForEntity(locationOfNewCashCard, String.class);`

- The test uses the `Location` header to fetch the newly created resource.
- A successful lookup should return `200 OK`.

## Run the Tests

Use:

```sh
./gradlew test
```

## Expected Result

The test should now fail on the first updated assertion:

```text
expected: 201 CREATED
  but was: 200 OK
```

## Why It Fails

The current `POST /cashcards` endpoint exists, but it still returns `200 OK` and does not provide a `Location` header for a newly created resource.

The next step is to fix the implementation so it behaves correctly for resource creation.

## Implement the POST Endpoint

The `POST` endpoint in `CashCardController` is still empty. This step fills in the correct behavior incrementally.

## Return `201 CREATED`

Start by returning the correct status code and a placeholder `Location` header.

Edit `src/main/java/example/cashcard/CashCardController.java` and add these imports:

```java
import java.net.URI;
import org.springframework.web.bind.annotation.RequestBody;
```

Then update the `POST` handler to:

```java
@PostMapping
private ResponseEntity<Void> createCashCard(@RequestBody CashCard newCashCardRequest) {
    return ResponseEntity.created(URI.create("/what/should/go/here?")).build();
}
```

## Run the Tests

Use:

```sh
./gradlew test
```

## Expected Result

The updated test should now pass the `201 CREATED` assertion and fail on the final lookup:

```text
expected: 200 OK
 but was: 404 NOT_FOUND
```

## Why It Fails

The endpoint now returns a semantically correct creation response, but it still does not actually save a new `CashCard`.

The final `GET` in the test uses the `Location` header to retrieve the created resource, and that resource does not exist yet, so the response is `404 NOT_FOUND`.

## Save the New Cash Card and Return Its Location

Now complete the `POST` implementation so it saves the new record and returns the correct URI.

Add this import:

```java
import org.springframework.web.util.UriComponentsBuilder;
```

Then update the method to:

```java
@PostMapping
private ResponseEntity<Void> createCashCard(@RequestBody CashCard newCashCardRequest, UriComponentsBuilder ucb) {
   CashCard savedCashCard = cashCardRepository.save(newCashCardRequest);
   URI locationOfNewCashCard = ucb
            .path("cashcards/{id}")
            .buildAndExpand(savedCashCard.id())
            .toUri();
   return ResponseEntity.created(locationOfNewCashCard).build();
}
```

## What Changed

`@RequestBody CashCard newCashCardRequest`

- Spring reads the incoming JSON request body and converts it into a `CashCard`.

`cashCardRepository.save(newCashCardRequest)`

- The new cash card is stored in the database.
- The returned `savedCashCard` contains the generated ID.

`UriComponentsBuilder`

- Builds the `Location` URI for the newly created resource.
- The generated ID is inserted into the path so clients know where to retrieve the new `CashCard`.

`ResponseEntity.created(locationOfNewCashCard).build()`

- Returns `201 CREATED`.
- Includes the `Location` header required for a newly created resource.

The next step is to walk through these changes in more detail.

## Understand `CrudRepository.save`

This line in `CashCardController.createCashCard` looks simple, but it does a lot of work:

```java
CashCard savedCashCard = cashCardRepository.save(newCashCardRequest);
```

As covered in earlier lessons and labs, Spring Data's `CrudRepository` provides methods for creating, reading, updating, and deleting data in a data store.

`cashCardRepository.save(newCashCardRequest)` does exactly what it says:

- It saves the new `CashCard`.
- It returns the saved object.
- The returned object includes the unique `id` generated by the database.

This generated `id` is what allows the controller to build the `Location` header for the newly created resource.

## Understand the Other Changes to `CashCardController`

The controller now implements the expected input and output behavior for an HTTP `POST`.

`createCashCard(@RequestBody CashCard newCashCardRequest, ...)`

- Unlike the earlier `GET` endpoint, `POST` expects a request body.
- That request body contains the data submitted to the API.
- Spring Web deserializes the incoming JSON into a `CashCard` object automatically.

`URI locationOfNewCashCard = ucb`
`   .path("cashcards/{id}")`
`   .buildAndExpand(savedCashCard.id())`
`   .toUri();`

- This builds a URI for the newly created `CashCard`.
- The caller can use that URI to `GET` the resource after creation.
- `savedCashCard.id()` is used as the identifier, which matches the `GET /cashcards/{id}` endpoint design.

## Where `UriComponentsBuilder` Comes From

`UriComponentsBuilder ucb` is listed as a method argument in the controller handler, and Spring provides it automatically.

This works because Spring Web injects framework-managed arguments into handler methods through the IoC container.

## Final Response

`return ResponseEntity.created(locationOfNewCashCard).build();`

- Returns `201 CREATED`.
- Includes the correct `Location` header.
- Signals to the client where the newly created resource can be retrieved.

## Final Testing and Learning Moment

Run the tests again:

```sh
./gradlew test
```

At this point, they should pass:

```text
BUILD SUCCESSFUL in 7s
```

The new `CashCard` was created successfully, and the URI in the `Location` response header was used to retrieve the new resource.

## Add More Test Assertions

If you want to strengthen the test, add assertions for the new `id` and `amount` after:

```java
assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
```

Add:

```java
DocumentContext documentContext = JsonPath.parse(getResponse.getBody());
Number id = documentContext.read("$.id");
Double amount = documentContext.read("$.amount");

assertThat(id).isNotNull();
assertThat(amount).isEqualTo(250.00);
```

These assertions verify:

- The newly created `CashCard.id` is not `null`.
- The `CashCard.amount` is `250.00`, matching the create request.

## Learning Moment

Earlier, we said the database and repository manage ID creation for new `CashCard` records.

What happens if you provide an `id` for a new, unsaved `CashCard`?

## Update the Test to Submit an ID

Change the created object from `null` to a non-existent ID such as `44L`:

```java
@Test
void shouldCreateANewCashCard() {
   CashCard newCashCard = new CashCard(44L, 250.00);
   ...
}
```

## Enable More Verbose Test Output

Edit `build.gradle` and make sure test logging includes:

```groovy
test {
  testLogging {
    ...
    // Set to `true` for more detailed logging.
    showStandardStreams = true
  }
}
```

## Run the Tests

Use:

```sh
./gradlew test
```

## Expected Result

The API should now fail with:

```text
expected: 201 CREATED
 but was: 500 INTERNAL_SERVER_ERROR
```

## Find and Understand the Failure

Search the test output for:

```text
Failed to update entity [CashCard[id=44, amount=250.0]]. Id [44] not found in database.
```

This shows that the repository is trying to update an existing `CashCard` with ID `44`, and it fails because that record does not exist.

## Why This Happens

Supplying an ID to `cashCardRepository.save(...)` is supported for updating an existing resource.

When you provide an ID for a create request, Spring Data treats it as an update attempt rather than a brand-new insert.

## What You Learned

When creating a new `CashCard`, the API expects that you do not provide a `CashCard.id`.

In a later step, the API can be improved to validate this rule explicitly and reject invalid create requests more cleanly.
