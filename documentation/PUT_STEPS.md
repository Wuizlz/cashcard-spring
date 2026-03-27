# Test the HTTP PUT Endpoint

This step defines the update behavior for `PUT /cashcards/{id}` before the controller endpoint exists.

## Add the Update Test

Edit `src/test/java/example/cashcard/CashCardApplicationTests.java` and add these imports:

```java
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpMethod;
```

Then add this test:

```java
@Test
@DirtiesContext
void shouldUpdateAnExistingCashCard() {
    CashCard cashCardUpdate = new CashCard(null, 19.99, null);
    HttpEntity<CashCard> request = new HttpEntity<>(cashCardUpdate);
    ResponseEntity<Void> response = restTemplate
            .withBasicAuth("sarah1", "abc123")
            .exchange("/cashcards/99", HttpMethod.PUT, request, Void.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NO_CONTENT);
}
```

## Learning Moment: Why `restTemplate.exchange()`?

The other tests use methods like `getForEntity()` and `postForEntity()`.

So why not `putForEntity()`?

Because `putForEntity()` does not exist on `RestTemplate`.

For PUT requests, `exchange()` is the general-purpose alternative. It lets you provide:

- the URL
- the HTTP method
- the request entity
- the expected response type

For example, these two lines are functionally equivalent:

```java
.exchange("/cashcards/99", HttpMethod.GET, new HttpEntity<>(null), String.class);
```

```java
.getForEntity("/cashcards/99", String.class);
```

## Understand the Test

First, create the request entity:

```java
HttpEntity<CashCard> request = new HttpEntity<>(cashCardUpdate);
```

Then send the PUT request with the updated Cash Card data:

```java
.exchange("/cashcards/99", HttpMethod.PUT, request, Void.class);
```

## Why `204 NO_CONTENT`?

The test expects:

```java
HttpStatus.NO_CONTENT
```

`204 NO_CONTENT` means the update succeeded and no response body is needed.

## Run the Tests

Use:

```sh
./gradlew test
```

## Expected Failure

Until the PUT endpoint exists, the new test should fail like this:

```text
CashCardApplicationTests > shouldUpdateAnExistingCashCard() FAILED
 org.opentest4j.AssertionFailedError:
 expected: 204 NO_CONTENT
  but was: 403 FORBIDDEN
```

## Why It Fails

There is no controller handler for `PUT /cashcards/99` yet.

Spring Security handles that missing endpoint by rejecting the request with `403 FORBIDDEN`.

## Next Step

Implement the controller endpoint for `PUT /cashcards/{id}` so the update succeeds and returns `204 NO_CONTENT`.

## 3. Implement `@PutMapping` in the Controller

Following the same pattern as the other controller handlers, add a minimal `PUT` endpoint first.

Edit `src/main/java/example/cashcard/CashCardController.java` and add:

```java
@PutMapping("/{requestedId}")
private ResponseEntity<Void> putCashCard(@PathVariable Long requestedId, @RequestBody CashCard cashCardUpdate) {
    // just return 204 NO CONTENT for now.
    return ResponseEntity.noContent().build();
}
```

## Understand the Minimal Controller Endpoint

- `@PutMapping("/{requestedId}")` handles `PUT /cashcards/{id}` requests.
- `@PathVariable Long requestedId` captures the target Cash Card ID from the URL.
- `@RequestBody CashCard cashCardUpdate` contains the updated Cash Card data from the request body.
- `ResponseEntity.noContent().build()` returns `204 NO_CONTENT` for now, which is enough to satisfy the first test.

## Run the Tests Again

Use:

```sh
./gradlew test
```

## Expected Result

At this point, the update test should pass:

```text
CashCardApplicationTests > shouldUpdateAnExistingCashCard() PASSED
BUILD SUCCESSFUL
```

## Why This Is Not Enough Yet

The endpoint now returns the correct status code, but it still does not update the Cash Card in the database.

The test passes only because it checks the HTTP response code, not the stored data.

## 4. Enhance the Test to Verify a Successful Update

Now strengthen the test so it verifies that Cash Card `99` was actually updated.

Update `shouldUpdateAnExistingCashCard()` to:

```java
@Test
@DirtiesContext
void shouldUpdateAnExistingCashCard() {
    CashCard cashCardUpdate = new CashCard(null, 19.99, null);
    HttpEntity<CashCard> request = new HttpEntity<>(cashCardUpdate);
    ResponseEntity<Void> response = restTemplate
            .withBasicAuth("sarah1", "abc123")
            .exchange("/cashcards/99", HttpMethod.PUT, request, Void.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NO_CONTENT);

    ResponseEntity<String> getResponse = restTemplate
            .withBasicAuth("sarah1", "abc123")
            .getForEntity("/cashcards/99", String.class);
    assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
    DocumentContext documentContext = JsonPath.parse(getResponse.getBody());
    Number id = documentContext.read("$.id");
    Double amount = documentContext.read("$.amount");
    assertThat(id).isEqualTo(99);
    assertThat(amount).isEqualTo(19.99);
}
```

## Understand the Test Update

This second `GET` request fetches Cash Card `99` again after the `PUT`.

These assertions verify:

- the same Cash Card was retrieved
- its amount changed from `123.45` to `19.99`

## Run the Tests

Use:

```sh
./gradlew test
```

## Expected Failure

The stronger test should now fail like this:

```text
CashCardApplicationTests > shouldUpdateAnExistingCashCard() FAILED
 org.opentest4j.AssertionFailedError:
 expected: 19.99
  but was: 123.45
```

## Why It Fails

The controller returns `204 NO_CONTENT`, but it still does not persist the updated amount.

So the follow-up `GET` still returns the original Cash Card data.

## 5. Update `CashCardController` to Perform the Data Update

Now implement the actual update logic, while ensuring that only the authenticated owner can update a Cash Card.

Replace the minimal `PUT` method with:

```java
@PutMapping("/{requestedId}")
private ResponseEntity<Void> putCashCard(@PathVariable Long requestedId,
        @RequestBody CashCard cashCardUpdate,
        Principal principal) {
    CashCard cashCard = cashCardRepository.findByIdAndOwner(requestedId, principal.getName());
    CashCard updatedCashCard = new CashCard(cashCard.id(), cashCardUpdate.amount(), principal.getName());
    cashCardRepository.save(updatedCashCard);
    return ResponseEntity.noContent().build();
}
```

## Understand the Controller Update

`Principal principal` is provided automatically by Spring Security and identifies the authenticated user.

This line scopes the lookup to the owner:

```java
cashCardRepository.findByIdAndOwner(requestedId, principal.getName());
```

That ensures only the logged-in owner can update the requested Cash Card.

Next, create a new `CashCard` instance with:

- the existing ID
- the updated amount from the request
- the authenticated owner name

Then save it:

```java
cashCardRepository.save(updatedCashCard);
```

## Run the Tests One More Time

Use:

```sh
./gradlew test
```

## Expected Result

Now the update test should pass:

```text
CashCardApplicationTests > shouldUpdateAnExistingCashCard() PASSED
BUILD SUCCESSFUL
```

## What You Have Now

At this point, the application supports updating an existing Cash Card with `PUT`, and the update is scoped to the authenticated owner.

## Next Step

Test the security edge cases:

- updating a Cash Card that does not exist
- updating a Cash Card that is owned by someone else

## Questions From This Lab

### Question: How does `Principal` work here?

You are correct that the test is sending Basic Authentication:

```java
.withBasicAuth("sarah1", "abc123")
```

That means the HTTP request includes an `Authorization` header. With Basic Auth, that header contains the username and password in encoded form.

Spring Security processes that header before the request reaches your controller:

1. It reads the credentials from the request.
2. It authenticates the user.
3. If authentication succeeds, it creates an authenticated security context for this request.
4. The controller method parameter `Principal principal` is then populated from that authenticated user.

So in this lab, `principal` is not a special hidden token passed directly to your controller method.

It is the authenticated user identity that Spring Security has already resolved from the incoming request.

When you call:

```java
principal.getName()
```

you get the username of the authenticated user, such as `sarah1`.

### Does `Principal` decide who can act on the Cash Card?

Not by itself.

`Principal` tells you who the authenticated user is.

Your code then uses that identity to enforce authorization:

```java
cashCardRepository.findByIdAndOwner(requestedId, principal.getName());
```

That line means:

- find the Cash Card with the requested ID
- but only if it belongs to the currently authenticated user

So the security rule is not just “the user is logged in.”

It is also “the logged-in user must be the owner of this Cash Card.”

### Is there a token we cannot see?

Not in this specific Basic Auth flow.

In this lab:

- the client sends username and password on each request
- Spring Security authenticates that request
- Spring exposes the authenticated identity as `Principal`

In other systems, such as JWT or OAuth2, the request may contain a bearer token instead, and Spring Security can still build a `Principal` from that token.

But for this lab, the important point is:

`Principal` is the authenticated identity for the current request, and your code uses it to enforce ownership-based authorization.

### Question: Is `withBasicAuth(...)` basically passing a user session into the app?

Not exactly.

`withBasicAuth("sarah1", "abc123")` tells `TestRestTemplate` to send an HTTP `Authorization` header using Basic Authentication.

So the request includes credentials, not a session object directly.

The flow looks like this:

1. The test sends the HTTP request with the `Authorization` header.
2. Spring Security intercepts the request before the controller method runs.
3. Spring Security reads the credentials from that header.
4. If the credentials are valid, Spring Security authenticates the user.
5. Spring Security creates the authenticated identity for this request.
6. Spring then injects that identity into your controller as `Principal principal`.

So when people say the request has not reached the controller yet, they mean:

- the request has reached the application
- but the security layer runs first
- only after successful authentication does the controller method execute

If authentication fails, the controller method is never called.

### So what is passed to the controller?

Not a raw username/password pair.

Also not a separate hidden token in this Basic Auth lab.

What the controller receives is the authenticated user identity, exposed as:

```java
Principal principal
```

That is why this works:

```java
principal.getName()
```

It returns the username Spring Security authenticated for the current request.

### Where do authentication and authorization split?

They are related, but they are not the same thing.

Authentication answers:

- who is making this request?

Authorization answers:

- is this authenticated user allowed to do this action?

In this lab:

- Spring Security handles authentication from the Basic Auth header
- your controller and repository logic handle authorization

This line is the authorization check:

```java
cashCardRepository.findByIdAndOwner(requestedId, principal.getName());
```

That means:

- authenticate the user first
- then only allow access to the Cash Card if that authenticated user is also the owner

## 6. Additional Testing and Spring Security's Influence

We've got two remaining scenarios to test:

- attempting to update a Cash Card that does not exist
- attempting to update a Cash Card that is owned by someone else

### Case 1: Attempt to update a Cash Card that does not exist

Add the following test to `CashCardApplicationTests`:

```java
@Test
void shouldNotUpdateACashCardThatDoesNotExist() {
    CashCard unknownCard = new CashCard(null, 19.99, null);
    HttpEntity<CashCard> request = new HttpEntity<>(unknownCard);
    ResponseEntity<Void> response = restTemplate
            .withBasicAuth("sarah1", "abc123")
            .exchange("/cashcards/99999", HttpMethod.PUT, request, Void.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
}
```

Here you attempt to update Cash Card `99999`, which does not exist.

The test expects a generic:

```java
HttpStatus.NOT_FOUND
```

### Run the Tests

Use:

```sh
./gradlew test
```

### Expected Failure

At first, this test fails like this:

```text
CashCardApplicationTests > shouldNotUpdateACashCardThatDoesNotExist() FAILED
 org.opentest4j.AssertionFailedError:
 expected: 404 NOT_FOUND
  but was: 403 FORBIDDEN
```

### Why did we get `403 FORBIDDEN`?

To inspect this more closely, enable additional test output in `build.gradle`:

```groovy
test {
    testLogging {
        // Change to `true` for more verbose test output
        showStandardStreams = true
    }
}
```

Then rerun the tests and look for output like:

```text
CashCardApplicationTests > shouldNotUpdateACashCardThatDoesNotExist() STANDARD_OUT
...
java.lang.NullPointerException: Cannot invoke "example.cashcard.CashCard.id()" because "cashCard" is null
```

### What is happening?

In `CashCardController.putCashCard`, this line can return `null`:

```java
CashCard cashCard = cashCardRepository.findByIdAndOwner(requestedId, principal.getName());
```

If `cashCard` is `null`, then this line throws a `NullPointerException`:

```java
CashCard updatedCashCard = new CashCard(cashCard.id(), cashCardUpdate.amount(), principal.getName());
```

Spring Security then returns a generic `403 FORBIDDEN` instead of exposing the internal failure details.

### Fix the Controller so it does not crash

Update `CashCardController.putCashCard` to return `404 NOT_FOUND` when no existing Cash Card is found:

```java
@PutMapping("/{requestedId}")
private ResponseEntity<Void> putCashCard(@PathVariable Long requestedId,
        @RequestBody CashCard cashCardUpdate,
        Principal principal) {
    CashCard cashCard = cashCardRepository.findByIdAndOwner(requestedId, principal.getName());
    if (cashCard != null) {
        CashCard updatedCashCard = new CashCard(cashCard.id(), cashCardUpdate.amount(), principal.getName());
        cashCardRepository.save(updatedCashCard);
        return ResponseEntity.noContent().build();
    }
    return ResponseEntity.notFound().build();
}
```

### Run the Tests Again

Now the tests should pass because the controller returns `404 NOT_FOUND` instead of crashing.

### Case 2: Attempt to update a Cash Card owned by someone else

Add this test:

```java
@Test
void shouldNotUpdateACashCardThatIsOwnedBySomeoneElse() {
    CashCard kumarsCard = new CashCard(null, 333.33, null);
    HttpEntity<CashCard> request = new HttpEntity<>(kumarsCard);
    ResponseEntity<Void> response = restTemplate
            .withBasicAuth("sarah1", "abc123")
            .exchange("/cashcards/102", HttpMethod.PUT, request, Void.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
}
```

This verifies that `sarah1` is not allowed to update one of `kumar2`'s Cash Cards.

### Run the Tests

Use:

```sh
./gradlew test
```

### Expected Result

This test should pass.

The reason is that the null-check you added in the previous step also handles the “owned by someone else” case. If `findByIdAndOwner(...)` finds nothing for the authenticated user, it returns `null`, and the controller returns `404 NOT_FOUND`.

### Verify why the test passes

As an experiment, temporarily comment out the null-check in the controller:

```java
@PutMapping("/{requestedId}")
private ResponseEntity<Void> putCashCard(@PathVariable Long requestedId,
        @RequestBody CashCard cashCardUpdate,
        Principal principal) {
    CashCard cashCard = cashCardRepository.findByIdAndOwner(requestedId, principal.getName());
    // if (null != cashCard) {
        CashCard updatedCashCard = new CashCard(cashCard.id(), cashCardUpdate.amount(), principal.getName());
        cashCardRepository.save(updatedCashCard);
        return ResponseEntity.noContent().build();
    // }
    // return ResponseEntity.notFound().build();
}
```

Run:

```sh
./gradlew test
```

You should see both tests fail with:

```text
expected: 404 NOT_FOUND
 but was: 403 FORBIDDEN
```

This confirms that both failures were caused by the underlying `NullPointerException`.

### Undo the experiment

Restore the null-check in `CashCardController.putCashCard`:

```java
@PutMapping("/{requestedId}")
private ResponseEntity<Void> putCashCard(@PathVariable Long requestedId,
        @RequestBody CashCard cashCardUpdate,
        Principal principal) {
    CashCard cashCard = cashCardRepository.findByIdAndOwner(requestedId, principal.getName());
    if (null != cashCard) {
        CashCard updatedCashCard = new CashCard(cashCard.id(), cashCardUpdate.amount(), principal.getName());
        cashCardRepository.save(updatedCashCard);
        return ResponseEntity.noContent().build();
    }
    return ResponseEntity.notFound().build();
}
```

### Learning Moment: Controversy Exists

You may ask whether it is worth keeping two tests that currently behave the same with respect to one code change.

One reason to keep both tests is that they still describe two distinct scenarios:

- a Cash Card record does not exist
- a Cash Card exists, but belongs to someone else

Today, both cases return `404 NOT_FOUND`.

In the future, a change in repository or controller behavior might make them diverge. Keeping both tests preserves the intent of each scenario.

## More Questions From This Lab

### Question: How is Kumar's card ownership compared to what we send with Basic Auth?

The comparison is based on the authenticated username and the `owner` value stored on the Cash Card record.

These are two separate pieces of data:

- the authenticated user comes from Spring Security
- the Cash Card owner comes from the database row

When the test uses:

```java
.withBasicAuth("sarah1", "abc123")
```

Spring Security authenticates the request as `sarah1`.

That means:

```java
principal.getName()
```

returns:

```java
"sarah1"
```

Then the controller does this lookup:

```java
cashCardRepository.findByIdAndOwner(requestedId, principal.getName());
```

So if the request is trying to update Cash Card `102`, the lookup effectively becomes:

- find Cash Card `102`
- but only if its `owner` is `"sarah1"`

### Where does `kumar2` come from?

`kumar2` is present in the test data, not in the authenticated user configuration.

The seeded test database includes:

```sql
INSERT INTO CASH_CARD(ID, AMOUNT, OWNER) VALUES (102, 200.00, 'kumar2');
```

So card `102` belongs to `kumar2`.

But your security configuration only defines login users like:

- `sarah1`
- `hank-owns-no-cards`

That means `kumar2` exists in the data as a card owner, even though `kumar2` is not currently configured as a login user in this lab.

### So how does the comparison work if `kumar2` is not a login user?

That is fine for this specific negative test.

The test is not trying to log in as `kumar2`.

It is only trying to prove that:

- `sarah1` is authenticated
- card `102` is owned by someone else
- therefore `sarah1` should not be allowed to update it

Since the query checks both the ID and owner, no record matches:

- requested ID = `102`
- authenticated owner = `"sarah1"`
- actual owner in the database = `"kumar2"`

So the repository returns `null`, and the controller returns `404 NOT_FOUND`.

### What would be needed to test Kumar updating his own card?

You would need both:

- a Cash Card owned by `kumar2`
- a configured authenticated user named `kumar2`

Right now, the first exists, but the second does not.

So in this lab, `kumar2` is only being used as ownership data for a negative authorization test.

## 7. Refactor the Controller Code

Let's reinforce the Red, Green, Refactor development loop.

At this point, you've completed several tests around updating an existing Cash Card.

Now look for opportunities to simplify the controller and reduce duplication without changing behavior.

### Simplify the Code

#### Remove the `Optional`

Since `CashCardController.findById` is not really taking advantage of `Optional`, and the other controller methods do not use it either, simplify the code by removing it.

Update `CashCardController.findById` to:

```java
// remove the unused Optional import if present
// import java.util.Optional;

@GetMapping("/{requestedId}")
private ResponseEntity<CashCard> findById(@PathVariable Long requestedId, Principal principal) {
    CashCard cashCard = cashCardRepository.findByIdAndOwner(requestedId, principal.getName());
    if (cashCard != null) {
        return ResponseEntity.ok(cashCard);
    } else {
        return ResponseEntity.notFound().build();
    }
}
```

### Run the Tests

Use:

```sh
./gradlew test
```

### Expected Result

Because functionality has not changed, the tests should still pass:

```text
BUILD SUCCESSFUL
```

### Reduce Code Duplication

Notice that both `CashCardController.findById` and `CashCardController.putCashCard` retrieve a Cash Card using the same inputs:

- the requested Cash Card ID
- the authenticated `Principal`

That duplication is a good refactoring target.

### Create a Shared `findCashCard` Method

Add this helper method to `CashCardController`:

```java
private CashCard findCashCard(Long requestedId, Principal principal) {
    return cashCardRepository.findByIdAndOwner(requestedId, principal.getName());
}
```

### Update `CashCardController.findById`

Now change `findById` to use the shared helper:

```java
@GetMapping("/{requestedId}")
private ResponseEntity<CashCard> findById(@PathVariable Long requestedId, Principal principal) {
    CashCard cashCard = findCashCard(requestedId, principal);
    if (cashCard != null) {
        return ResponseEntity.ok(cashCard);
    } else {
        return ResponseEntity.notFound().build();
    }
}
```

### Run the Tests Again

Use:

```sh
./gradlew test
```

### Expected Result

No behavior has changed, so the tests should still pass:

```text
BUILD SUCCESSFUL
```

### Update `CashCardController.putCashCard`

Next, use the same helper method in `putCashCard`:

```java
@PutMapping("/{requestedId}")
private ResponseEntity<Void> putCashCard(@PathVariable Long requestedId,
        @RequestBody CashCard cashCardUpdate,
        Principal principal) {
    CashCard cashCard = findCashCard(requestedId, principal);
    if (null != cashCard) {
        CashCard updatedCashCard = new CashCard(cashCard.id(), cashCardUpdate.amount(), principal.getName());
        cashCardRepository.save(updatedCashCard);
        return ResponseEntity.noContent().build();
    }
    return ResponseEntity.notFound().build();
}
```

### Run the Tests One More Time

Use:

```sh
./gradlew test
```

### Expected Result

Again, there is no functional change, so the tests should still pass:

```text
BUILD SUCCESSFUL
```

### What This Refactor Achieved

You improved the controller in two ways:

- simplified `findById` by removing unnecessary `Optional` usage
- reduced duplication by extracting `findCashCard(...)`

That gives you one place to change the Cash Card lookup behavior as the controller evolves.

## Even More Questions From This Lab

### Question: Why did the lab say `Optional` was not serving much purpose here?

Because in this controller method, `Optional` was only wrapping a value and then immediately being unwrapped again.

The repository method returns a plain `CashCard`, not an `Optional<CashCard>`:

```java
CashCard findByIdAndOwner(Long id, String owner);
```

So the earlier controller code was effectively doing this:

```java
CashCard cashCard = cashCardRepository.findByIdAndOwner(requestedId, principal.getName());
if (cashCard != null) {
    return ResponseEntity.ok(cashCard);
} else {
    return ResponseEntity.notFound().build();
}
```

But written in a more indirect way:

```java
Optional<CashCard> cashCardOptional =
        Optional.ofNullable(cashCardRepository.findByIdAndOwner(requestedId, principal.getName()));

if (cashCardOptional.isPresent()) {
    return ResponseEntity.ok(cashCardOptional.get());
} else {
    return ResponseEntity.notFound().build();
}
```

Both versions are checking the exact same thing:

- did we get a Cash Card?
- or did we get nothing?

### So was `Optional` redundant?

In this specific case, yes, mostly.

It was not wrong, but it was not adding much value because:

- the repository did not return `Optional`
- the controller manually wrapped the nullable result
- the controller immediately called `isPresent()` and `get()`

That means `Optional` was basically being used as a verbose null check.

### Does `Optional` prevent `NullPointerException` automatically?

No.

`Optional` only helps if you use it in a way that makes absence explicit and safer to work with.

In this lab, the code was still doing manual presence checks, so `Optional` was not providing a meaningful advantage over:

```java
if (cashCard != null)
```

### When would `Optional` be more useful?

`Optional` becomes more useful when:

- the repository method itself returns `Optional<CashCard>`
- you use methods like `map`, `orElse`, or `orElseGet`
- it makes the API clearer by explicitly modeling “this value may be missing”

So the lab's point was not that `Optional` is bad.

The point was that in this exact controller method, it was not doing anything better than a plain null check, so removing it made the code simpler.

## 8. Summary

In this lab, you learned how to implement an HTTP `PUT` endpoint that allows an authenticated, authorized owner to update their Cash Card.

You also learned how Spring Security automatically manages error handling by Spring Web to help ensure that security-sensitive information is not accidentally revealed when a controller encounters an error.

In addition, you reinforced your understanding and use of the Red, Green, Refactor development cycle by refactoring the controller code without changing its functionality.
