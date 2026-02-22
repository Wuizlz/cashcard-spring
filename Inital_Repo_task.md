# Initial Repository Task - Review Current Data Management Pattern

This step reviews the current data management pattern in the Cash Card API and prepares for a refactor to a Repository and database.

**Review the Controller**
- Locate the hard-coded data logic in `src/main/java/example/cashcard/CashCardController.java`.
- Example pattern:
  - `if (requestedId.equals(99L)) {`
  - `CashCard cashCard = new CashCard(99L, 123.45);`
  - `return ResponseEntity.ok(cashCard);`
- This is data management and should not live in the web controller.

**Review the Tests**
- Inspect `src/test/java/example/cashcard/CashCardApplicationTests.java`.
- Note that the tests assert data outcomes but do not care how the data is created or stored.
- This decoupling is intentional and enables refactoring without changing tests.

**Prepare to Refactor**
- Goal: move data management out of the controller and into a Repository backed by a database.
- Approach: refactor implementation while keeping inputs/outputs/behavior unchanged.
- Expect tests to fail during refactor and pass again once the repository-backed implementation is complete.

**Concepts**
- Separation of Concerns: web traffic and data access belong in different layers.
- Red, Green, Refactor: tests guide the implementation, then validate refactoring.

# Step 3 - Add Spring Data Dependencies

This step adds Spring Data JDBC and H2 database dependencies to the project.

**Update `build.gradle`**
Add the two dependencies below in the `dependencies` block:

```groovy
dependencies {
   implementation 'org.springframework.boot:spring-boot-starter-web'
   testImplementation 'org.springframework.boot:spring-boot-starter-test'

   // Add the two dependencies below
   implementation 'org.springframework.data:spring-data-jdbc'
   implementation 'com.h2database:h2'
}
```

**Understand the dependencies**
- `implementation 'org.springframework.data:spring-data-jdbc'`
  - Spring Data JDBC is a simple, opinionated ORM that integrates with relational databases.
- `implementation 'com.h2database:h2'`
  - H2 is an in-memory SQL database that works well with Spring Data JDBC.

**Run the tests (expect success)**
```sh
./gradlew test
```

Expected success excerpt:
```
BUILD SUCCESSFUL in 4s
```


# Step 4 - Create the `CashCardRepository`

This step introduces a Spring Data repository for `CashCard` and configures the `@Id` field.

**Create the repository**
Create `src/main/java/example/cashcard/CashCardRepository.java`:

```java
package example.cashcard;

import org.springframework.data.repository.CrudRepository;

interface CashCardRepository extends CrudRepository {
}
```

**Run the tests (expect failure)**
```sh
./gradlew test
```

Expected failure excerpt:
```
CashCardApplicationTests > shouldNotReturnACashCardWithAnUnknownId() FAILED
 java.lang.IllegalStateException: Failed to load ApplicationContext for ...

Caused by:
java.lang.IllegalArgumentException: Could not resolve domain type of interface example.cashcard.CashCardRepository
```

**Configure the repository domain type**
Update the repository to specify the domain type (`CashCard`) and ID type (`Long`):

```java
interface CashCardRepository extends CrudRepository<CashCard, Long> {
}
```

**Configure the `CashCard` ID**
Annotate the `id` field in `CashCard` with `@Id` so Spring Data knows the identifier:

```java
package example.cashcard;

import org.springframework.data.annotation.Id;

record CashCard(@Id Long id, Double amount) {
}
```

**Run the tests (expect success)**
```sh
./gradlew test
```

Expected success excerpt:
```
BUILD SUCCESSFUL in 4s
```

You may see additional output such as “Shutting down embedded database,” which indicates Spring has started and configured H2 during tests.

# Step 5 - Inject the `CashCardRepository`

This step injects the repository into the controller and briefly demonstrates what happens when dependency injection is broken.

**Inject the repository into the controller**
Update `src/main/java/example/cashcard/CashCardController.java` to accept a `CashCardRepository` via constructor injection:

```java
@RestController
@RequestMapping("/cashcards")
class CashCardController {
   private final CashCardRepository cashCardRepository;

   private CashCardController(CashCardRepository cashCardRepository) {
      this.cashCardRepository = cashCardRepository;
   }
   ...
}
```

**Run the tests (expect success)**
```sh
./gradlew test
```

Expected success excerpt:
```
BUILD SUCCESSFUL in 7s
```

**Learning moment: remove DI temporarily**
Temporarily remove the `CrudRepository` extension:

```java
interface CashCardRepository {
}
```

Compile and observe the failure:
```sh
./gradlew build
```

Expected failure excerpt:
```
org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'example.cashcard.CashCardRepository' available: expected at least 1 bean which qualifies as autowire candidate. Dependency annotations: {}
```

This indicates Spring can’t find a qualifying bean to inject.

**Undo the change**
Restore the repository definition:

```java
interface CashCardRepository extends CrudRepository<CashCard, Long> {
}
```

# Step 6 - Use the `CashCardRepository` for Data Management

This step switches the controller from hard-coded data to repository lookups and prepares for the missing-table failure.

**Update the controller to use `findById`**
Add `Optional` and query the repository:

```java
import java.util.Optional;
...
@GetMapping("/{requestedId}")
private ResponseEntity<CashCard> findById(@PathVariable Long requestedId) {
    Optional<CashCard> cashCardOptional = cashCardRepository.findById(requestedId);
    if (cashCardOptional.isPresent()) {
        return ResponseEntity.ok(cashCardOptional.get());
    } else {
        return ResponseEntity.notFound().build();
    }
}
```

**Run the tests (expect failure)**
```
CashCardApplicationTests > shouldReturnACashCardWhenDataIsSaved() FAILED
   org.opentest4j.AssertionFailedError:
   expected: 200 OK
   but was: 500 INTERNAL_SERVER_ERROR
```

**Increase test output verbosity**
Temporarily update the test logging in `build.gradle`:

```groovy
test {
 testLogging {
     events "passed", "skipped", "failed" //, "standardOut", "standardError"

     showExceptions true
     exceptionFormat "full"
     showCauses true
     showStackTraces true

     // Change from false to true
     showStandardStreams = true
 }
}
```

**Rerun the tests and inspect the failure**
Expected error excerpt:
```
org.h2.jdbc.JdbcSQLSyntaxErrorException: Table "CASH_CARD" not found (this database is empty); SQL statement:
 SELECT "CASH_CARD"."ID" AS "ID", "CASH_CARD"."AMOUNT" AS "AMOUNT" FROM "CASH_CARD" WHERE "CASH_CARD"."ID" = ? [42104-214]
```

This confirms the database/table doesn’t exist yet.

# Step 7 - Configure the Database

This step enables the schema and test data so Spring Data + H2 can create and populate the in-memory database.

**Enable `schema.sql`**
Edit `src/main/resources/schema.sql` and remove the block comment so it becomes:

```sql
CREATE TABLE cash_card
(
   ID     BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
   AMOUNT NUMBER NOT NULL DEFAULT 0
);
```

**Run the tests (expect failure changes)**
After enabling the schema, the 500 error is gone but the test now fails with 404:
```
CashCardApplicationTests > shouldReturnACashCardWhenDataIsSaved() FAILED
 org.opentest4j.AssertionFailedError:
 expected: 200 OK
  but was: 404 NOT_FOUND
```

**Enable `data.sql`**
Edit `src/test/resources/data.sql` and remove the block comment so it becomes:

```sql
INSERT INTO CASH_CARD(ID, AMOUNT) VALUES (99, 123.45);
```

**Run the tests (expect success)**
```sh
./gradlew test
```

Expected success excerpt:
```
BUILD SUCCESSFUL in 7s
```

**Learning moment: main vs test resources**
- `schema.sql` lives in `src/main/resources` because the schema is shared across environments.
- `data.sql` lives in `src/test/resources` because the test data (CashCard 99, Amount 123.45) is fake and should only load for tests.
