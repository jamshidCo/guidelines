# Java Coding Guidelines

## 1. Prefer `Optional` for Potentially Absent Return Values
* Don't return `null` for methods that might not produce a result.
  ```java
  // DON'T
  public User findUser(String id) {
      User user = //... find user
      return (user != null) ? user : null; // Caller must check null
  }
  ```
* Do use `java.util.Optional` to clearly signal potential absence.
  ```java
  // DO
  public Optional<User> findUser(String id) {
      User user = //... find user
      return Optional.ofNullable(user);
  }
  // Caller: userOpt.ifPresent(u -> ...); or userOpt.orElseThrow(...);
  ```
* Why? Improves API clarity and reduces `NullPointerException` risks by forcing conscious handling of absent values. Primarily for return types.
* Want to learn more? [Oracle Java Docs: Optional](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Optional.html), [Baeldung: Guide to Optional](https://www.baeldung.com/java-optional)

## 2. Use Streams API for Collection Processing
* Don't use verbose loops for common collection transformations.
  ```java
  // DON'T
  List<String> results = new ArrayList<>();
  for (Item item : items) {
      if (item.isActive()) {
          results.add(item.getName().toUpperCase());
      }
  }
  ```
* Do use the Streams API for declarative and concise collection processing.
  ```java
  // DO
  List<String> results = items.stream()
                              .filter(Item::isActive)
                              .map(Item::getName)
                              .map(String::toUpperCase)
                              .collect(Collectors.toList());
  ```
* Why? Streams often improve readability by describing *what* to do, not *how*. They can simplify complex processing pipelines.
* Want to learn more? [Oracle Java Tutorial: Aggregate Operations](https://docs.oracle.com/javase/tutorial/collections/streams/index.html), [Baeldung: Introduction to Java 8 Streams](https://www.baeldung.com/java-8-streams-introduction)

## 3. Handle Exceptions Properly
* Don't swallow exceptions or catch overly broad exceptions like `Exception`.
  ```java
  // DON'T
  try {
    riskyOperation();
  } catch (Exception e) {
    // Ignored or generic log - hides the real problem
    log.error("Something failed");
  }
  ```
* Do catch specific exceptions you can handle meaningfully, or let them propagate. Log exceptions with context.
  ```java
  // DO
  try {
      readFile("config.txt");
  } catch (FileNotFoundException e) {
      log.warn("Config not found, using defaults.", e);
      useDefaults();
  } catch (IOException e) {
      log.error("Failed reading config.", e);
      throw new UnrecoverableConfigException("Cannot load config", e);
  }
  ```
* Why? Proper exception handling ensures robustness, aids debugging, and prevents hiding critical errors.
* Want to learn more? [Oracle Java Tutorial: Exceptions](https://docs.oracle.com/javase/tutorial/essential/exceptions/index.html), [Effective Java Item 70-77 (Exceptions)](https://www.oreilly.com/library/view/effective-java-3rd/9780134686097/)

## 4. Use `.equals()` for Object Comparison, `==` for Primitives/Identity
* Don't use `==` to compare object content (like Strings).
  ```java
  // DON'T
  String s1 = new String("test");
  String s2 = new String("test");
  if (s1 == s2) { /* false: compares references */ }
  ```
* Do use `.equals()` for object content comparison. Use `==` for primitives and checking if references point to the *same* object. Use `Objects.equals()` for null-safety.
  ```java
  // DO
  String s1 = "test";
  String s2 = "test";
  if (s1.equals(s2)) { /* true: compares content */ }
  if (Objects.equals(s1, s2)) { /* true: null-safe compare */ }

  int a = 5; int b = 5;
  if (a == b) { /* true: primitive compare */ }
  ```
* Why? `==` on objects checks reference equality, `.equals()` checks logical equality. Confusing them leads to bugs.
* Want to learn more? [Baeldung: Difference Between == and .equals() in Java](https://www.baeldung.com/java-equals-hashcode-contracts), [Effective Java Item 10 (equals)](https://www.oreilly.com/library/view/effective-java-3rd/9780134686097/)

## 5. Use Standard Naming Conventions
* Don't use inconsistent or non-standard naming.
  ```java
  // DON'T
  class user_data { // non-standard class name
      String UserNAME; // non-standard variable name
      final int MAX_VAL = 10; // non-standard constant name
      void Process_It() {} // non-standard method name
  }
  ```
* Do follow standard Java conventions: `PascalCase` for classes/interfaces, `camelCase` for methods/variables, `SCREAMING_SNAKE_CASE` for constants (`static final`).
  ```java
  // DO
  class UserData {
      String userName;
      static final int MAX_VALUE = 10;
      void processData() {}
  }
  ```
* Why? Consistency improves code readability and understandability across teams.
* Want to learn more? [Oracle Code Conventions for the Java Programming Language (Section 9 - Naming Conventions)](https://www.oracle.com/java/technologies/javase/codeconventions-namingconventions.html), [Google Java Style Guide: Naming](https://google.github.io/styleguide/javaguide.html#s5-naming)

## 6. Always Use Braces for Control Statements
* Don't omit braces, even for single-line blocks.
  ```java
  // DON'T
  if (isValid)
      process(); // Error-prone if lines are added without braces
  ```
* Do always use braces `{}` for `if`, `else`, `for`, `while`.
  ```java
  // DO
  if (isValid) {
      process();
  } else {
      logError();
  }
  ```
* Why? Avoids potential errors when modifying code later and improves clarity.
* Want to learn more? [Google Java Style Guide: Braces](https://google.github.io/styleguide/javaguide.html#s4.1-braces)

## 7. Avoid Magic Numbers and Strings
* Don't use unexplained literal values directly in code.
  ```java
  // DON'T
  if (user.getStatus() == 3) { /* What is 3? */ }
  draw(x, y, 25); /* What is 25? */
  if (type.equals("URGENT_ORDER")) { /* Typo-prone */ }
  ```
* Do define named constants (`static final`) or use Enums.
  ```java
  // DO
  private static final int STATUS_PENDING = 3;
  private static final int DEFAULT_RADIUS = 25;
  public static final String TYPE_URGENT = "URGENT_ORDER"; // Or use Enum

  if (user.getStatus() == STATUS_PENDING) {}
  draw(x, y, DEFAULT_RADIUS);
  if (TYPE_URGENT.equals(type)) {} // Or: if (orderType == OrderType.URGENT)
  ```
* Why? Improves readability and maintainability. Makes changes easier and less error-prone.
* Want to learn more? [Clean Code (Book by Robert C. Martin)](https://www.oreilly.com/library/view/clean-code-a/9780136083238/), [Refactoring Guru: Replace Magic Number with Symbolic Constant](https://refactoring.guru/replace-magic-number-with-symbolic-constant)

## 8. Write Small, Focused Methods
* Don't write long methods performing multiple distinct tasks.
  ```java
  // DON'T (Conceptual)
  void processUserData(User user) {
      // Validate user (20 lines)
      // Fetch permissions (15 lines)
      // Calculate metrics (30 lines)
      // Save results (10 lines)
  }
  ```
* Do decompose logic into small methods, each doing one thing well.
  ```java
  // DO (Conceptual)
  void processUserData(User user) {
      validateUser(user);
      List<Permission> perms = fetchPermissions(user);
      Metrics metrics = calculateMetrics(user, perms);
      saveResults(user, metrics);
  }
  private void validateUser(User u) {/*...*/}
  private List<Permission> fetchPermissions(User u) {/*...*/}
  // ... more private helper methods ...
  ```
* Why? Enhances readability, testability, and reusability. Easier to understand and maintain.
* Want to learn more? [Clean Code (Book by Robert C. Martin) - Chapter 3: Functions](https://www.oreilly.com/library/view/clean-code-a/9780136083238/), [Refactoring Guru: Extract Method](https://refactoring.guru/extract-method)

## 9. Comment Why, Not What
* Don't write comments explaining *what* obvious code does.
  ```java
  // DON'T
  // increment count
  count++;
  ```
* Do use comments to explain *why* something is done, the intent, or non-obvious logic/workarounds. Use Javadoc (`/** ... */`) for public APIs.
  ```java
  // DO
  // Workaround for API bug #123 where count is off-by-one in rare cases.
  if (needsAdjustment(result)) {
      count--;
  }

  /** Javadoc for public API */
  public int calculateScore(...) {...}
  ```
* Why? Code should explain itself. Comments add value by providing context the code cannot.
* Want to learn more? [Clean Code (Book by Robert C. Martin) - Chapter 4: Comments](https://www.oreilly.com/library/view/clean-code-a/9780136083238/), [Oracle: How to Write Doc Comments for the Javadoc Tool](https://www.oracle.com/technical-resources/articles/java/javadoc-tool.html)

## 10. Minimize Mutability
* Don't leave fields mutable without a good reason. Avoid returning mutable internal collections directly.
  ```java
  // DON'T
  public class Config {
      public List<String> servers; // Mutable list, public field
      public List<String> getServers() { return this.servers; } // Returns internal list
  }
  ```
* Do make fields `final`. Use immutable collections or return unmodifiable views/copies of internal collections.
  ```java
  // DO
  public final class Config { // Class possibly final
      private final List<String> servers; // Final field

      public Config(List<String> srv) {
          this.servers = List.copyOf(srv); // Immutable copy (Java 10+)
          // Or: Collections.unmodifiableList(new ArrayList<>(srv)) for older Java
      }
      public List<String> getServers() { return this.servers; /* Already immutable */ }
  }
  ```
* Why? Immutable objects are simpler, predictable, and inherently thread-safe. Reduces side effects.
* Want to learn more? [Effective Java Item 17 (Minimize mutability)](https://www.oreilly.com/library/view/effective-java-3rd/9780134686097/), [Baeldung: Immutability in Java](https://www.baeldung.com/java-immutable-object)

## 11. Use Dependency Injection Principle
* Don't let components create their own dependencies directly with `new`.
  ```java
  // DON'T - Tight coupling
  public class ReportService {
      private DatabaseConnection dbConn;
      public ReportService() {
          // Creates its own dependency - hard to test/swap
          this.dbConn = new DatabaseConnection("url...");
      }
      public void generateReport() { /* uses dbConn */ }
  }
  ```
* Do provide dependencies from the outside, typically via constructors (Constructor Injection).
  ```java
  // DO - Loose coupling
  public class ReportService {
      private final DbConnection dbConn; // Use interface preferably

      // Dependency provided via constructor
      public ReportService(DbConnection connection) {
          this.dbConn = Objects.requireNonNull(connection);
      }
      public void generateReport() { /* uses dbConn */ }
  }
  // Somewhere else:
  // DbConnection connection = new DatabaseConnection("url...");
  // ReportService service = new ReportService(connection);
  ```
* Why? Promotes loose coupling, making code modular, testable (with mocks), and flexible.
* Want to learn more? [Martin Fowler: Inversion of Control Containers and the Dependency Injection pattern](https://martinfowler.com/articles/injection.html), [Spring Framework Documentation: Core Technologies (IoC)](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#spring-core)

## 12. Program to Interfaces, Not Implementations
* Don't declare variables or return types using concrete implementation types (like `ArrayList`) if an interface (`List`) suffices.
  ```java
  // DON'T
  public ArrayList<String> getUsers() { // Returns concrete type
      ArrayList<String> users = new ArrayList<>();
      // ...
      return users;
  }
  ```
* Do use interface types in declarations. Instantiate with concrete classes.
  ```java
  // DO
  public List<String> getUsers() { // Returns interface type
      List<String> users = new ArrayList<>(); // Instantiates concrete class
      // ...
      return users;
  }
  // Caller: List<String> userList = service.getUsers();
  ```
* Why? Reduces coupling, increases flexibility to change implementations later without affecting callers.
* Want to learn more? [Effective Java Item 64 (Refer to objects by their interfaces)](https://www.oreilly.com/library/view/effective-java-3rd/9780134686097/), [Baeldung: Programming to an Interface](https://www.baeldung.com/java-program-to-interface)

## 13. Use Standard Library Utilities
* Don't reinvent common utility functions (string checks, array copies, etc.).
  ```java
  // DON'T
  boolean isEmpty(String s) { return s == null || s.length() == 0; }
  int[] copyArray(int[] src) { /* manual loop copy */ }
  ```
* Do use utilities from the Java Standard Library (`java.lang`, `java.util`, `java.io`, etc.) or established libraries (Apache Commons, Guava).
  ```java
  // DO
  boolean isEmpty = (s == null || s.isEmpty()); // Standard Java
  // boolean isEmpty = StringUtils.isEmpty(s); // Apache Commons example
  int[] copy = Arrays.copyOf(original, original.length); // Standard Java
  ```
* Why? Leverages well-tested, optimized, and familiar code. Saves time, reduces bugs.
* Want to learn more? [Java Platform Standard Edition API Specification](https://docs.oracle.com/en/java/javase/17/docs/api/index.html), [Apache Commons Lang](https://commons.apache.org/proper/commons-lang/), [Google Guava](https://github.com/google/guava)

## 14. Keep Class Members Private Unless Necessary
* Don't make fields or helper methods `public` or `protected` by default.
  ```java
  // DON'T
  public class Cache {
      public Map<String, Object> internalMap; // Public mutable state
      public void cleanup() { /* internal detail exposed */ }
  }
  ```
* Do default to `private`. Increase visibility only when needed for the public API or subclass extension.
  ```java
  // DO
  public class Cache {
      private final Map<String, Object> internalMap = new ConcurrentHashMap<>(); // Private

      public void put(String key, Object value) { /* Public API method */ }
      public Optional<Object> get(String key) { /* Public API method */ }

      private void cleanupExpired() { /* Private internal helper */ }
  }
  ```
* Why? Encapsulation protects internal state, reduces coupling, and improves maintainability.
* Want to learn more? [Effective Java Item 15 (Minimize the accessibility of classes and members)](https://www.oreilly.com/library/view/effective-java-3rd/9780134686097/), [Oracle Java Tutorial: Controlling Access to Members of a Class](https://docs.oracle.com/javase/tutorial/java/javaOO/accesscontrol.html)

## 15. Use `final` Where Appropriate
* Don't leave variables non-`final` if their reference shouldn't change after assignment.
  ```java
  // DON'T
  void process(String id) {
      String key = generateKey(id); // Could be final
      // ... potentially reassign key accidentally later ...
  }
  ```
* Do use `final` for parameters, local variables, and fields whose reference/value assignment should not change.
  ```java
  // DO
  void process(final String id) { // Final parameter
      final String key = generateKey(id); // Final local variable
      final User user = findUser(key); // Final local variable
      // ... code using key and user ...
  }
  private final String serviceUrl; // Final field (set in constructor)
  ```
* Why? Improves clarity about intent, prevents accidental reassignment, helps reasoning.
* Want to learn more? [Effective Java Item 17 (Minimize mutability - uses final heavily)](https://www.oreilly.com/library/view/effective-java-3rd/9780134686097/), [Baeldung: The final Keyword in Java](https://www.baeldung.com/java-final)

## 16. Remove Dead or Commented-Out Code
* Don't leave unused fields, methods, variables, or commented-out code blocks.
  ```java
  // DON'T
  public class Processor {
      // private String oldValue; // Unused field
      public void run(Data data) {
          int temp = data.getValue(); // Unused variable?
          /* Old logic:
          if (data.isLegacy()) { ... }
          */
          processNewLogic(data);
      }
      // private void helper() {} // Unused method
  }
  ```
* Do remove unused code. Rely on version control (Git) for history.
  ```java
  // DO
  public class Processor {
      public void run(Data data) {
          // Only necessary variables used
          processNewLogic(data);
      }
      private void processNewLogic(Data data) { /* ... */ }
      // Unused members and commented blocks removed
  }
  ```
* Why? Keeps the codebase clean, relevant, and easier to understand and maintain. Avoids confusion.
* Want to learn more? [Clean Code (Book by Robert C. Martin) - Chapter 4 Comments (Avoid Commented-Out Code)](https://www.oreilly.com/library/view/clean-code-a/9780136083238/), [SonarSource Rule: squid:S1148 (Commented-out code lines should be removed)](https://rules.sonarsource.com/java/RSPEC-1148)
