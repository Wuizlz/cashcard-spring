# Delete Steps

## Overview

In the associated lesson, you learned about implementing the Delete operation in a REST API. In this lab, we'll implement hard delete in our Cash Card API, using the API specifications we defined in the associated lesson.

## Credentials for Test User Kumar

If you've taken previous labs in this course you'll notice the following changes to the codebase, which we've made on your behalf, to make this lab easier to understand and complete.

We've added credentials for the user `kumar2` to the `testOnlyUsers` bean in `src/main/java/example/cashcard/SecurityConfig.java`.

Let's implement the Delete endpoint.

## 2. Test the Happy Path

Let's start with the simplest happy path: successfully deleting a `CashCard` which exists.

We need the Delete endpoint to return the `204 NO CONTENT` status code.

### Write the Test

Add the following test method to `src/test/java/example/cashcard/CashCardApplicationTests.java`:

```java
@Test
@DirtiesContext
void shouldDeleteAnExistingCashCard() {
    ResponseEntity<Void> response = restTemplate
            .withBasicAuth("sarah1", "abc123")
            .exchange("/cashcards/99", HttpMethod.DELETE, null, Void.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NO_CONTENT);
}
```

Notice that we've added the `@DirtiesContext` annotation. We'll add this annotation to all tests which change the data. If we don't, then these tests could affect the result of other tests in the file.

### Why Not Use `RestTemplate.delete()`?

Notice that we're using `RestTemplate.exchange()` even though `RestTemplate` supplies a method that looks like we could use: `RestTemplate.delete()`. However, let's look at the signature:

```java
public class RestTemplate ... {
    public void delete(String url, Object... uriVariables)
}
```

The other methods we've been using, such as `getForEntity()` and `exchange()`, return a `ResponseEntity`, but `delete()` doesn't. Instead, it's a `void` method. Why is this?

The Spring Web framework supplies the `delete()` method as a convenience, but it comes with some assumptions:

- A response to a `DELETE` request will have no body.
- The client shouldn't care what the response code is unless it's an error, in which case it'll throw an exception.

Given those assumptions, no return value is needed from `delete()`.

But the second assumption makes `delete()` unsuitable for us: we need the `ResponseEntity` in order to assert on the status code. So, we won't use the convenience method, but rather the more general method: `exchange()`.

### Run the Tests

As always, we'll use `./gradlew test` to run the tests.

```bash
[~/exercises] $ ./gradlew test
...
CashCardApplicationTests > shouldDeleteAnExistingCashCard() FAILED
    org.opentest4j.AssertionFailedError:
    expected: 204 NO_CONTENT
    but was: 403 FORBIDDEN
```

The test failed because the `DELETE /cashcards/99` request returned a `403 FORBIDDEN`.

At this point you probably expected this result: Spring Security returns a `403` response for any endpoint which isn't mapped.

We need to implement the Controller method.

### Implement the Delete Endpoint in the Controller

Add the following method to the `CashCardController` class:

```java
@DeleteMapping("/{id}")
private ResponseEntity<Void> deleteCashCard(@PathVariable Long id) {
    return ResponseEntity.noContent().build();
}
```

### Run the Tests Again

```bash
[~/exercises] $ ./gradlew test
...
BUILD SUCCESSFUL in 8s
```

They pass.

Does that mean that we're done? Not yet.

We haven't written the code to actually delete the item. Let's do that next.

We'll write the test first, of course.

### Test That We're Actually Deleting the `CashCard`

Add the following assertions to the `shouldDeleteAnExistingCashCard()` test:

```java
@Test
@DirtiesContext
void shouldDeleteAnExistingCashCard() {
    ResponseEntity<Void> response = restTemplate
            .withBasicAuth("sarah1", "abc123")
            .exchange("/cashcards/99", HttpMethod.DELETE, null, Void.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NO_CONTENT);

    // Add the following code:
    ResponseEntity<String> getResponse = restTemplate
            .withBasicAuth("sarah1", "abc123")
            .getForEntity("/cashcards/99", String.class);
    assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
}
```

We want to test that the deleted Cash Card is actually deleted, so we try to `GET` it and assert that the result code is `404 NOT FOUND`.

### Run the Test

Does it pass? Of course not.

```text
CashCardApplicationTests > shouldDeleteAnExistingCashCard() FAILED
    org.opentest4j.AssertionFailedError:
    expected: 404 NOT_FOUND
    but was: 200 OK
```

What do we need to do to make the test pass? Write some code to delete the record.

Let's go.

## 3. Implement the DELETE Endpoint

Now we need to write a Controller method which will be called when we send a `DELETE` request with the proper URI.

### Add Code to the Controller to Delete the Record

Change the `CashCardController.deleteCashCard()` method:

```java
@DeleteMapping("/{id}")
private ResponseEntity<Void> deleteCashCard(@PathVariable Long id) {
    cashCardRepository.deleteById(id); // Add this line
    return ResponseEntity.noContent().build();
}
```

The change is straightforward:

- We use `@DeleteMapping` with the `"{id}"` parameter, which Spring Web matches to the `id` method parameter.
- `CashCardRepository` already has the method we need: `deleteById()`; it's inherited from `CrudRepository`.

### Run the Tests

Run the tests, and watch them pass:

```bash
[~/exercises] $ ./gradlew test
...
BUILD SUCCESSFUL in 8s
```

Great, what to do next?

## 4. Test Case: The Cash Card Doesn't Exist

Our contract states that we should return `404 NOT FOUND` if we try to delete a card that doesn't exist.

### Write the Test

Add the following test method to `CashCardApplicationTests`:

```java
@Test
void shouldNotDeleteACashCardThatDoesNotExist() {
    ResponseEntity<Void> deleteResponse = restTemplate
            .withBasicAuth("sarah1", "abc123")
            .exchange("/cashcards/99999", HttpMethod.DELETE, null, Void.class);
    assertThat(deleteResponse.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
}
```

### Run the Tests

```text
CashCardApplicationTests > shouldNotDeleteACashCardThatDoesNotExist() FAILED
    org.opentest4j.AssertionFailedError:
    expected: 404 NOT_FOUND
    but was: 204 NO_CONTENT
```

No surprise here.

We need to enforce the security of our app by checking that the user trying to delete a card is the owner.

Spring Security does a lot of things for us, but this isn't one of them.

## 5. Enforce Ownership

We need to check whether the record exists. If not, we should not delete the Cash Card, and return `404 NOT FOUND`.

### Update `CashCardController.deleteCashCard`

Make the following changes to `CashCardController.deleteCashCard`:

```java
private ResponseEntity<Void> deleteCashCard(
        @PathVariable Long id,
        Principal principal // Add Principal to the parameter list
    ) {
    // Add the following 3 lines:
    if (!cashCardRepository.existsByIdAndOwner(id, principal.getName())) {
        return ResponseEntity.notFound().build();
    }
...
}
```

Let's be sure to add the `Principal` parameter.

We're using the `Principal` in order to check whether the Cash Card exists, and at the same time, enforce ownership.

### Add `existsByIdAndOwner()` to the Repository

Also, let's add the new method `existsByIdAndOwner()` to `CashCardRepository`:

```java
interface CashCardRepository extends CrudRepository<CashCard, Long>, PagingAndSortingRepository<CashCard, Long> {
    ...
    boolean existsByIdAndOwner(Long id, String owner);
    ...
}
```

### Understand the Repository Code

We added logic to the Controller method to check whether the Cash Card ID in the request actually exists in the database. The method we'll use is `CashCardRepository.existsByIdAndOwner(id, username)`.

This is another case where Spring Data will generate the implementation of this method as long as we add it to the Repository.

So why not just use the `findByIdAndOwner()` method and check whether it returns `null`? We could absolutely do that. But such a call would return extra information, namely the content of the Cash Card retrieved, so we'd like to avoid that extra complexity.

If you'd rather not use the `existsByIdAndOwner()` method, that's ok. You may choose to use `findByIdAndOwner()`. The test result will be the same.

### Watch the Test Pass

Let's run the test and, no big surprise, the test passes:

```bash
[~/exercises] $ ./gradlew test
...
BUILD SUCCESSFUL in 8s
```

## 6. Refactor

At this point, we have an opportunity to practice the Red, Green, Refactor process. We've already done Red, the failing test, and Green, the passing test. Now we can ask ourselves: should we refactor anything?

Here's the body of our `CashCardController.deleteCashCard` method:

```java
...
if (!cashCardRepository.existsByIdAndOwner(id, principal.getName())) {
    return ResponseEntity.notFound().build();
}
cashCardRepository.deleteById(id);
return ResponseEntity.noContent().build();
```

You might find the following version, which is logically equivalent but slightly simpler, to be easier to read:

```java
if (cashCardRepository.existsByIdAndOwner(id, principal.getName())) {
    cashCardRepository.deleteById(id);
    return ResponseEntity.noContent().build();
}
return ResponseEntity.notFound().build();
```

The differences are slight, but removing a not-operator (`!`) from an `if` statement often makes for easier-to-read code, and readability is important.

If you do find the second version easier to read and understand, then replace the existing code with the new version.

### Run the Tests Again

They still pass:

```bash
[~/exercises] $ ./gradlew test
...
BUILD SUCCESSFUL in 8s
```

## 7. Hide Unauthorized Records

At this point, you may ask yourself, "Are we done?" You're the best person to answer that question. If you want, take a couple minutes to refresh yourself with the accompanying lesson in order to see if we've tested and implemented every aspect of the API contract for `DELETE`.

OK, that was time well-spent, wasn't it? That's right: there's one more case that we have yet to test. What if the user attempts to delete a Cash Card owned by someone else? We decided in the associated lesson that the response should be `404 NOT FOUND`.

That's enough information for us to write a test for that case.

### Add the Test

In `CashCardApplicationTests.java`, add the following test method at the end of the class:

```java
@Test
void shouldNotAllowDeletionOfCashCardsTheyDoNotOwn() {
    ResponseEntity<Void> deleteResponse = restTemplate
            .withBasicAuth("sarah1", "abc123")
            .exchange("/cashcards/102", HttpMethod.DELETE, null, Void.class);
    assertThat(deleteResponse.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
}
```

### Run the Test

Do you think the test will pass? Take a moment to predict the outcome, then run the test.

```bash
[~/exercises] $ ./gradlew test
...
BUILD SUCCESSFUL in 8s
```

They all passed. That's great news.

We've written a test for a specific case in our API.
The test passed without any code changes.

Now, let's consider a question which may have occurred to you: why do we need that test, since it passes without having to make any code changes? Isn't the purpose of TDD to use tests to guide the implementation of the application? If that's the case, why did we bother to write that test?

### Why This Test Still Matters

Yes, that is one of many benefits that TDD provides: a process to guide the creation of code in order to arrive at a desired outcome.

Tests themselves, though, have another purpose, separate from TDD: tests are a powerful safety net to enforce correctness. Since the test you just wrote covers a different case than those already written, it provides value. If someone were to make a code change that caused this new test to fail, then you'll have caught the error before it could become an issue.

### One More Test

Shouldn't we also test that the record we tried to delete still exists in the database, and that it didn't get deleted?

Yes, that's a valid test.

Add the following code to the test method to verify that the record you tried unsuccessfully to delete is still there:

```java
void shouldNotAllowDeletionOfCashCardsTheyDoNotOwn() {
 ...
    // Add this code at the end of the test method:
    ResponseEntity<String> getResponse = restTemplate
            .withBasicAuth("kumar2", "xyz789")
            .getForEntity("/cashcards/102", String.class);
    assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
}
```

Do you think the test will pass? Of course it will.

### Run the Tests One Last Time

Please run all the tests and ensure that they pass.

```bash
[~/exercises] $ ./gradlew test
...
BUILD SUCCESSFUL in 6s
```

We're the best!

## 8. Summary

In this lab, you implemented a RESTful Delete endpoint which doesn't "leak" security information to would-be attackers. You also had a chance to do a small, but useful refactoring to practice the Red, Green, Refactor process.

This is the last of the CRUD operations to implement in the API, which brings us to a successful conclusion of our technical work. Congratulations!

## 9. Additional Question

### Why Does an Unauthorized Delete Return `404 NOT FOUND`?

Yes. The unauthorized request is handled by the `if` statement in the `@DeleteMapping` controller method:

```java
if (!cashCardRepository.existsByIdAndOwner(id, principal.getName())) {
    return ResponseEntity.notFound().build();
}
```

The call to `existsByIdAndOwner(id, principal.getName())` checks two things at the same time:

- A `CashCard` with that `id` exists.
- The authenticated user, identified by `principal.getName()`, is the owner of that card.

If the authenticated user's name does not match the owner of the card, the condition fails and the controller returns `404 NOT FOUND`.

This is intentional. Returning `404 NOT FOUND` instead of exposing more detail helps avoid leaking information about whether a record exists for another user.
