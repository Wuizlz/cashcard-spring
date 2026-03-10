# Security Steps

## Understand our Security Requirements

Who should be allowed to manage any given Cash Card?

In this domain, the user who created a Cash Card is considered the owner of that card. That user is the "card owner".

Only the card owner should be allowed to view or update a Cash Card.

The security logic is:

- If the user is authenticated
- And they are authorized as a card owner
- And they own the requested Cash Card
- Then the request should be allowed

Users must not be allowed to access Cash Cards they do not own.

## Review update from Previous Lab

In this lab, the Family Cash Card API will be secured so access to a given Cash Card is limited to that card's owner.

To support that, the application now includes the concept of an owner.

The owner is the unique identity of the person who created and can manage a given Cash Card.

The following updates were made:

- `owner` was added as a field to the `CashCard` Java record.
- `owner` was added to all `.sql` files in `src/main/resources/` and `src/test/resources/`.
- `owner` was added to all `.json` files in `src/test/resources/example/cashcard`.

All application code and tests were updated to support the new `owner` field, but no functionality changed as a result of those updates.

Take some time to review those changes and get familiar with where `owner` now appears in the project.

## Add the Spring Security Dependency

Spring Security support is added by including the appropriate dependency in the build.

## Add the Dependency

Add this dependency to the `dependencies {}` block in `build.gradle`:

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'

    // Add the following dependency
    implementation 'org.springframework.boot:spring-boot-starter-security'
    ...
}
```

## Run the Tests

Use:

```sh
./gradlew test
```

At this point, Spring Security has been added but no application code has been changed.

## Expected Result

The test run should now show multiple failures, for example:

```text
CashCardApplicationTests > shouldReturnASortedPageOfCashCards() FAILED
...
CashCardApplicationTests > shouldReturnACashCardWhenDataIsSaved() FAILED
...
CashCardApplicationTests > shouldCreateANewCashCard() FAILED
...
CashCardApplicationTests > shouldReturnAPageOfCashCards() FAILED
...
CashCardApplicationTests > shouldReturnAllCashCardsWhenListIsRequested() FAILED
...
CashCardApplicationTests > shouldReturnASortedPageOfCashCardsWithNoParametersAndUseDefaultValues() FAILED
...
CashCardApplicationTests > shouldNotReturnACashCardWithAnUnknownId() FAILED
11 tests completed, 7 failed
> Task :test FAILED
```

Many of the failures look like:

```text
expected: <SOME NUMBER>
 but was: 0
```

In most cases, the tests expect `CashCard` data to be returned, but the API returns nothing useful.

## Understand Why Everything Is Broken

Adding the Spring Security dependency enables security by default.

Since the application has not yet defined how authentication and authorization should work, Spring Security locks down the API completely.

That is why all the API tests in `CashCardApplicationTests` begin failing after the dependency is added.

The next step is to configure Spring Security so the application can authenticate users and authorize access correctly.

## Satisfy Spring Security's Dependencies

The next goal is to get the tests passing again by providing the minimum configuration Spring Security requires.

A file has already been added for this purpose: `example/cashcard/SecurityConfig.java`.

This class will become the Spring bean used to configure application security.

## Uncomment `SecurityConfig.java` and Review It

Open `SecurityConfig.java`.

Most of the file is commented out.

Uncomment all commented lines in the file, including the imports and methods, so it looks conceptually like:

```java
package example.cashcard;
...
class SecurityConfig {

    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http.build();
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
...
```

For now, `filterChain()` simply returns `http.build()`, which is the minimum required configuration.

Ignore `passwordEncoder()` for the moment.

## Enable Spring Security Configuration

At this stage, `SecurityConfig` is still just a plain Java class. Spring is not using it yet.

Add the following annotations:

```java
@Configuration
class SecurityConfig {

    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http.build();
    }
    ...
}
```

## Understand the Annotations

`@Configuration`

- Tells Spring this class contributes application configuration.
- Beans defined here become available to Spring Boot’s configuration system.

`@Bean`

- Tells Spring to register the return value of the method as a managed bean.
- Spring Security expects a `SecurityFilterChain` bean for HTTP security configuration.

By annotating `filterChain(...)` with `@Bean`, the application supplies the minimum security configuration Spring Security needs.

## Run the Tests

Use:

```sh
./gradlew test
```

## Expected Result

At this point, all tests should pass again except the test that creates a new `CashCard` with `POST`:

```text
CashCardApplicationTests > shouldCreateANewCashCard() FAILED
    org.opentest4j.AssertionFailedError:
    expected: 201 CREATED
     but was: 403 FORBIDDEN
...
11 tests completed, 1 failed
```

## Why the POST Test Still Fails

This is expected.

The basic security configuration is now present, but the `POST` request is still being blocked with `403 FORBIDDEN`.

That behavior will be addressed in a later step.

## Configure Basic Authentication

So far, Spring Security has been bootstrapped, but the application has not yet been explicitly secured.

The next step is to enable HTTP Basic Authentication.

## Update `SecurityConfig.filterChain`

Change `filterChain(...)` to:

```java
@Bean
SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
     http
             .authorizeHttpRequests(request -> request
                     .requestMatchers("/cashcards/**")
                     .authenticated())
             .httpBasic(Customizer.withDefaults())
             .csrf(csrf -> csrf.disable());
     return http.build();
}
```

## Understand the Configuration

This configuration means:

- All HTTP requests to `/cashcards/**` must be authenticated.
- Authentication uses HTTP Basic Authentication, which sends a username and password.
- CSRF protection is disabled for now.

The chained method calls are using Spring Security’s builder pattern to define those rules.

Note: CSRF will be discussed later in the lab.

## Run the Tests

Use:

```sh
./gradlew test
```

## Expected Result

Most tests should now fail with `401 UNAUTHORIZED`, for example:

```text
expected: 200 OK
  but was: 401 UNAUTHORIZED
```

## Why This Is Progress

The application is now actually enforcing authentication.

The tests are failing because the HTTP requests in `CashCardApplicationTests` do not yet include a username and password.

The next step is to update the tests so they authenticate successfully.

## Testing Basic Authentication

Spring Security supports many authentication and authorization approaches.

For the tests in this project, a test-only in-memory user service is sufficient.

This works similarly to the in-memory H2 database used for Spring Data tests.

## Configure a Test-Only `UserDetailsService`

The test data in `src/test/resources/data.sql` uses `OWNER` values such as `sarah1`:

```sql
INSERT INTO CASH_CARD(ID, AMOUNT, OWNER) VALUES (100, 1.00, 'sarah1');
```

So the test user should match that owner value.

Add this bean to `SecurityConfig`:

```java
@Bean
UserDetailsService testOnlyUsers(PasswordEncoder passwordEncoder) {
   User.UserBuilder users = User.builder();
   UserDetails sarah = users
     .username("sarah1")
     .password(passwordEncoder.encode("abc123"))
     .roles() // No roles for now
     .build();
   return new InMemoryUserDetailsManager(sarah);
}
```

This configures a test-only user:

- username: `sarah1`
- password: `abc123`

Spring’s IoC container will register this `UserDetailsService` bean, and Spring Security will use it for authentication.

## Configure Basic Auth in the HTTP Tests

Pick one test that uses `restTemplate.getForEntity(...)` and update it to send credentials.

For example:

```java
void shouldReturnACashCardWhenDataIsSaved() {
    ResponseEntity<String> response = restTemplate
            .withBasicAuth("sarah1", "abc123")
            .getForEntity("/cashcards/99", String.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    ...
}
```

## Run the Tests

Use:

```sh
./gradlew test
```

The updated test that includes valid credentials should now pass.

## Update the Remaining Tests

Next, update all remaining `restTemplate`-based tests so every HTTP request includes:

```java
.withBasicAuth("sarah1", "abc123")
```

Then rerun the tests:

```sh
./gradlew test
```

Expected result:

```text
BUILD SUCCESSFUL in 9s
```

At this point, all tests should pass again.

## Verify Basic Auth with Additional Tests

Add a test that expects `401 UNAUTHORIZED` when invalid credentials are used:

```java
@Test
void shouldNotReturnACashCardWhenUsingBadCredentials() {
    ResponseEntity<String> response = restTemplate
      .withBasicAuth("BAD-USER", "abc123")
      .getForEntity("/cashcards/99", String.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.UNAUTHORIZED);

    response = restTemplate
      .withBasicAuth("sarah1", "BAD-PASSWORD")
      .getForEntity("/cashcards/99", String.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.UNAUTHORIZED);
}
```

This test should pass and confirms that invalid usernames or passwords are rejected correctly.

With authentication working, the next step is to implement authorization.

## Support Authorization

Spring Security supports several authorization strategies.

In this lab, authorization is implemented using Role-Based Access Control (RBAC).

The idea is that many users may exist in the user service, but only card owners should be allowed to access Family Cash Cards.

## Add Users and Roles to the `UserDetailsService` Bean

To test authorization, the in-memory user service needs multiple users with different roles.

Update `SecurityConfig.testOnlyUsers(...)` so:

- `sarah1` has the `CARD-OWNER` role
- `hank-owns-no-cards` is added with the `NON-OWNER` role

Use:

```java
@Bean
UserDetailsService testOnlyUsers(PasswordEncoder passwordEncoder) {
  User.UserBuilder users = User.builder();
  UserDetails sarah = users
    .username("sarah1")
    .password(passwordEncoder.encode("abc123"))
    .roles("CARD-OWNER")
    .build();
  UserDetails hankOwnsNoCards = users
    .username("hank-owns-no-cards")
    .password(passwordEncoder.encode("qrs456"))
    .roles("NON-OWNER")
    .build();
  return new InMemoryUserDetailsManager(sarah, hankOwnsNoCards);
}
```

## Add a Role Verification Test

Add this test:

```java
@Test
void shouldRejectUsersWhoAreNotCardOwners() {
    ResponseEntity<String> response = restTemplate
      .withBasicAuth("hank-owns-no-cards", "qrs456")
      .getForEntity("/cashcards/99", String.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.FORBIDDEN);
}
```

This test asserts that a user without the `CARD-OWNER` role should not be allowed to access a cash card.

It is true that cash card `99` belongs to `sarah1`, and later in the lab ownership-specific authorization will matter too.

For now, the focus is just verifying the role requirement.

## Run the Tests

Use:

```sh
./gradlew test
```

## Expected Failure

The new test should fail:

```text
CashCardApplicationTests > shouldRejectUsersWhoAreNotCardOwners() FAILED
 org.opentest4j.AssertionFailedError:
 expected: 403 FORBIDDEN
  but was: 200 OK
```

## Why It Fails

Although the users now have roles, the application is not yet enforcing role-based authorization.

That is why `hank-owns-no-cards` still receives `200 OK`.

## Enable Role-Based Security

Update `SecurityConfig.filterChain(...)` so the `/cashcards/**` endpoints require the `CARD-OWNER` role.

Replace `.authenticated()` with `.hasRole("CARD-OWNER")`:

```java
@Bean
SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
     http
             .authorizeHttpRequests(request -> request
                     .requestMatchers("/cashcards/**")
                     .hasRole("CARD-OWNER"))
             .httpBasic(Customizer.withDefaults())
             .csrf(csrf -> csrf.disable());
     return http.build();
}
```

## Run the Tests Again

Use:

```sh
./gradlew test
```

Now the authorization test should pass:

```text
CashCardApplicationTests > shouldRejectUsersWhoAreNotCardOwners() PASSED
```

At this point, RBAC-based authorization is enabled successfully.

## Cash Card ownership: Repository Updates

There is still a major security problem in the application.

Any authenticated user with the `CARD-OWNER` role can currently view any other user's Family Cash Cards.

To fix that, the repository and controller must both restrict access by owner.

## What Will Change

This step introduces three changes:

- A repository that scopes queries to the correct `OWNER`
- A controller that always uses the correct authenticated owner
- Tests that verify users cannot access each other's data

## Learning Moment: Best Practices

Even with Spring Security in place, it is still the developer’s responsibility to write secure application code.

Spring Security can help with authorization, and features such as method security may also be useful, but that does not replace the need to enforce data ownership in the repository and controller layers.

A secure API should never allow one user to access another user’s data.

## Add a New CashCard for Another User

Update `src/test/resources/data.sql` with a new record owned by `kumar2`:

```sql
INSERT INTO CASH_CARD(ID, AMOUNT, OWNER) VALUES (102, 200.00, 'kumar2');
```

## Add a Test for Ownership Enforcement

Add this test:

```java
@Test
void shouldNotAllowAccessToCashCardsTheyDoNotOwn() {
    ResponseEntity<String> response = restTemplate
      .withBasicAuth("sarah1", "abc123")
      .getForEntity("/cashcards/102", String.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
}
```

This test asserts that `sarah1` should not be allowed to access cash card `102`, because that record belongs to `kumar2`.

Returning `404 NOT_FOUND` is a safer choice than revealing whether the resource exists but is forbidden.

## Run the Tests

Use:

```sh
./gradlew test
```

## Expected Failures

The new ownership test should fail:

```text
CashCardApplicationTests > shouldNotAllowAccessToCashCardsTheyDoNotOwn() FAILED
 org.opentest4j.AssertionFailedError:
 expected: 404 NOT_FOUND
  but was: 200 OK
```

The list test should also fail:

```text
CashCardApplicationTests > shouldReturnAllCashCardsWhenListIsRequested() FAILED
 org.opentest4j.AssertionFailedError:
 expected: 3
  but was: 4
```

## Why These Tests Fail

Right now:

- `sarah1` is authenticated
- `sarah1` has the `CARD-OWNER` role
- the repository is not filtering by owner

So `sarah1` is still able to access `kumar2`'s cash card directly, and that extra card is also included in the list response.

## Update `CashCardRepository`

The simplest fix is to scope all data access by owner.

Edit `CashCardRepository` and add these imports:

```java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
```

Then add these finder methods:

```java
interface CashCardRepository extends CrudRepository<CashCard, Long>, PagingAndSortingRepository<CashCard, Long> {
   CashCard findByIdAndOwner(Long id, String owner);
   Page<CashCard> findByOwner(String owner, PageRequest pageRequest);
}
```

Spring Data will generate the SQL for these methods automatically based on the method names.

That allows the application to query:

- one `CashCard` by both `id` and `owner`
- a page of `CashCard` records by `owner`

The next step is to update the controller to use these new repository methods.

## Cash Card ownership: Controller Updates

The repository now supports filtering data by owner, but the controller is not using that functionality yet.

This step updates the controller so it always queries using the authenticated user.

To do that, use `Principal`, which Spring makes available to controller methods.

The `Principal` holds information about the currently authenticated user.

## Update the GET-by-ID Endpoint

Add this import to `CashCardController`:

```java
import java.security.Principal;
```

Then update the `GET /cashcards/{requestedId}` handler:

```java
@GetMapping("/{requestedId}")
private ResponseEntity<CashCard> findById(@PathVariable Long requestedId, Principal principal) {
    Optional<CashCard> cashCardOptional = Optional.ofNullable(
            cashCardRepository.findByIdAndOwner(requestedId, principal.getName()));
    if (cashCardOptional.isPresent()) {
     ...
```

`principal.getName()` returns the authenticated username from Basic Auth.

That username is now used to scope the query to the current owner.

## Run the Tests

Use:

```sh
./gradlew test
```

## Expected Result

The single-record `GET` tests should now pass again, but list-related tests should still fail.

You should see output like:

```text
CashCardApplicationTests > shouldReturnASortedPageOfCashCards() FAILED
...
CashCardApplicationTests > shouldReturnACashCardWhenDataIsSaved() PASSED
...
CashCardApplicationTests > shouldReturnAllCashCardsWhenListIsRequested() FAILED
...
CashCardApplicationTests > shouldReturnASortedPageOfCashCardsWithNoParametersAndUseDefaultValues() FAILED
...
```

## Update the GET-for-Lists Endpoint

Now update the list endpoint so it also filters by owner:

```java
@GetMapping
private ResponseEntity<List<CashCard>> findAll(Pageable pageable, Principal principal) {
    Page<CashCard> page = cashCardRepository.findByOwner(principal.getName(),
            PageRequest.of(
                pageable.getPageNumber(),
                ...
```

Once again, the authenticated username comes from `principal.getName()`.

That means the controller will return only the `CashCard` records owned by the currently authenticated user.

## Run the Tests Again

Use:

```sh
./gradlew test
```

At this point, the tests should all pass:

```text
BUILD SUCCESSFUL in 8s
```

The application is now enforcing ownership at the controller and repository levels.

## Cash Card ownership: Creation Updates

There is one remaining security gap: creating new `CashCard` records.

The authenticated and authorized `Principal` should always be used as the owner of a newly created cash card.

If the application trusted the submitted `owner` value directly, a user could create a cash card on behalf of someone else.

The fix is to ignore the submitted owner and always use the authenticated principal.

## Update the POST Test

To prove that the client does not need to submit an owner, update the create test to use `null` for the owner:

```java
void shouldCreateANewCashCard() {
  CashCard newCashCard = new CashCard(null, 250.00, null);
  ...
}
```

## Run the Tests

Use:

```sh
./gradlew test --info
```

## Expected Failure

The create test should fail with:

```text
CashCardApplicationTests > shouldCreateANewCashCard() FAILED
 org.opentest4j.AssertionFailedError:
 expected: 201 CREATED
  but was: 403 FORBIDDEN
```

## Why It Fails

The stacktrace should contain a database failure such as:

```text
DbActionExecutionException: Failed to execute InsertRoot{entity=CashCard[id=null, amount=250.0, owner=null], idValueSource=GENERATED}] with root cause

org.h2.jdbc.JdbcSQLIntegrityConstraintViolationException: NULL not allowed for column "OWNER"; SQL statement:
 INSERT INTO "CASH_CARD" ("AMOUNT", "OWNER") VALUES (?, ?) [23502-214]
```

The database rejects the insert because `OWNER` is required.

That requirement is defined in `src/main/resources/schema.sql`:

```sql
CREATE TABLE cash_card
(
  ...
  OWNER    VARCHAR(256) NOT NULL
);
```

So even though the test sees `403 FORBIDDEN`, the deeper issue is that the app tried to save a cash card with a `null` owner and failed.

## Learning Moment: Spring Security and Error Handling

Why is the client seeing `403 FORBIDDEN` instead of `500 INTERNAL_SERVER_ERROR`?

Spring Security intentionally helps reduce information leakage.

If the application exposed detailed internal failures too freely, that information could be useful to an attacker.

Returning a generic `403 FORBIDDEN` in many failure cases reveals less about what went wrong internally.

That is why the application appears to forbid the request, even though the root problem is really a failed database insert.

## Update the POST Endpoint

Fix the controller so it always uses the authenticated principal as the owner when creating a new cash card.

Update the `POST` handler to:

```java
@PostMapping
private ResponseEntity<Void> createCashCard(@RequestBody CashCard newCashCardRequest, UriComponentsBuilder ucb, Principal principal) {
    CashCard cashCardWithOwner = new CashCard(null, newCashCardRequest.amount(), principal.getName());
    CashCard savedCashCard = cashCardRepository.save(cashCardWithOwner);
    ...
}
```

This change ensures:

- The submitted `owner` is ignored.
- The owner is always set to the authenticated username.
- A user can create only their own cash cards.

## Run the Tests Again

Use:

```sh
./gradlew test
```

Everything should now pass again:

```text
CashCardApplicationTests > shouldCreateANewCashCard() PASSED
...
BUILD SUCCESSFUL in 7s
```

At this point, cash card creation is also correctly scoped to the authenticated principal.

## About CSRF

As covered in the lesson, protection against Cross-Site Request Forgery (CSRF, or "sea-surf") is an important aspect of HTTP-based APIs used by web-based applications.

Yet, CSRF has been disabled via the `csrf.disable()` code in `SecurityConfig.filterChain(...)`.

## Why CSRF Was Disabled

For the Family Cash Card API, the lab follows Spring Security’s guidance for non-browser clients:

Use CSRF protection for requests that could be processed by a browser on behalf of a normal user.

If the application is only serving non-browser clients, disabling CSRF is often appropriate.

That is why CSRF protection was turned off for this exercise.

## If You Want to Add CSRF Later

If you want to secure the application with CSRF protection in the future, useful places to review include:

- MockMvc CSRF testing examples
- WebTestClient CSRF testing examples
- The Double-Submit Cookie Pattern

These topics are good next steps if you want to understand how CSRF protection is tested and applied in Spring Security.

## Summary

In this lab, Spring Security was used to ensure that only authenticated and authorized users can access the Family Cash Card API.

In addition, the controller and repository layers were updated to follow secure ownership rules so users can access only their own cash card data.

The lab also showed that Spring Security does more than protect requests. It also influences how errors are exposed, helping avoid leaking information about crashes and other internal application behavior.
