# Writing a Failing Test (Reference)

This is a one-screen reference to the steps you pasted. It is not implemented.

**Create the test class**
1. Create `src/test/java/example/cashcard/CashCardJsonTest.java`.
2. Put this content in the file:

```java
package example.cashcard;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class CashCardJsonTest {

   @Test
   void myFirstTest() {
      assertThat(1).isEqualTo(42);
   }
}
```

**Run the tests (expect failure)**
```sh
./gradlew test
```

Expected failure excerpt:
```
CashCardJsonTest > myFirstTest() FAILED
    org.opentest4j.AssertionFailedError:
    expected: 42
     but was: 1
...
```

**Fix the failing test**
Change the assertion to a known-true statement:
```java
assertThat(42).isEqualTo(42);
```

**Run the tests again (expect success)**
```sh
./gradlew test
```

Expected success excerpt:
```
CashCardJsonTest > myFirstTest() PASSED
CashCardApplicationTests > contextLoads() PASSED
BUILD SUCCESSFUL in 4s
```
