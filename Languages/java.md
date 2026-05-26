# Java Cheatsheet

## Mental Model

Java is a **statically typed, object-oriented** language that runs on the JVM. Everything is a class. The language prioritizes readability, backward compatibility, and enterprise tooling over brevity. Modern Java (17+) has closed the gap with Kotlin/Scala significantly: records, sealed classes, pattern matching, text blocks, and virtual threads make it much more expressive than Java 8. The ecosystem (Maven, Gradle, Spring) is the most mature in the industry.

---

## Install & Minimal Setup

```bash
# SDKMAN — manages multiple Java versions (recommended)
curl -s "https://get.sdkman.io" | bash
sdk install java 21.0.3-tem    # Eclipse Temurin — most common distro
sdk install java 17.0.11-tem
sdk use java 21.0.3-tem

# Verify
java --version
javac --version

# Run a single file (Java 11+)
java HelloWorld.java

# Maven project
mvn archetype:generate -DgroupId=com.example -DartifactId=my-app \
  -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
cd my-app && mvn package && java -jar target/my-app-1.0.jar

# Gradle project
gradle init --type java-application
./gradlew run
./gradlew test
```

---

## Core Concepts

### 1. Variables & Types

```java
// Primitive types
int     age = 28;
long    population = 8_000_000_000L;
double  price = 19.99;
float   tax = 0.15f;
boolean active = true;
char    grade = 'A';
byte    b = 127;
short   s = 32767;

// var — local type inference (Java 10+)
var name = "Joshua";        // inferred as String
var list = new ArrayList<String>();

// Wrapper types (needed for collections, generics)
Integer boxed = 42;
int unboxed = boxed;        // auto-unboxing

// String — immutable
String s = "Hello";
String s2 = s + " World";          // creates new String
String s3 = s.concat(" World");
s.length();
s.contains("ell");
s.substring(1, 3);                  // "el"
s.trim();
s.toLowerCase();
s.split(",");
s.replace("old", "new");
String.format("Hello, %s! Age: %d", name, age);

// Text block (Java 15+)
String json = """
    {
        "name": "Joshua",
        "role": "AI Engineer"
    }
    """;
```

### 2. Control Flow

```java
// if / else
if (score >= 90) {
    grade = "A";
} else if (score >= 80) {
    grade = "B";
} else {
    grade = "C";
}

// Ternary
String status = active ? "active" : "inactive";

// switch expression (Java 14+) — preferred over switch statement
String result = switch (day) {
    case MONDAY, TUESDAY -> "Weekday";
    case SATURDAY, SUNDAY -> "Weekend";
    default -> "Midweek";
};

// switch with yield (multi-line cases)
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY                -> 7;
    default -> {
        System.out.println("checking " + day);
        yield day.toString().length();
    }
};

// for loops
for (int i = 0; i < 10; i++) { }
for (String item : list) { }           // enhanced for
list.forEach(item -> System.out.println(item));   // forEach + lambda

// while
while (condition) { }
do { } while (condition);
```

### 3. Classes & Objects

```java
public class User {
    // Fields — prefer private
    private final int id;
    private String name;
    private String email;

    // Constructor
    public User(int id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }

    // Getters
    public int getId()       { return id; }
    public String getName()  { return name; }
    public String getEmail() { return email; }

    // Setter (only for mutable fields)
    public void setName(String name) { this.name = name; }

    @Override
    public String toString() {
        return "User{id=%d, name='%s'}".formatted(id, name);
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User user)) return false;
        return id == user.id;
    }

    @Override
    public int hashCode() { return Integer.hashCode(id); }
}
```

### 4. Records (Java 16+) — Immutable Data Classes

```java
// Replaces boilerplate getter/equals/hashCode/toString
public record Point(double x, double y) {
    // Compact constructor — validate on creation
    public Point {
        if (x < 0 || y < 0) throw new IllegalArgumentException("coords must be positive");
    }

    // Additional methods
    public double distanceTo(Point other) {
        return Math.sqrt(Math.pow(x - other.x, 2) + Math.pow(y - other.y, 2));
    }
}

var p = new Point(3.0, 4.0);
p.x();          // accessor (not getX())
p.toString();   // auto-generated: Point[x=3.0, y=4.0]
```

### 5. Interfaces & Abstract Classes

```java
// Interface — can have default methods (Java 8+)
public interface Processable {
    void process();    // abstract

    default void log(String msg) {    // default implementation
        System.out.println("[LOG] " + msg);
    }

    static Processable noOp() {       // static factory
        return () -> {};
    }
}

// Sealed classes (Java 17+) — restrict which classes can extend
public sealed interface Shape permits Circle, Rectangle, Triangle { }

public record Circle(double radius) implements Shape { }
public record Rectangle(double width, double height) implements Shape { }

// Pattern matching with switch (Java 21+)
double area = switch (shape) {
    case Circle c        -> Math.PI * c.radius() * c.radius();
    case Rectangle r     -> r.width() * r.height();
    case Triangle t      -> 0.5 * t.base() * t.height();
};
```

### 6. Generics

```java
// Generic class
public class Box<T> {
    private T value;
    public Box(T value) { this.value = value; }
    public T get() { return value; }
}

Box<String> box = new Box<>("hello");

// Generic method
public <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}

// Bounded wildcards
void printList(List<? extends Number> list) { }    // read — covariant
void addToList(List<? super Integer> list) { }     // write — contravariant
```

### 7. Collections

```java
import java.util.*;

// List
List<String> list = new ArrayList<>();
list.add("a");
list.get(0);
list.remove(0);
list.size();
Collections.sort(list);

List<String> immutable = List.of("a", "b", "c");  // Java 9+

// Map
Map<String, Integer> map = new HashMap<>();
map.put("key", 1);
map.get("key");                              // null if not found
map.getOrDefault("key", 0);
map.containsKey("key");
map.remove("key");
map.putIfAbsent("key", 0);
map.computeIfAbsent("key", k -> k.length());

Map<String, Integer> ordered = new LinkedHashMap<>();     // insertion order
Map<String, Integer> sorted  = new TreeMap<>();           // sorted by key

// Set
Set<String> set = new HashSet<>(List.of("a", "b", "c"));
set.add("d");
set.contains("a");

// Map.of / Set.of (immutable, Java 9+)
var map2 = Map.of("a", 1, "b", 2);
var set2 = Set.of("x", "y", "z");
```

### 8. Streams (Java 8+)

```java
import java.util.stream.*;

List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// filter → map → collect
List<Integer> evensDoubled = numbers.stream()
    .filter(n -> n % 2 == 0)
    .map(n -> n * 2)
    .collect(Collectors.toList());       // or .toList() in Java 16+

// reduce
int sum = numbers.stream().reduce(0, Integer::sum);

// groupingBy
Map<Boolean, List<Integer>> grouped = numbers.stream()
    .collect(Collectors.groupingBy(n -> n % 2 == 0));

// joining
String csv = List.of("a", "b", "c").stream()
    .collect(Collectors.joining(", "));

// findFirst, anyMatch, allMatch
Optional<Integer> first = numbers.stream().filter(n -> n > 5).findFirst();
boolean hasLarge = numbers.stream().anyMatch(n -> n > 9);

// sorted, distinct, limit, skip
numbers.stream()
    .sorted(Comparator.reverseOrder())
    .distinct()
    .limit(5)
    .forEach(System.out::println);
```

### 9. Optional

```java
Optional<String> opt = Optional.of("value");
Optional<String> empty = Optional.empty();
Optional<String> nullable = Optional.ofNullable(maybeNull);

opt.isPresent();
opt.get();                              // throws if empty — avoid
opt.orElse("default");
opt.orElseGet(() -> computeDefault()); // lazy
opt.orElseThrow(() -> new RuntimeException("not found"));
opt.map(String::toUpperCase);
opt.filter(s -> s.length() > 3);
opt.ifPresent(System.out::println);
```

### 10. Exception Handling

```java
// Checked vs unchecked
// Checked — must handle or declare (IOException, SQLException)
// Unchecked — RuntimeException subclasses (NullPointerException, etc.)

try {
    String content = Files.readString(Path.of("file.txt"));
} catch (IOException e) {
    System.err.println("Could not read file: " + e.getMessage());
} finally {
    // always runs
}

// Try-with-resources — auto-closes AutoCloseable
try (var reader = new BufferedReader(new FileReader("file.txt"))) {
    String line = reader.readLine();
}

// Custom exception
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(int id) {
        super("User not found: " + id);
    }
}
```

---

## Modern Java Patterns (Java 17–21)

```java
// Pattern matching instanceof (Java 16+)
if (obj instanceof String s && s.length() > 5) {
    System.out.println(s.toUpperCase());
}

// Virtual threads (Java 21) — lightweight, millions can run concurrently
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> handleRequest(request));
}

// Structured concurrency (Java 21 preview)
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var user  = scope.fork(() -> fetchUser(id));
    var order = scope.fork(() -> fetchOrder(id));
    scope.join().throwIfFailed();
    return new Result(user.get(), order.get());
}
```

---

## Gotchas

- **`==` vs `.equals()`** — `==` compares references for objects, not values. Always use `.equals()` for strings and objects. Exception: enums and primitives.
- **`null` everywhere** — Java's biggest design flaw. Use `Optional`, `@NonNull` annotations, and validate eagerly. Never return `null` from a method if you can avoid it.
- **Integer cache** — `Integer.valueOf(127) == Integer.valueOf(127)` is `true`, but `Integer.valueOf(128) == Integer.valueOf(128)` is `false`. Always use `.equals()`.
- **String concatenation in loops** — `+=` in a loop creates a new `String` each time. Use `StringBuilder` for loop concatenation.
- **Checked exceptions** — Java forces you to handle or declare them. Can leak implementation details. Wrap in unchecked exceptions at boundaries.
- **`List.of()` is immutable** — calling `.add()` on it throws `UnsupportedOperationException`. Use `new ArrayList<>(List.of(...))` if you need mutation.

---

## Quick Links

- [Java Documentation (Oracle)](https://docs.oracle.com/en/java/)
- [Baeldung](https://www.baeldung.com) — best practical Java tutorials
- [Java Magazine](https://blogs.oracle.com/javamagazine/)
- [JEP Index](https://openjdk.org/jeps/0) — track new language features
- [SDKMAN](https://sdkman.io) — manage Java versions
- [Effective Java (book)](https://www.oreilly.com/library/view/effective-java/9780134686097/) — essential reading
