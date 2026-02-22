# Step 1 - Write a Spring Boot Test for the GET Endpoint

This step captures the test-first workflow for the GET `/cashcards/{id}` endpoint. The test is expected to fail until the endpoint is implemented.

**Update the test file**
Edit `src/test/java/example/cashcard/CashCardApplicationTests.java` to match the following:

```java
package example.cashcard;

import com.jayway.jsonpath.DocumentContext;
import com.jayway.jsonpath.JsonPath;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class CashCardApplicationTests {
    @Autowired
    TestRestTemplate restTemplate;

    @Test
    void shouldReturnACashCardWhenDataIsSaved() {
        ResponseEntity<String> response = restTemplate.getForEntity("/cashcards/99", String.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

**Understand the test setup**
- `@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)` starts the application on a random port for the test.
- `TestRestTemplate` is injected to make HTTP requests against the running app.
- `getForEntity("/cashcards/99", String.class)` performs the GET request.
- The test asserts the HTTP status is `200 OK`.

**Run the tests (expect failure)**
```sh
./gradlew test
```

Expected failure excerpt:
```
CashCardApplicationTests > shouldReturnACashCardWhenDataIsSaved() FAILED
  org.opentest4j.AssertionFailedError:
  expected: 200 OK
   but was: 404 NOT_FOUND
```

**Why it fails**
The `/cashcards/99` endpoint does not exist yet, so Spring returns `404 NOT_FOUND`. This is the expected test-first failure. Do not implement the endpoint in this step.

# Step 2 - Create a REST Controller (Not Yet Wired)

This step introduces a controller class and a stub handler method. It is still expected to fail because the controller is not annotated as a Spring Web controller, and the handler method is not mapped to a URL.

**Create the controller class**
Create `src/main/java/example/cashcard/CashCardController.java` with this content:

```java
package example.cashcard;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

class CashCardController {
    private ResponseEntity<String> findById() {
        return ResponseEntity.ok("{}");
    }
}
```

**Run the tests again (still expect failure)**
```sh
./gradlew test
```

Expected failure excerpt (unchanged):
```
CashCardApplicationTests > shouldReturnACashCardWhenDataIsSaved() FAILED
  org.opentest4j.AssertionFailedError:
  expected: 200 OK
   but was: 404 NOT_FOUND
```

**Why it still fails**
`CashCardController` is just a plain class, so Spring Web does not register it as a controller. Without `@RestController` (and without a mapped handler), Spring still returns `404 NOT_FOUND` for `/cashcards/99`.

# Step 3 - Add the GET Endpoint

Wire the controller to handle `/cashcards/{id}` requests so the test passes.

**Update the controller annotations**
Replace `src/main/java/example/cashcard/CashCardController.java` with:

```java
package example.cashcard;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/cashcards")
class CashCardController {
    @GetMapping("/{requestedId}")
    private ResponseEntity<String> findById() {
        return ResponseEntity.ok("{}");
    }
}
```

**Run the tests (expect success)**
```sh
./gradlew test
```

Expected success excerpt:
```
BUILD SUCCESSFUL in 6s
```

**Why it passes now**
- `@RestController` registers the class as a Spring Web controller.
- `@RequestMapping("/cashcards")` sets the base path.
- `@GetMapping("/{requestedId}")` wires the handler to `/cashcards/{id}`.

Together, these match the request made by the test, so the status is now `200 OK`.

# Step 4 - Complete the GET Endpoint

This step deepens the test to verify response content, then iteratively updates the controller to return the correct values. Do not change the code in this step unless you are following along in order.

**Update the test to check `id` is present**
Add these assertions after the status check in `src/test/java/example/cashcard/CashCardApplicationTests.java`:

```java
assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);

DocumentContext documentContext = JsonPath.parse(response.getBody());
Number id = documentContext.read("$.id");
assertThat(id).isNotNull();
```

**Run the tests (expect failure)**
```sh
./gradlew test
```

Expected failure excerpt:
```
CashCardApplicationTests > shouldReturnACashCardWhenDataIsSaved() FAILED
    com.jayway.jsonpath.PathNotFoundException: No results for path: $['id']
```

**Return a Cash Card (intentionally wrong ID)**
Update the controller to return a `CashCard` with an incorrect ID (for now):

```java
@GetMapping("/{requestedId}")
private ResponseEntity<CashCard> findById() {
   CashCard cashCard = new CashCard(1000L, 0.0);
   return ResponseEntity.ok(cashCard);
}
```

**Run the tests (expect success, but incorrect data)**
```sh
./gradlew test
```

**Update the test to assert the correct ID**
Replace the `id` assertion with:

```java
DocumentContext documentContext = JsonPath.parse(response.getBody());
Number id = documentContext.read("$.id");
assertThat(id).isEqualTo(99);
```

**Run the tests (expect failure)**
```
expected: 99
 but was: 1000
```

**Fix the controller to return the correct ID**

```java
@GetMapping("/{requestedId}")
private ResponseEntity<CashCard> findById() {
   CashCard cashCard = new CashCard(99L, 0.0);
   return ResponseEntity.ok(cashCard);
}
```

**Run the tests (expect success)**
```sh
./gradlew test
```

**Add the amount assertion**
Extend the test to validate the JSON amount:

```java
DocumentContext documentContext = JsonPath.parse(response.getBody());
Number id = documentContext.read("$.id");
assertThat(id).isEqualTo(99);

Double amount = documentContext.read("$.amount");
assertThat(amount).isEqualTo(123.45);
```

**Run the tests (expect failure)**
```
expected: 123.45
 but was: 0.0
```

**Return the correct amount**
Update the controller to return the correct amount:

```java
@GetMapping("/{requestedId}")
private ResponseEntity<CashCard> findById() {
   CashCard cashCard = new CashCard(99L, 123.45);
   return ResponseEntity.ok(cashCard);
}
```

**Run the tests (expect success)**
```sh
./gradlew test
```

Expected success excerpt:
```
BUILD SUCCESSFUL in 6s
```

# Step 5 - Use `@PathVariable`

This step adds a negative-case test and uses the path variable in the controller to return `404 NOT_FOUND` for unknown IDs.

**Add a new test method**
Append this test to `src/test/java/example/cashcard/CashCardApplicationTests.java`:

```java
@Test
void shouldNotReturnACashCardWithAnUnknownId() {
  ResponseEntity<String> response = restTemplate.getForEntity("/cashcards/1000", String.class);

  assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
  assertThat(response.getBody()).isBlank();
}
```

**Run the tests (expect failure)**
```
expected: 404 NOT_FOUND
 but was: 200 OK
```

**Add `@PathVariable` to the handler**
Update the controller signature so Spring injects the ID from the path:

```java
@GetMapping("/{requestedId}")
private ResponseEntity<CashCard> findById(@PathVariable Long requestedId) {
    CashCard cashCard = new CashCard(99L, 123.45);
    return ResponseEntity.ok(cashCard);
}
```

**Use the path variable to gate the response**
Return the cash card only for ID `99`, otherwise return `404`:

```java
@GetMapping("/{requestedId}")
private ResponseEntity<CashCard> findById(@PathVariable Long requestedId) {
    if (requestedId.equals(99L)) {
        CashCard cashCard = new CashCard(99L, 123.45);
        return ResponseEntity.ok(cashCard);
    } else {
        return ResponseEntity.notFound().build();
    }
}
```

**Run the tests (expect success)**
```sh
./gradlew test
```

Expected success excerpt:
```
BUILD SUCCESSFUL in 6s
```
