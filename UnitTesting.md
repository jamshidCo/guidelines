# SAP CAP-Java Unit Testing Guidelines

## 1. Purpose

These guidelines aim to ensure consistent, effective, and maintainable unit tests for SAP Cloud Application Programming Model (CAP) Java projects. The goal is to verify the smallest testable parts of the application (units) in isolation, primarily focusing on custom business logic within event handlers and helper classes.

## 2. What is a Unit Test in CAP Java?

*   **Focus:** Test a single class or method (e.g., an event handler method, a utility function).
*   **Isolation:** The unit under test should be isolated from its dependencies like the database, external services, or even other CAP event handlers. These dependencies **must** be mocked.
*   **Speed:** Unit tests should run quickly, providing fast feedback to developers.
*   **Scope:** Do **not** test framework internals (assume CAP/CDS works), database connectivity, or full request/response cycles. These belong to integration or system tests.

## 3. Core Principles

*   **Test Your Logic:** Focus tests on your custom code (conditions, calculations, transformations) within event handlers (`@On`, `@Before`, `@After`) and helper/utility classes.
*   **Mock Dependencies:** Use mocking frameworks (like Mockito) extensively to simulate interactions with:
    *   `PersistenceService` / `CqnService` (Database interactions)
    *   `RemoteService` (External service calls)
    *   `EventContext` object and its methods (`getCqn()`, `getHttpServletRequest()`, parameters, etc.)
    *   Other custom services or classes injected into your unit.
*   **Arrange-Act-Assert (AAA):** Structure your tests clearly:
    *   **Arrange:** Set up preconditions. Instantiate the class under test, create mock objects, define mock behavior (`when(...).thenReturn(...)`), and prepare test data.
    *   **Act:** Execute the method you want to test.
    *   **Assert:** Verify the outcome using assertion libraries (like AssertJ or JUnit Assertions). Check return values, state changes, or interactions with mocks (`verify(...)`).

## 4. Recommended Tools

*   **JUnit 5:** The standard testing framework for Java.
*   **Mockito:** The standard mocking framework for Java. Used to create test doubles (mocks) of dependencies.
*   **AssertJ:** A fluent assertion library for more readable assertions (optional but recommended over standard JUnit assertions).

## 5. What to Test

*   **Event Handlers (`@On`, `@Before`, `@After` methods):**
    *   Verify correct logic based on input data (from `EventContext`, parameters).
    *   Verify correct data manipulation.
    *   Verify correct CQN statements are *prepared* (verify arguments passed to mocked `PersistenceService.run(...)`).
    *   Verify correct calls to other services (using `verify(...)` on mocks).
    *   Verify correct modification of the `EventContext` or result if applicable.
*   **Custom Service/Utility Classes:** Test public methods like any standard Java class, mocking their dependencies.
*   **Data Validation Logic:** Ensure validation rules are correctly applied.
*   **Complex Calculations or Transformations:** Isolate and test these thoroughly.

## 6. How to Test Event Handlers

1.  **Instantiate:** Create an instance of your handler class directly (`MyEventHandler handler = new MyEventHandler();`).
2.  **Mock Dependencies:** Use `@Mock` annotation for dependencies (`PersistenceService`, other services) and `@InjectMocks` on your handler instance (or manually inject mocks via constructor/setters). Remember to initialize mocks (e.g., using `MockitoAnnotations.openMocks(this);` in a `@BeforeEach` method).
3.  **Mock `EventContext`:** Create a mock `EventContext`: `EventContext context = mock(EventContext.class);`
4.  **Stub Context Methods:** Define behavior for methods called on the context:
    *   `when(context.get(DataModel.ENTITY)).thenReturn(someTestData);`
    *   `when(context.getCqn()).thenReturn(mockCqnStatement);`
    *   `when(context.getParameterInfo().getQueryParams()).thenReturn(mockQueryParams);`
5.  **Stub Service Calls:** Define behavior for mocked services:
    *   `when(persistenceService.run(any(CqnSelect.class))).thenReturn(mockResult);`
    *   `doNothing().when(remoteService).emit(any());`
6.  **Act:** Call the handler method directly, passing the mocked context: `handler.beforeCreateBooks(context, bookDataList);`
7.  **Assert:**
    *   Use `verify()` to check if methods on mocks were called with expected arguments:
        *   `verify(persistenceService).run(cqnUpdateArgumentCaptor.capture());`
        *   `// Assert details about the captured CQN statement`
    *   Use assertion libraries (`assertThat(...)`) to check results or state changes on test data potentially modified by the handler.

## 7. Naming Conventions

*   Use descriptive method names for tests, clearly stating the action and expected outcome, e.g., `shouldRejectOrder_whenStockIsInsufficient()` or `givenAdminUser_whenCreatingProduct_thenStatusIsApproved()`.
*   Test class names should correspond to the class under test, suffixed with `Test`, e.g., `OrderServiceHandlerTest`.

## 8. Using `@DisplayName` for Readability

*   **Purpose:** Use JUnit 5's `@DisplayName` annotation to provide custom, human-readable names for test classes and methods. This improves clarity in IDEs, build logs, and test reports.
*   **Benefit:** Allows descriptive sentences with spaces and punctuation, which can be clearer than conventional method names, especially for BDD-style (Given/When/Then) descriptions or when explaining intent to non-developers.
*   **Usage:** Apply the annotation directly above a test method or class.
*   **Recommendation:** Consider using `@DisplayName` when the standard method name isn't sufficiently descriptive or when you want more narrative test output, particularly with BDD. It can complement a descriptive method name (hybrid approach). Be mindful to keep the display name synchronized with the test's actual behavior during refactoring.

*   **Example (BDD Style):**

    ```java
    @Test
    // BDD style: Given [context], When [action], Then [outcome]
    @DisplayName("Given a product with negative stock, When validating before create, Then a ServiceException should be thrown")
    void validateProductStock_shouldThrowException_whenStockIsNegative() {
        // Arrange: A product with negative stock
        Products productWithNegativeStock = Products.create();
        productWithNegativeStock.setStock(-10);
        List<Products> products = List.of(productWithNegativeStock);
        ProductHandler handler = new ProductHandler(); // Assuming no dependencies for this specific test logic

        // Act: Validating before create
        ServiceException exception = assertThrows(ServiceException.class, () -> {
            handler.validateProductStock(products);
        }, "A ServiceException should be thrown for negative stock");

        // Assert: Then a ServiceException should be thrown
        assertThat(exception.getStatusCode()).isEqualTo(ErrorStatusCodes.BAD_REQUEST);
        assertThat(exception.getMessage()).contains("Stock cannot be negative");
    }
    ```

## 9. Best Practices

*   **Keep Tests Small & Focused:** Each test should verify one specific behavior or scenario.
*   **Write Readable Tests:** Use clear variable names, the AAA structure, and consider `@DisplayName` (especially BDD style) for enhanced clarity (see section 8). Add comments *only* when the code isn't self-explanatory.
*   **Independent Tests:** Tests should not depend on each other or the order of execution. Use `@BeforeEach` / `@AfterEach` for setup/teardown.
*   **Test Edge Cases & Errors:** Don't just test the "happy path". Test null inputs, empty lists, error conditions, boundary values.
*   **Refactor Tests:** Keep tests clean and maintainable, just like production code.
*   **Run Tests Frequently:** Integrate tests into your build process and run them often.

## 10. Example (Conceptual Handler & Test with BDD @DisplayName)

**Handler Snippet:**

```java
@Component
@ServiceName("CatalogService")
public class ProductHandler implements EventHandler {

    // Assume PersistenceService is injected (e.g., via constructor)
    private final PersistenceService db;

    public ProductHandler(PersistenceService persistenceService) {
         this.db = persistenceService;
    }

    @Before(event = CdsService.EVENT_CREATE, entity = Products_.CDS_NAME)
    public void validateProductStock(List<Products> products) {
        for (Products product : products) {
            if (product.getStock() != null && product.getStock() < 0) { // Check for null before comparing
                throw new ServiceException(ErrorStatusCodes.BAD_REQUEST, "Stock cannot be negative.");
            }
            // Set a default status if not provided
            if (product.getStatus() == null) {
                product.setStatus("NEW");
            }
        }
    }
    // ... other handler methods
}
```

**Unit Test Snippet (with BDD Style @DisplayName):**

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.DisplayName; // Import DisplayName
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.assertj.core.api.Assertions.assertThat; // Using AssertJ
// ... other imports including your entities, ServiceException etc.
import com.sap.cds.services.ServiceException;
import com.sap.cds.services.ErrorStatusCodes;
import com.sap.cds.services.persistence.PersistenceService; // Import PersistenceService
import com.sap.cds.services.handler.EventHandler; // Import EventHandler
import com.sap.cds.services.handler.annotations.Before; // Import Before
import com.sap.cds.services.handler.annotations.ServiceName; // Import ServiceName
import com.sap.cds.ql.cqn.CqnSelect; // Import CqnSelect if needed for other tests
import com.sap.cds.services.cds.CdsService; // Import CdsService for event constants
import cds.gen.catalogservice.CatalogService_; // Import Service marker
import cds.gen.catalogservice.Products; // Assuming generated entity
import cds.gen.catalogservice.Products_; // Import entity metadata
import java.util.List;
import org.springframework.stereotype.Component; // Import Component


// Test class name matches the class under test
class ProductHandlerTest {

    @InjectMocks // Creates an instance of ProductHandler and injects mocks
    private ProductHandler productHandler;

    // Mock dependencies (even if not used in *all* tests, declare if class needs it)
    @Mock
    private PersistenceService db;

    @BeforeEach
    void setUp() {
        // Initialize mocks defined with @Mock and inject them into @InjectMocks instance
        MockitoAnnotations.openMocks(this);
        // Re-inject mocks if constructor injection is used in the real handler
        // productHandler = new ProductHandler(db); // If using constructor injection
    }

    @Test
    @DisplayName("Given a product with negative stock, When validating before create, Then a ServiceException should be thrown")
    void validateProductStock_shouldThrowException_whenStockIsNegative() {
        // Arrange: A product with negative stock
        Products productWithNegativeStock = Products.create();
        productWithNegativeStock.setStock(-10);
        List<Products> products = List.of(productWithNegativeStock);

        // Act: Validating before create
        ServiceException exception = assertThrows(ServiceException.class, () -> {
            productHandler.validateProductStock(products);
        }, "A ServiceException should be thrown for negative stock");

        // Assert: Then a ServiceException is thrown with correct details
        assertThat(exception.getMessage()).isEqualTo("Stock cannot be negative.");
        assertThat(exception.getStatusCode()).isEqualTo(ErrorStatusCodes.BAD_REQUEST);
    }

    @Test
    @DisplayName("Given a product with null status, When validating before create, Then the status should be set to 'NEW'")
    void validateProductStock_shouldSetDefaultStatus_whenStatusIsNull() {
        // Arrange: A product with null status
        Products productWithoutStatus = Products.create();
        productWithoutStatus.setStock(100); // Valid stock
        productWithoutStatus.setStatus(null); // Status is null
        List<Products> products = List.of(productWithoutStatus);

        // Act: Validating before create
        productHandler.validateProductStock(products); // No exception expected

        // Assert: Then the status is set to 'NEW'
        assertThat(productWithoutStatus.getStatus()).isEqualTo("NEW");
    }

     @Test
     @DisplayName("Given a product with an existing status, When validating before create, Then the status should remain unchanged")
    void validateProductStock_shouldNotChangeStatus_whenStatusIsProvided() {
        // Arrange: A product with an existing status
        Products productWithStatus = Products.create();
        productWithStatus.setStock(50);
        productWithStatus.setStatus("ACTIVE"); // Status provided
        List<Products> products = List.of(productWithStatus);

        // Act: Validating before create
        productHandler.validateProductStock(products);

        // Assert: Then the status remains unchanged
        assertThat(productWithStatus.getStatus()).isEqualTo("ACTIVE");
    }
}
```

## 11. References

- SAP Cloud Application Programming Model (CAP):
  - [Testing CAP Applications (Java)](https://cap.cloud.sap/docs/java/developing-applications/testing#sample-tests)
  - [CAP Java SDK Core Documentation](https://cap.cloud.sap/docs/java/)
- JUnit 5:
  - [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)
  - [@DisplayName Annotation](https://junit.org/junit5/docs/current/user-guide/#writing-tests-display-names)
- Mockito:
  - [Mockito Core Documentation](https://site.mockito.org/)
  - [Mockito Javadoc](https://javadoc.io/doc/org.mockito/mockito-core/latest/org.mockito/module-summary.html)
- AssertJ:
  - [AssertJ Core Features](https://assertj.github.io/doc/#assertj-core-features-overview)
  - [AssertJ Fluent Assertions for Java](https://joel-costigliola.github.io/assertj/)
