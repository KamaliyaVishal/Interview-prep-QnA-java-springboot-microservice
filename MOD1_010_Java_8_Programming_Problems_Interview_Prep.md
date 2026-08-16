# 7. Java 8 Programming Problems — Interview Prep 

*Companion to `006_Java_8_Features_Interview_Prep.md`. Hands-on Stream/Optional/Comparator programs interviewers ask you to **write or explain on a whiteboard**.*

---

## SHARED MODEL (Employee problems)

```java
record Employee(String name, String department, double salary, int age) {}

List<Employee> employees = List.of(
    new Employee("Alice", "Engineering", 120_000, 30),
    new Employee("Bob",   "Engineering",  95_000, 28),
    new Employee("Carol", "Sales",       110_000, 35),
    new Employee("Dave",  "Sales",        85_000, 40),
    new Employee("Eve",   "Engineering", 120_000, 32)
);
```

---

## SECTION 1: LIST / COLLECTION PROGRAMS (Q1–Q20)

### Q1. Find duplicate elements in a List
**Approach 1 — Stream + groupingBy (shows frequency > 1):**
```java
List<Integer> list = List.of(1, 2, 3, 2, 4, 5, 3, 6);

List<Integer> duplicates = list.stream()
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()))
    .entrySet().stream()
    .filter(e -> e.getValue() > 1)
    .map(Map.Entry::getKey)
    .toList();
// [2, 3]
```

**Approach 2 — Set (detects presence of prior element):**
```java
Set<Integer> seen = new HashSet<>();
List<Integer> dups = list.stream()
    .filter(n -> !seen.add(n))   // add returns false if already present
    .distinct()                  // first occurrence of each duplicate only
    .toList();
```

**Interview follow-up:** "Which is better?" → Set approach is O(n) and simpler for "is there any duplicate?". `groupingBy` is better when you also need **count** of each duplicate.

---

### Q2. Find unique elements from a List
**Answer:** Elements that appear exactly once.
```java
List<String> words = List.of("a", "b", "a", "c", "b", "d");

List<String> unique = words.stream()
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()))
    .entrySet().stream()
    .filter(e -> e.getValue() == 1)
    .map(Map.Entry::getKey)
    .toList();
// [c, d]
```

**Alternative — distinct()** returns all unique values (not "appears only once"):
```java
List<String> distinct = words.stream().distinct().toList();   // [a, b, c, d]
```
Know the difference — interviewers often trap you here.

---

### Q3. Find frequency/count of each element
**Answer:**
```java
List<String> fruits = List.of("apple", "banana", "apple", "cherry", "banana", "apple");

Map<String, Long> frequency = fruits.stream()
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
// {apple=3, banana=2, cherry=1}
```

**Character frequency in a String:**
```java
String input = "programming";

Map<Character, Long> charFreq = input.chars()
    .mapToObj(c -> (char) c)
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
// {p=1, r=2, o=1, g=2, a=1, m=2, i=1, n=1}
```

**Senior tip:** For Strings, `input.chars()` gives an `IntStream` of Unicode code units — fine for ASCII interview problems; for full Unicode graphemes use a proper library in production.

---

### Q4. Find the first non-repeated character in a String
**Answer:** Count first, then scan in original order.
```java
public static Optional<Character> firstNonRepeated(String s) {
    Map<Character, Long> freq = s.chars()
        .mapToObj(c -> (char) c)
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));

    return s.chars()
        .mapToObj(c -> (char) c)
        .filter(c -> freq.get(c) == 1)
        .findFirst();
}
// "swiss" → Optional['w']
```

**Follow-up:** "Can you do it in one pass?" → Yes with `LinkedHashMap` to preserve insertion order while counting, then iterate entries — or use a single pass with a pair `(count, firstIndex)` per character.

---

### Q5. Find the first repeated character in a String
**Answer:**
```java
public static Optional<Character> firstRepeated(String s) {
    Set<Character> seen = new HashSet<>();
    return s.chars()
        .mapToObj(c -> (char) c)
        .filter(c -> !seen.add(c))
        .findFirst();
}
// "programming" → Optional['r']
```

**Alternative — index-based (finds earliest repeat by position):**
```java
Map<Character, Integer> firstIndex = new HashMap<>();
for (int i = 0; i < s.length(); i++) {
    char c = s.charAt(i);
    if (firstIndex.containsKey(c)) return Optional.of(c);
    firstIndex.put(c, i);
}
return Optional.empty();
```

---

### Q6. Find the second-highest number in a List
**Answer:**
```java
List<Integer> numbers = List.of(10, 5, 20, 8, 20, 15);

Optional<Integer> secondHighest = numbers.stream()
    .distinct()
    .sorted(Comparator.reverseOrder())
    .skip(1)
    .findFirst();
// Optional[15]
```

**O(n) alternative (better for large lists — senior follow-up):**
```java
public static OptionalInt secondHighest(List<Integer> list) {
    int first = Integer.MIN_VALUE, second = Integer.MIN_VALUE;
    for (int n : list) {
        if (n > first) {
            second = first;
            first = n;
        } else if (n > second && n < first) {
            second = n;
        }
    }
    return second == Integer.MIN_VALUE ? OptionalInt.empty() : OptionalInt.of(second);
}
```

**Trap:** If list has fewer than 2 distinct values, return `Optional.empty()` — always mention edge cases.

---

### Q7. Find the highest and lowest number in a List
**Answer:**
```java
List<Integer> list = List.of(3, 7, 1, 9, 4);

OptionalInt max = list.stream().mapToInt(Integer::intValue).max();
OptionalInt min = list.stream().mapToInt(Integer::intValue).min();
// max=9, min=1

// Or
int highest = Collections.max(list);
int lowest  = Collections.min(list);
```

**With reduce:**
```java
int maxReduce = list.stream().reduce(Integer.MIN_VALUE, Integer::max);
int minReduce = list.stream().reduce(Integer.MAX_VALUE, Integer::min);
```

---

### Q8. Find the sum and average of numbers
**Answer:**
```java
List<Integer> numbers = List.of(10, 20, 30, 40, 50);

int sum = numbers.stream().mapToInt(Integer::intValue).sum();           // 150
OptionalDouble avg = numbers.stream().mapToInt(Integer::intValue).average(); // 30.0

// reduce alternative
int sumReduce = numbers.stream().reduce(0, Integer::sum);
```

**Senior note:** Prefer `mapToInt` over `map` + `reduce` — avoids boxing overhead and reads clearer. `IntStream` has dedicated `sum()`, `average()`, `max()`, `min()`.

---

### Q9. Sort a List of Strings — ascending and descending
**Answer:**
```java
List<String> names = new ArrayList<>(List.of("Charlie", "Alice", "Bob"));

// Ascending — natural order
names.sort(Comparator.naturalOrder());
// or: names.stream().sorted().toList()  (returns new list, original unchanged)

// Descending
names.sort(Comparator.reverseOrder());
// or:
List<String> desc = names.stream()
    .sorted(Comparator.reverseOrder())
    .toList();
```

**Case-insensitive:**
```java
names.sort(String.CASE_INSENSITIVE_ORDER);
// or: Comparator.comparing(String::toLowerCase)
```

---

### Q10. Sort employees by salary (ascending / descending)
**Answer:**
```java
// Ascending
List<Employee> bySalaryAsc = employees.stream()
    .sorted(Comparator.comparing(Employee::salary))
    .toList();

// Descending
List<Employee> bySalaryDesc = employees.stream()
    .sorted(Comparator.comparing(Employee::salary).reversed())
    .toList();

// In-place
employees.sort(Comparator.comparing(Employee::salary));
```

**Follow-up:** "`Comparator.comparing` vs `Comparator.comparingDouble`?" → For `double salary`, `comparingDouble(Employee::salary)` avoids boxing; for `record` accessor returning `double`, both work — `comparingDouble` is slightly more efficient.

---

### Q11. Sort employees by multiple fields (salary → name → age)
**Answer:**
```java
List<Employee> sorted = employees.stream()
    .sorted(Comparator
        .comparing(Employee::salary).reversed()          // highest salary first
        .thenComparing(Employee::name)                   // then name A-Z
        .thenComparing(Employee::age))                   // then age
    .toList();
```

**Null-safe (production):**
```java
Comparator.comparing(Employee::department, Comparator.nullsLast(String::compareTo))
```

**Senior tip:** `thenComparing` chains lexicographically — same pattern as SQL `ORDER BY salary DESC, name ASC, age ASC`.

---

### Q12. Find employees with salary greater than X
**Answer:**
```java
double threshold = 100_000;

List<Employee> highEarners = employees.stream()
    .filter(e -> e.salary() > threshold)
    .toList();
```

**With Predicate (reusable):**
```java
Predicate<Employee> highEarner = e -> e.salary() > 100_000;
List<Employee> result = employees.stream().filter(highEarner).toList();
```

---

### Q13. Find the highest-paid employee
**Answer:**
```java
Optional<Employee> top = employees.stream()
    .max(Comparator.comparing(Employee::salary));
// Optional[Alice or Eve — both 120_000; max returns first encountered in stable order]
```

**If multiple share max and you need all:**
```java
double maxSalary = employees.stream()
    .mapToDouble(Employee::salary)
    .max()
    .orElseThrow();

List<Employee> allTop = employees.stream()
    .filter(e -> e.salary() == maxSalary)
    .toList();
```

---

### Q14. Find the second-highest-paid employee
**Answer:**
```java
Optional<Employee> secondHighest = employees.stream()
    .sorted(Comparator.comparing(Employee::salary).reversed())
    .collect(Collectors.toList())          // materialize sorted list
    .stream()
    .map(Employee::salary)
    .distinct()
    .skip(1)
    .findFirst()
    .flatMap(sal -> employees.stream()
        .filter(e -> e.salary() == sal)
        .findFirst());
```

**Cleaner — distinct salaries first:**
```java
OptionalDouble secondSalary = employees.stream()
    .mapToDouble(Employee::salary)
    .distinct()
    .boxed()
    .sorted(Comparator.reverseOrder())
    .skip(1)
    .findFirst()
    .map(Double::doubleValue);

Optional<Employee> secondPaid = secondSalary
    .flatMap(sal -> employees.stream()
        .filter(e -> e.salary() == sal)
        .findFirst());
// Optional[Carol — 110_000]
```

---

### Q15. Find highest salary in each department ⭐ (Top senior question)
**Answer:**
```java
Map<String, Optional<Employee>> topByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::department,
        Collectors.maxBy(Comparator.comparing(Employee::salary))
    ));
// Engineering → Alice/Eve (120k), Sales → Carol (110k)
```

**If you only need the salary value:**
```java
Map<String, Double> maxSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::department,
        Collectors.collectingAndThen(
            Collectors.maxBy(Comparator.comparing(Employee::salary)),
            opt -> opt.map(Employee::salary).orElse(0.0)
        )
    ));
```

**All employees tied for max in each dept:**
```java
Map<String, List<Employee>> topEarnersByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department))
    .entrySet().stream()
    .collect(Collectors.toMap(
        Map.Entry::getKey,
        e -> {
            double max = e.getValue().stream()
                .mapToDouble(Employee::salary).max().orElse(0);
            return e.getValue().stream()
                .filter(emp -> emp.salary() == max)
                .toList();
        }
    ));
```

---

### Q16. Group employees by department
**Answer:**
```java
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department));
// Engineering → [Alice, Bob, Eve], Sales → [Carol, Dave]
```

**Sorted groups:**
```java
Map<String, List<Employee>> sortedKeys = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::department,
        TreeMap::new,                    // sorted key order
        Collectors.toList()
    ));
```

---

### Q17. Count employees in each department
**Answer:**
```java
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::department, Collectors.counting()));
// Engineering=3, Sales=2
```

**Alternative:**
```java
Map<String, Long> count = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::department,
        Collectors.summingLong(e -> 1)
    ));
```

---

### Q18. Find average salary by department
**Answer:**
```java
Map<String, Double> avgByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::department,
        Collectors.averagingDouble(Employee::salary)
    ));
// Engineering=111666.67, Sales=97500.0
```

**With employee names per dept (downstream collector):**
```java
Map<String, Double> avg = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::department,
        Collectors.averagingDouble(Employee::salary)
    ));
```

---

### Q19. Partition employees based on salary (e.g., > 50,000 vs <= 50,000)
**Answer:**
```java
Map<Boolean, List<Employee>> partitioned = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.salary() > 50_000));
// true  → all employees (all > 50k in sample)
// false → empty list

// Realistic threshold:
Map<Boolean, List<Employee>> by100k = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.salary() > 100_000));
// true → [Alice, Carol, Eve], false → [Bob, Dave]
```

**Count per partition:**
```java
Map<Boolean, Long> count = employees.stream()
    .collect(Collectors.partitioningBy(
        e -> e.salary() > 100_000,
        Collectors.counting()
    ));
```

**Key insight:** `partitioningBy` always produces **exactly two keys** — `true` and `false` — even if one bucket is empty. `groupingBy` only creates keys that exist.

---

### Q20. Convert List to Map — handle duplicate keys
**Answer — simple toMap (throws on duplicate key):**
```java
List<Employee> list = employees;

Map<String, Employee> nameToEmployee = list.stream()
    .collect(Collectors.toMap(Employee::name, Function.identity()));
// DuplicateKeyException if two employees share a name
```

**Handle duplicates — keep first:**
```java
Map<String, Employee> keepFirst = list.stream()
    .collect(Collectors.toMap(
        Employee::name,
        Function.identity(),
        (existing, replacement) -> existing    // merge function on duplicate key
    ));
```

**Keep higher salary on duplicate name:**
```java
Map<String, Employee> keepHigherPaid = list.stream()
    .collect(Collectors.toMap(
        Employee::name,
        Function.identity(),
        (e1, e2) -> e1.salary() >= e2.salary() ? e1 : e2
    ));
```

**Group into List (no duplicate key problem):**
```java
Map<String, List<Employee>> deptMap = list.stream()
    .collect(Collectors.groupingBy(Employee::department));
```

**Senior trap:** `toMap` without merge function throws `IllegalStateException: Duplicate key` — always mention the 3-arg overload. Also specify map supplier for ordering: `() -> new LinkedHashMap<>()`.

---

## SECTION 2: STRING-BASED PROGRAMS (Q21–Q30)

### Q21. Reverse a String using Java 8
**Answer:**
```java
String input = "hello";

// Stream approach
String reversed = new StringBuilder(input).reverse().toString();  // still valid

// Pure Stream (interview answer)
String streamReversed = input.chars()
    .mapToObj(c -> String.valueOf((char) c))
    .reduce((a, b) -> b + a)
    .orElse("");
// "olleh"
```

**Using Collectors:**
```java
String rev = input.chars()
    .collect(StringBuilder::new,
             StringBuilder::appendCodePoint,
             StringBuilder::append)
    .reverse()
    .toString();
```

---

### Q22. Check whether a String is a palindrome
**Answer:**
```java
public static boolean isPalindrome(String s) {
    String cleaned = s.replaceAll("[^a-zA-Z0-9]", "").toLowerCase();
    return IntStream.range(0, cleaned.length() / 2)
        .allMatch(i -> cleaned.charAt(i) == cleaned.charAt(cleaned.length() - 1 - i));
}

// One-liner
boolean pal = IntStream.range(0, s.length() / 2)
    .allMatch(i -> s.charAt(i) == s.charAt(s.length() - 1 - i));
```

**Stream compare forward vs reversed:**
```java
boolean isPal = cleaned.chars()
    .mapToObj(c -> (char) c)
    .collect(StringBuilder::new, StringBuilder::append, StringBuilder::append)
    .toString()
    .equals(
        new StringBuilder(cleaned).reverse().toString()
    );
```

---

### Q23. Count vowels and consonants
**Answer:**
```java
String s = "Hello World";

long vowels = s.chars()
    .mapToObj(c -> (char) Character.toLowerCase(c))
    .filter(c -> "aeiou".indexOf(c) >= 0)
    .count();

long consonants = s.chars()
    .mapToObj(c -> (char) Character.toLowerCase(c))
    .filter(Character::isLetter)
    .filter(c -> "aeiou".indexOf(c) < 0)
    .count();
```

**Partitioning approach:**
```java
Map<Boolean, Long> counts = s.chars()
    .mapToObj(c -> (char) Character.toLowerCase(c))
    .filter(Character::isLetter)
    .collect(Collectors.partitioningBy(
        c -> "aeiou".indexOf(c) >= 0,
        Collectors.counting()
    ));
// true=vowels, false=consonants
```

---

### Q24. Find duplicate characters in a String
**Answer:**
```java
String s = "programming";

List<Character> duplicates = s.chars()
    .mapToObj(c -> (char) c)
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()))
    .entrySet().stream()
    .filter(e -> e.getValue() > 1)
    .map(Map.Entry::getKey)
    .toList();
// [r, g, m]
```

---

### Q25. Find character frequency
*(Same as Q3 — included for String context)*
```java
Map<Character, Long> freq = "hello".chars()
    .mapToObj(c -> (char) c)
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
// {h=1, e=1, l=2, o=1}
```

---

### Q26. Remove duplicate characters from a String
**Answer — preserve first occurrence order:**
```java
String s = "programming";

String unique = s.chars()
    .mapToObj(c -> (char) c)
    .collect(Collectors.toCollection(LinkedHashSet::new))
    .stream()
    .map(String::valueOf)
    .collect(Collectors.joining());
// "progamin"
```

**Using distinct (works because IntStream boxes consistently):**
```java
String result = s.chars()
    .distinct()
    .collect(StringBuilder::new, StringBuilder::appendCodePoint, StringBuilder::append)
    .toString();
// Note: distinct on IntStream may not preserve order in all JDK versions for chars — LinkedHashSet is safer
```

---

### Q27. Find the longest word in a String
**Answer:**
```java
String sentence = "Java 8 streams make code concise";

Optional<String> longest = Arrays.stream(sentence.split("\\s+"))
    .max(Comparator.comparingInt(String::length));
// Optional["concise"]
```

**With max length value:**
```java
int maxLen = Arrays.stream(sentence.split("\\s+"))
    .mapToInt(String::length)
    .max()
    .orElse(0);
```

---

### Q28. Find the longest substring without repeating characters
**Answer:** Classic sliding window — not a pure one-liner Stream problem, but interviewers expect this algorithm:
```java
public static String longestUniqueSubstring(String s) {
    Map<Character, Integer> lastIndex = new HashMap<>();
    int start = 0, maxLen = 0, maxStart = 0;

    for (int i = 0; i < s.length(); i++) {
        char c = s.charAt(i);
        if (lastIndex.containsKey(c) && lastIndex.get(c) >= start) {
            start = lastIndex.get(c) + 1;
        }
        lastIndex.put(c, i);
        if (i - start + 1 > maxLen) {
            maxLen = i - start + 1;
            maxStart = start;
        }
    }
    return s.substring(maxStart, maxStart + maxLen);
}
// "abcabcbb" → "abc", "programming" → "progin" or "rogamin" depending on tie
```

**Interview tip:** Acknowledge this is **O(n) sliding window**, not naturally suited to Stream pipelines — trying to force Streams here shows lack of judgment. Mention Java 8 for helper logic only.

---

### Q29. Check whether two Strings are anagrams
**Answer:**
```java
public static boolean areAnagrams(String a, String b) {
    if (a.length() != b.length()) return false;

    Map<Character, Long> freqA = a.chars()
        .mapToObj(c -> (char) Character.toLowerCase(c))
        .filter(Character::isLetterOrDigit)
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));

    Map<Character, Long> freqB = b.chars()
        .mapToObj(c -> (char) Character.toLowerCase(c))
        .filter(Character::isLetterOrDigit)
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));

    return freqA.equals(freqB);
}
```

**Shortcut — sort and compare:**
```java
boolean anagram = Arrays.equals(
    a.chars().sorted().toArray(),
    b.chars().sorted().toArray()
);
```

---

### Q30. Find the number of words in a String
**Answer:**
```java
String s = "  Java 8   Stream API  ";

// Split on whitespace
long wordCount = Arrays.stream(s.trim().split("\\s+"))
    .filter(w -> !w.isEmpty())
    .count();
// 4

// Using StringTokenizer-style filter
long count = Pattern.compile("\\s+")
    .splitAsStream(s.trim())
    .filter(w -> !w.isBlank())   // isBlank is Java 11+ — use !w.isEmpty() for Java 8
    .count();
```

**Java 8 only:**
```java
long wc = Arrays.stream(s.trim().split("\\s+"))
    .filter(w -> !w.isEmpty())
    .count();
```

---

## SECTION 3: ARRAY / LIST PROGRAMS (Q31–Q40)

### Q31. Find missing number from an array (1 to n)
**Answer:** Expected sum = `n*(n+1)/2`; actual sum subtracted gives missing number.
```java
int[] arr = {1, 2, 4, 5, 6};   // n=6, missing 3

int n = arr.length + 1;
int expectedSum = n * (n + 1) / 2;
int actualSum = Arrays.stream(arr).sum();
int missing = expectedSum - actualSum;   // 3
```

**XOR alternative (senior):** XOR all 1..n and all array elements — result is missing number. O(1) space.

---

### Q32. Find duplicate number in an array
**Answer:**
```java
int[] arr = {1, 3, 4, 2, 3, 5};

OptionalInt duplicate = Arrays.stream(arr)
    .boxed()
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()))
    .entrySet().stream()
    .filter(e -> e.getValue() > 1)
    .map(Map.Entry::getKey)
    .findFirst()
    .map(Integer::intValue);
// Optional[3]
```

**Set approach (Floyd's cycle for "one duplicate in 1..n" is a separate classic problem):**
```java
Set<Integer> seen = new HashSet<>();
int dup = Arrays.stream(arr).filter(n -> !seen.add(n)).findFirst().orElse(-1);
```

---

### Q33. Find common elements between two Lists
**Answer:**
```java
List<Integer> a = List.of(1, 2, 3, 4, 5);
List<Integer> b = List.of(3, 4, 5, 6, 7);

List<Integer> common = a.stream()
    .filter(b::contains)
    .toList();
// [3, 4, 5]
```

**Efficient for large lists — use Set:**
```java
Set<Integer> setB = new HashSet<>(b);
List<Integer> commonFast = a.stream()
    .filter(setB::contains)
    .distinct()
    .toList();
```

---

### Q34. Find union of two Lists
**Answer:**
```java
List<Integer> union = Stream.concat(a.stream(), b.stream())
    .distinct()
    .toList();
// [1, 2, 3, 4, 5, 6, 7]
```

**Preserve order — a first, then b-only:**
```java
Set<Integer> seen = new HashSet<>();
List<Integer> orderedUnion = Stream.concat(a.stream(), b.stream())
    .filter(seen::add)
    .toList();
```

---

### Q35. Find intersection of two Lists
**Answer:**
```java
Set<Integer> setB = new HashSet<>(b);
List<Integer> intersection = a.stream()
    .filter(setB::contains)
    .distinct()
    .toList();
// [3, 4, 5]
```

**Symmetric — elements in both:**
```java
Set<Integer> setA = new HashSet<>(a);
List<Integer> both = b.stream().filter(setA::contains).distinct().toList();
```

---

### Q36. Merge two Lists and remove duplicates
**Answer:**
```java
List<Integer> merged = Stream.concat(list1.stream(), list2.stream())
    .distinct()
    .toList();
```

**Into existing mutable list:**
```java
List<Integer> result = new ArrayList<>(list1);
list2.stream().filter(e -> !result.contains(e)).forEach(result::add);
// Better: use Set for O(n) dedup
List<Integer> efficient = new ArrayList<>(
    Stream.concat(list1.stream(), list2.stream())
        .collect(Collectors.toCollection(LinkedHashSet::new))
);
```

---

### Q37. Find numbers starting with `1`
**Answer:**
```java
List<Integer> numbers = List.of(10, 21, 102, 15, 1, 99, 1001);

List<Integer> startsWithOne = numbers.stream()
    .filter(n -> String.valueOf(n).startsWith("1"))
    .toList();
// [10, 102, 15, 1, 1001]
```

**Starts with "1" as prefix — note 15 does NOT start with "1":**
```java
// Correct: [10, 102, 1, 1001] only
List<Integer> correct = numbers.stream()
    .filter(n -> String.valueOf(Math.abs(n)).startsWith("1"))
    .toList();
```

**Interview trap:** Clarify whether "starts with 1" means string prefix `"1"` or first digit — `15` starts with digit `1` but string `"15".startsWith("1")` is **true**. Be explicit in interviews.

---

### Q38. Separate even and odd numbers
**Answer:**
```java
List<Integer> nums = List.of(1, 2, 3, 4, 5, 6, 7, 8);

Map<Boolean, List<Integer>> evenOdd = nums.stream()
    .collect(Collectors.partitioningBy(n -> n % 2 == 0));
// true → [2, 4, 6, 8], false → [1, 3, 5, 7]

List<Integer> evens = nums.stream().filter(n -> n % 2 == 0).toList();
List<Integer> odds  = nums.stream().filter(n -> n % 2 != 0).toList();
```

---

### Q39. Find the sum of even numbers
**Answer:**
```java
int evenSum = nums.stream()
    .filter(n -> n % 2 == 0)
    .mapToInt(Integer::intValue)
    .sum();
// 20
```

**With reduce:**
```java
int sum = nums.stream()
    .filter(n -> n % 2 == 0)
    .reduce(0, Integer::sum);
```

---

### Q40. Find maximum/minimum element using Streams
**Answer:**
```java
OptionalInt max = nums.stream().mapToInt(Integer::intValue).max();
OptionalInt min = nums.stream().mapToInt(Integer::intValue).min();

// Custom objects
Optional<Employee> highestPaid = employees.stream()
    .max(Comparator.comparing(Employee::salary));
```

**Empty list safety:**
```java
int maxVal = nums.stream()
    .mapToInt(Integer::intValue)
    .max()
    .orElseThrow(() -> new IllegalArgumentException("Empty list"));
```

---

## SECTION 4: TOP 10 PRIORITY PROGRAMS — QUICK REVISION (Senior)

| # | Problem | Core API | One-liner pattern |
|---|---|---|---|
| 1 | Highest salary by department | `groupingBy` + `maxBy` | `.collect(groupingBy(Employee::department, maxBy(comparing(salary))))` |
| 2 | Second-highest salary | `distinct` + `sorted` + `skip(1)` | `.map(salary).distinct().sorted(reverseOrder()).skip(1).findFirst()` |
| 3 | Group by department | `groupingBy` | `.collect(groupingBy(Employee::department))` |
| 4 | Duplicate elements | `groupingBy` + count > 1 | `.collect(groupingBy(identity(), counting()))` filter value > 1 |
| 5 | First non-repeated char | count then `findFirst` | two-pass: freq map → filter count==1 |
| 6 | Frequency map | `groupingBy` + `counting` | `.collect(groupingBy(identity(), counting()))` |
| 7 | Sort employees | `Comparator.comparing` | `.sorted(comparing(salary).reversed().thenComparing(name))` |
| 8 | List to Map (dup keys) | `toMap` + merge fn | `.collect(toMap(key, identity, (a,b) -> a))` |
| 9 | Second-highest number | `distinct` + `sorted` + `skip` | same as #2 for integers |
| 10 | Partition by condition | `partitioningBy` | `.collect(partitioningBy(e -> e.salary() > 100_000))` |

---

## SECTION 5: FOLLOW-UP QUESTIONS INTERVIEWERS ASK AFTER THESE PROGRAMS

### Q41. Explain `map()` vs `flatMap()` with an example from these programs.
**Answer:**
- **`map`** — 1-to-1 transform: `Employee → salary (Double)`, `String → length (Integer)`.
- **`flatMap`** — 1-to-many, then flatten: `Order → Stream<OrderLine>`, `Optional<User> → Optional<Address>`.

```java
// map
List<Double> salaries = employees.stream().map(Employee::salary).toList();

// flatMap — all order lines across orders
List<LineItem> allItems = orders.stream()
    .flatMap(order -> order.getLines().stream())
    .toList();
```

---

### Q42. When would you use `reduce()` instead of `collect()`?
**Answer:** `reduce` for **single immutable result** (sum, max, concatenation). `collect` for **mutable containers** (List, Map, Set) or complex aggregations via `Collector` (groupingBy, partitioningBy). Example: sum → `mapToInt().sum()` or `reduce(0, Integer::sum)`; group by dept → must use `collect(groupingBy(...))`.

---

### Q43. What is the difference between `map()` and `mapToInt()`?
**Answer:** `map()` returns `Stream<R>` (boxed). `mapToInt()` returns `IntStream` (primitive) — avoids boxing overhead, provides `sum()`, `average()`, `max()` directly. Prefer `mapToInt` for numeric pipelines on large data.

---

### Q44. How does `Collectors.groupingBy()` differ from `partitioningBy()`?
| | groupingBy | partitioningBy |
|---|---|---|
| Keys | Arbitrary (user-defined classifier) | Always `Boolean` (true/false) |
| Key count | Only keys that exist in data | Always 2 keys (true + false) |
| Example | Group by department | Salary > 100k vs <= 100k |

---

### Q45. What happens if `findFirst()` returns empty? How do you handle it?
**Answer:** Returns `Optional.empty()` — never returns null. Handle with:
```java
.orElse(defaultValue)
.orElseGet(() -> computeDefault())
.orElseThrow(() -> new NotFoundException())
.ifPresent(value -> ...)
```
**Never** call `.get()` without checking — throws `NoSuchElementException`.

---

## SECTION 6: IMPORTS CHEAT SHEET

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;
import java.util.stream.Collectors;
```

---

*End of Java 8 Programming Problems — Topic 7*

*Study order: Master Section 4 (Top 10) first → then String programs (Q4, Q5, Q24, Q25) → then Array/List set operations → full Employee pipeline suite (Q10–Q20).*

# SENIOR EXPANSION — CODING PROBLEM DEPTH

> The original programming problems are preserved above. This section adds the interviewer's follow-up layer: optimal approach, complexity, edge cases, Java 8 implementation patterns, and production-oriented reasoning.

## Q46. Find the first non-repeated character
### Approach
Count characters while preserving encounter order, then return the first character with count 1.

```java
String firstUnique(String s) {
    Map<Character, Long> counts =
        s.chars().mapToObj(c -> (char)c)
          .collect(Collectors.groupingBy(
              Function.identity(),
              LinkedHashMap::new,
              Collectors.counting()));

    return counts.entrySet().stream()
        .filter(e -> e.getValue() == 1)
        .map(Map.Entry::getKey)
        .findFirst()
        .map(String::valueOf)
        .orElse(null);
}
```

**Complexity:** O(n) time, O(k) space where k is the character domain represented.

**Follow-up:** Why `LinkedHashMap`? Because normal `HashMap` does not provide encounter-order semantics.

## Q47. Find the first duplicate element in an array
### Answer
Use a `HashSet` and return when insertion fails.

```java
Integer firstDuplicate(int[] a) {
    Set<Integer> seen = new HashSet<>();
    for (int x : a) {
        if (!seen.add(x)) return x;
    }
    return null;
}
```

**Time:** O(n) average. **Space:** O(n).

## Q48. Find the top K largest numbers
### Answer
For large input and small K, use a min-heap of size K.

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();

for (int x : numbers) {
    pq.offer(x);
    if (pq.size() > k) pq.poll();
}
```

**Time:** O(n log k), **space:** O(k).

Senior follow-up: sorting gives O(n log n), which may be simpler but is unnecessary when K is much smaller than n.

## Q49. Find the second-highest distinct salary
### Answer
Use a distinct descending ordering and skip the first value.

```java
Optional<Integer> second =
    employees.stream()
        .map(Employee::getSalary)
        .filter(Objects::nonNull)
        .distinct()
        .sorted(Comparator.reverseOrder())
        .skip(1)
        .findFirst();
```

**Trap:** Clarify whether duplicate salaries count as separate positions. Interviewers often test "second highest" vs "second highest distinct".

For very large datasets, a full sort may be unnecessary; maintain the top two distinct values in one pass.

## Q50. Longest substring without repeating characters
### Approach
Use a sliding window and last-seen positions.

```java
int longestUnique(String s) {
    Map<Character, Integer> last = new HashMap<>();
    int left = 0, best = 0;

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (last.containsKey(c)) {
            left = Math.max(left, last.get(c) + 1);
        }
        last.put(c, right);
        best = Math.max(best, right - left + 1);
    }
    return best;
}
```

**Time:** O(n), **space:** O(k).

**Senior follow-up:** If Unicode code points rather than UTF-16 `char` values matter, the representation must be reconsidered.

## Q51. Merge overlapping intervals
### Approach
Sort by start time, then scan and merge overlaps.

```java
intervals.sort(Comparator.comparingInt(a -> a[0]));

List<int[]> result = new ArrayList<>();
for (int[] current : intervals) {
    if (result.isEmpty() ||
        result.get(result.size() - 1)[1] < current[0]) {
        result.add(current.clone());
    } else {
        result.get(result.size() - 1)[1] =
            Math.max(result.get(result.size() - 1)[1], current[1]);
    }
}
```

**Complexity:** O(n log n) due to sorting; O(n) output space.

**Follow-up:** Clarify whether touching intervals such as `[1,2]` and `[2,3]` count as overlapping.

## Q52. Find the missing number from 1..n
### Answer
XOR or arithmetic sum can solve it in O(n) time and O(1) extra space.

XOR avoids integer-sum overflow concerns:

```java
int missing(int[] a) {
    int result = a.length + 1;
    for (int i = 0; i < a.length; i++) {
        result ^= (i + 1) ^ a[i];
    }
    return result;
}
```

Clarify the input contract before coding: exactly one missing value, no duplicates, range 1..n.

## Q53. Remove duplicates while preserving order
### Answer
Use `LinkedHashSet`:

```java
List<Integer> unique =
    new ArrayList<>(new LinkedHashSet<>(numbers));
```

For a stream:

```java
List<Integer> unique =
    numbers.stream().distinct().toList();
```

The latter preserves encounter order for an ordered sequential stream.

## Q54. Group employees by department and find the highest salary
### Answer
```java
Map<String, Optional<Employee>> result =
    employees.stream()
        .collect(Collectors.groupingBy(
            Employee::getDepartment,
            Collectors.maxBy(Comparator.comparing(Employee::getSalary))));
```

If the business rule requires a concrete employee rather than Optional, define how an empty department should behave rather than blindly calling `get()`.

## Q55. Partition numbers into even and odd
### Answer
```java
Map<Boolean, List<Integer>> result =
    numbers.stream()
        .collect(Collectors.partitioningBy(n -> n % 2 == 0));
```

`partitioningBy()` communicates a boolean split more directly than `groupingBy()`.

## Q56. Find common elements of two large lists
### Answer
If order is irrelevant and duplicates are irrelevant, put one side into a `HashSet` and filter the other.

```java
Set<Integer> set = new HashSet<>(a);
List<Integer> common =
    b.stream().filter(set::contains).distinct().toList();
```

**Average time:** O(n + m). **Space:** O(n).

Clarify whether duplicates and output order matter.

# INTERVIEWER FOLLOW-UPS

## Q57. When should you prefer a loop over Streams?
**Answer:** When the algorithm is stateful, performance-sensitive, requires early mutation, or the stream expression would become less readable. Senior developers should optimize for correctness and maintainability, not for maximum use of Streams.

## Q58. How do you explain complexity during an interview?
Use:
```text
Approach
→ dominant operation
→ time complexity
→ extra space
→ edge cases
→ why this is better than the naive solution
```

Example:
> "Sorting dominates at O(n log n), then the scan is O(n), so overall O(n log n)."

## Q59. What edge cases should you proactively mention?
- null input if allowed
- empty input
- one element
- duplicates
- negative numbers
- integer overflow
- already sorted input
- all values identical
- very large input
- Unicode vs ASCII assumptions
- whether ordering matters

## Q60. What makes a coding answer senior-level?
A senior answer does not stop at code. It explains:
1. assumptions
2. brute-force approach
3. optimized approach
4. complexity
5. edge cases
6. correctness reasoning
7. trade-offs
8. production implications

# TOP CODING PATTERNS TO MASTER

| Pattern | Problems |
|---|---|
| Hashing | duplicates, frequency, two-sum, anagrams |
| Sliding window | longest substring, max window |
| Two pointers | palindrome, sorted arrays, pair problems |
| Stack | balanced brackets, next greater element |
| Queue/BFS | level traversal, shortest unweighted path |
| Heap | top K, running median |
| Sorting + scan | intervals, merging |
| Binary search | sorted lookup, answer-space search |
| Recursion/backtracking | subsets, permutations |
| Dynamic programming | coin change, LIS, knapsack-style problems |

# SENIOR CODING RAPID-FIRE

1. Ask for constraints before choosing the algorithm.
2. Do not use sorting when a hash-based O(n) approach solves the problem and ordering is irrelevant.
3. Do not use a HashSet if duplicates are part of the required output semantics.
4. Clarify whether "second highest" means distinct.
5. Explain integer overflow when using arithmetic sums.
6. Prefer `PriorityQueue` for top-K when K is small relative to N.
7. Preserve ordering explicitly when the API requires it.
8. Use `mapToInt`/`mapToLong` when primitive stream performance or numeric semantics matter.
9. A correct O(n log n) solution is often preferable to a fragile "clever" O(n) solution when constraints permit.
10. Always state complexity and edge cases.
