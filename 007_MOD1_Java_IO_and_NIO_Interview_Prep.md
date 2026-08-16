# 006. Java I/O & NIO — Interview Questions & Answers — Senior Java Developer

**Focus:** Streams, Byte/Character/Buffered I/O, Serialization, `transient`, `Externalizable`, NIO Channels, Buffers, Selectors, `Path` & `Files`, file performance, resource management, and production scenarios.

---

# SECTION 1: JAVA I/O FUNDAMENTALS

### Q1. What is Java I/O?

**Answer:** Java I/O provides APIs for reading and writing data between an application and external resources such as files, network connections, memory, console, and other streams.

The traditional I/O API is primarily in `java.io`. The main abstractions are:

- `InputStream` / `OutputStream` → byte-oriented I/O
- `Reader` / `Writer` → character-oriented I/O
- Buffered streams → improve efficiency by reducing underlying I/O operations
- Object streams → Java object serialization

**Senior answer:** I/O is fundamentally about moving data between the application and an external resource. I select the abstraction based on the data type and access pattern—byte streams for binary data, character streams for text, buffering for efficient sequential access, and NIO channels/path APIs when I need more advanced or scalable I/O.

---

### Q2. What is the difference between byte streams and character streams?

**Answer:**

| Byte Streams | Character Streams |
|---|---|
| `InputStream` / `OutputStream` | `Reader` / `Writer` |
| Work with bytes | Work with characters |
| Suitable for binary data | Suitable for text |
| No character decoding by itself | Handles character decoding/encoding |
| Images, PDFs, ZIPs, audio | JSON, CSV, XML, TXT |

```java
try (InputStream in = new FileInputStream("image.png")) {
    // binary data
}

try (Reader reader = new FileReader("data.txt")) {
    // text data
}
```

**Interview trap:** A character is not necessarily one byte. Text should be decoded using a known charset such as UTF-8.

---

### Q3. Why does InputStream.read() return int instead of byte?

**Answer:** `read()` must represent both every possible byte value (`0` to `255`) and end-of-stream (`-1`). Therefore the return type is `int`.

```java
int data;

while ((data = input.read()) != -1) {
    // process data
}
```

If it returned `byte`, there would be no separate value available to represent EOF. The `int` return type is specifically required so the API can represent all unsigned byte values plus the EOF sentinel `-1`.

---

### Q4. What does read() return when the end of a stream is reached?

**Answer:** It returns `-1`.

```java
int value;
while ((value = input.read()) != -1) {
    process(value);
}
```

For bulk reads, the method can return the number of bytes/characters read, or `-1` at EOF.

---

### Q5. What is the difference between InputStream and Reader?

**Answer:** `InputStream` reads bytes, while `Reader` reads characters.

```text
InputStream
    ↓
bytes

Reader
    ↓
characters
```

Typical examples: `FileInputStream` → binary data; `InputStreamReader` → converts bytes to characters; `FileReader` → file-based character reader; `BufferedReader` → buffered text processing.

**Senior answer:** For text, I prefer an explicit charset through `InputStreamReader` or `Files.newBufferedReader()` rather than depending on an implicit platform encoding.

---

### Q6. What is buffering and why does it improve I/O performance?

**Answer:** Buffering means temporarily storing data in memory so that many small application-level reads/writes can be combined into fewer underlying I/O operations.

```java
try (BufferedInputStream in =
         new BufferedInputStream(new FileInputStream("data.bin"))) {

    byte[] buffer = new byte[8192];
    int count;

    while ((count = in.read(buffer)) != -1) {
        process(buffer, count);
    }
}
```

Without buffering, frequent small reads may result in many expensive system-level operations. Buffering mainly reduces the frequency of expensive underlying I/O operations; the right buffer size depends on the workload and should be measured rather than guessed.

---

### Q7. What is the difference between BufferedInputStream and BufferedReader?

**Answer:** `BufferedInputStream` buffers **bytes**. `BufferedReader` buffers **characters**.

```text
BufferedInputStream
    -> binary data

BufferedReader
    -> text data
```

`BufferedReader` also provides convenient line-based processing: `String line = reader.readLine();`.

---

### Q8. What is the difference between flush() and close()?

**Answer:** `flush()` pushes buffered output to the underlying destination but keeps the resource open. `close()` releases the resource and normally performs a final flush first.

```java
writer.write("Hello");
writer.flush(); // writer remains open

writer.close(); // resource released
```

**Interview trap:** Calling `flush()` does not mean the stream is closed.

---

### Q9. What is try-with-resources and why should it be used?

**Answer:** Try-with-resources automatically closes objects implementing `AutoCloseable`.

```java
try (BufferedReader reader =
         Files.newBufferedReader(Path.of("data.txt"),
                                  StandardCharsets.UTF_8)) {

    System.out.println(reader.readLine());
}
```

Benefits: prevents resource leaks, works correctly when exceptions occur, makes cleanup concise, preserves suppressed exceptions.

**Senior answer:** For files, streams, channels, readers, writers, and directory streams, I normally use try-with-resources so resource ownership and cleanup are explicit.

---

### Q10. What are suppressed exceptions in try-with-resources?

**Answer:** If the try block throws an exception and resource closing also throws an exception, the try-block exception remains the primary exception and the close exception becomes suppressed.

```java
try (SomeResource resource = new SomeResource()) {
    resource.process();
} catch (Exception e) {
    for (Throwable suppressed : e.getSuppressed()) {
        System.out.println(suppressed);
    }
}
```

This prevents the cleanup exception from hiding the original failure.

---

# SECTION 2: CHARACTER ENCODING & TEXT I/O

### Q11. What is character encoding, and why is it important in Java I/O?

**Answer:** Character encoding defines how characters are represented as bytes.

```text
Characters
    ↓ UTF-8 encoding
Bytes

Bytes
    ↓ UTF-8 decoding
Characters
```

Use an explicit charset:

```java
Files.readString(path, StandardCharsets.UTF_8);
```

**Senior answer:** Encoding bugs often appear only when non-ASCII data reaches production. I therefore make the charset explicit at file, API, messaging, and database boundaries.

---

### Q12. What is the difference between FileReader/FileWriter and InputStreamReader/OutputStreamWriter?

**Answer:** `InputStreamReader` and `OutputStreamWriter` are bridges between byte streams and character streams and allow explicit charset selection.

```java
Reader reader = new InputStreamReader(
        new FileInputStream("data.txt"),
        StandardCharsets.UTF_8);
```

For modern code, explicit charset handling is preferred.

---

### Q13. What is BufferedReader.readLine(), and what should you consider when processing large files?

**Answer:** `readLine()` reads one line at a time.

```java
try (BufferedReader reader =
         Files.newBufferedReader(path, StandardCharsets.UTF_8)) {

    String line;
    while ((line = reader.readLine()) != null) {
        process(line);
    }
}
```

It avoids loading the complete file into memory. Important: a very large individual line can still consume significant memory.

---

### Q14. What is the difference between Files.readAllLines(), Files.lines(), and BufferedReader?

**Answer:**

| API | Behavior | Large file |
|---|---|---|
| `readAllLines()` | Loads all lines | Avoid unless appropriate |
| `lines()` | Lazy stream | Good |
| `BufferedReader` | Sequential line processing | Good |

```java
try (Stream<String> lines = Files.lines(path, StandardCharsets.UTF_8)) {
    lines.filter(this::isValid)
         .forEach(this::process);
}
```

**Interview trap:** `Files.lines()` returns a resource-backed stream and should be closed.

---

# SECTION 3: SERIALIZATION

### Q15. What is Java serialization?

**Answer:** Serialization converts an object's state into a byte stream so it can be stored or transferred.

```java
class User implements Serializable {
    private static final long serialVersionUID = 1L;

    private String name;
    private int age;
}
```

Serialization:

```java
try (ObjectOutputStream out =
         new ObjectOutputStream(new FileOutputStream("user.ser"))) {
    out.writeObject(user);
}
```

Deserialization:

```java
try (ObjectInputStream in =
         new ObjectInputStream(new FileInputStream("user.ser"))) {
    User user = (User) in.readObject();
}
```

**Senior answer:** Java serialization is useful for specific Java-to-Java compatibility scenarios, but I would not use native Java serialization as an untrusted network payload format. For service communication I generally prefer explicit formats such as JSON or Protobuf.

---

### Q16. What is serialVersionUID?

**Answer:** `serialVersionUID` is the version identifier used to check serialization compatibility.

```java
private static final long serialVersionUID = 1L;
```

If the serialized object's class and current class are incompatible, deserialization can fail with `InvalidClassException`. I explicitly declare it because the compiler can otherwise generate one based on class details, and seemingly unrelated class changes can alter the generated value.

---

### Q17. What is transient in Java?

**Answer:** A `transient` field is excluded from default Java serialization.

```java
class User implements Serializable {

    private String username;
    private transient String password;
}
```

After deserialization, `password` gets its default value — `null` for object references, `0` for `int`. Important: `transient` is not encryption. It only prevents the field from participating in default serialization.

---

### Q18. Are static fields serialized?

**Answer:** No. Static fields belong to the class, not to an individual object instance.

```java
class Counter implements Serializable {
    static int count;
    int value;
}
```

`value` is part of instance state and can be serialized. `count` is not.

---

### Q19. What happens to constructors during deserialization?

**Answer:** For normal `Serializable` classes, the serializable class's constructors are not executed during default deserialization. The no-argument constructor of the first non-serializable superclass is invoked. **Interview trap:** do not say "no constructor is called" — the superclass behavior matters.

---

### Q20. Can transient fields be restored after deserialization?

**Answer:** Yes. Custom `readObject()` can reconstruct transient state.

```java
private void readObject(ObjectInputStream in)
        throws IOException, ClassNotFoundException {

    in.defaultReadObject();

    // reconstruct transient state
}
```

This is useful for derived values or runtime-only resources.

---

### Q21. What are writeObject() and readObject()?

**Answer:** They allow custom serialization/deserialization logic.

```java
private void writeObject(ObjectOutputStream out)
        throws IOException {

    out.defaultWriteObject();
    // custom state
}

private void readObject(ObjectInputStream in)
        throws IOException, ClassNotFoundException {

    in.defaultReadObject();
    // restore custom state
}
```

They can transform serialized data, validate state, reconstruct transient fields, and preserve compatibility.

---

### Q22. What are readResolve() and writeReplace()?

**Answer:** They can customize the object instance used during serialization/deserialization. `readResolve()` is commonly discussed with singleton-like objects because it can replace the deserialized object with the desired instance.

```java
private Object readResolve() {
    return INSTANCE;
}
```

`writeReplace()` can return another object to be serialized instead. These methods are powerful but should be used carefully because serialization has complex object-graph semantics.

---

### Q23. What is Externalizable?

**Answer:** `Externalizable` provides explicit control over serialization.

```java
class User implements Externalizable {

    private String name;
    private int age;

    public User() {
        // required public no-arg constructor
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeUTF(name);
        out.writeInt(age);
    }

    @Override
    public void readExternal(ObjectInput in)
            throws IOException, ClassNotFoundException {
        name = in.readUTF();
        age = in.readInt();
    }
}
```

Unlike `Serializable`, the class explicitly controls what is written and read.

---

### Q24. Serializable vs Externalizable — what is the difference?

**Answer:**

| Serializable | Externalizable |
|---|---|
| Mostly automatic | Explicit control |
| Easier to implement | More code |
| `serialVersionUID` commonly used | Explicit read/write methods |
| No public no-arg constructor requirement for serializable class | Public no-arg constructor required |
| Less control | More control |

**Senior answer:** Externalizable gives tighter control over the serialized representation, but it increases implementation responsibility. For modern distributed systems, I would generally prefer an explicit schema-based format instead of relying on either native mechanism.

---

### Q25. What are the security concerns with Java deserialization?

**Answer:** Native Java deserialization should not blindly process untrusted input. Potential risks include gadget-chain attacks, unexpected object creation, denial of service through malicious object graphs, compatibility problems, and difficult-to-audit behavior.

**Production recommendation:** Do not deserialize arbitrary untrusted bytes using `ObjectInputStream`. Prefer safer explicit data formats and validation. If native deserialization is unavoidable, apply strict filtering and trust boundaries.

---

# SECTION 4: JAVA NIO CHANNELS

### Q26. What is Java NIO?

**Answer:** Java NIO provides modern APIs for I/O, filesystem operations, buffers, channels, and non-blocking network communication.

Important areas: `java.nio`, `java.nio.channels`, `java.nio.file`.

Key concepts: Channels, Buffers, Selectors, Path, Files.

---

### Q27. What is a Channel?

**Answer:** A channel represents an open connection to an entity capable of I/O. Common channels: `FileChannel`, `SocketChannel`, `ServerSocketChannel`, `DatagramChannel`. A channel generally transfers data using a `Buffer`.

```text
Channel <----> Buffer
```

---

### Q28. What is FileChannel?

**Answer:** `FileChannel` provides channel-based file operations. Capabilities include read/write, random access, file position, file size, file locking, file transfer, and memory mapping.

```java
try (FileChannel channel =
         FileChannel.open(path, StandardOpenOption.READ)) {

    ByteBuffer buffer = ByteBuffer.allocate(4096);

    while (channel.read(buffer) != -1) {
        buffer.flip();

        while (buffer.hasRemaining()) {
            process(buffer.get());
        }

        buffer.clear();
    }
}
```

---

### Q29. FileInputStream vs FileChannel — when would you use each?

**Answer:** Use `FileInputStream` when simple sequential byte-stream processing is sufficient. Use `FileChannel` when you need random access, file locking, `transferTo()`/`transferFrom()`, memory mapping, or explicit channel/buffer interaction.

**Senior answer:** I don't choose FileChannel merely because it is newer. I choose it when its capabilities match the workload.

---

### Q30. What is transferTo() / transferFrom()?

**Answer:** They allow efficient transfer of file/channel data without manually copying every chunk through application code.

```java
source.transferTo(position, count, target);
```

Useful for large file transfers. Important: production code should account for partial transfers and continue until the desired amount has been transferred.

---

# SECTION 5: NIO BUFFERS

### Q31. What is a Buffer?

**Answer:** A buffer is a container used by NIO channels to hold data being read or written. Common types: `ByteBuffer`, `CharBuffer`, `IntBuffer`, `LongBuffer`, `DoubleBuffer`, `FloatBuffer`, `ShortBuffer`. `ByteBuffer` is especially common for I/O.

---

### Q32. Explain position, limit, capacity, and mark.

**Answer:** A buffer has four important properties.

```text
0 -------- position -------- limit -------- capacity
                    ^
                   mark
```

- **capacity** → maximum number of elements
- **position** → current read/write location
- **limit** → first element that should not be read/written
- **mark** → saved position

```java
ByteBuffer buffer = ByteBuffer.allocate(10);

buffer.put((byte) 10);
buffer.put((byte) 20);

buffer.flip();
```

After `flip()`: `position = 0`, `limit = 2`, `capacity = 10`.

---

### Q33. Explain flip(), clear(), rewind(), and compact().

**Answer:**

| Method | Purpose |
|---|---|
| `flip()` | Switch write mode to read mode |
| `clear()` | Prepare the buffer for writing again |
| `rewind()` | Re-read existing data from beginning |
| `compact()` | Preserve unread data and make room for more writes |

```java
channel.read(buffer);

buffer.flip();

while (buffer.hasRemaining()) {
    process(buffer.get());
}

buffer.clear();
```

**Interview trap:** `clear()` does not erase the underlying data. It only changes buffer metadata.

---

### Q34. Why is flip() necessary before reading from a ByteBuffer?

**Answer:** When data is written into a buffer, `position` moves forward. `flip()` changes `limit = current position` and `position = 0`, telling the buffer "the data written so far is now the data available for reading." Without `flip()`, the buffer's position would remain at the end of the written data.

---

### Q35. What is compact() used for?

**Answer:** `compact()` is useful when the buffer contains partially consumed data that must be preserved.

```text
[unread data][free space]
```

After `compact()`: `[unread data][free space for new data]`. This is especially useful in non-blocking network protocols where a complete message may arrive over multiple reads.

---

### Q36. What is the difference between heap and direct ByteBuffer?

**Answer:**

```java
ByteBuffer heap = ByteBuffer.allocate(8192);
ByteBuffer direct = ByteBuffer.allocateDirect(8192);
```

Heap buffer: uses JVM heap memory, usually cheaper to allocate, easy for normal application processing. Direct buffer: uses memory outside the normal Java heap, can benefit certain native I/O operations, more expensive to allocate, requires careful lifecycle/memory consideration.

**Senior answer:** Direct buffers can reduce certain memory-copying overheads, but they are not automatically faster. I use them selectively and validate the benefit through profiling.

---

### Q37. What are slice() and duplicate() in ByteBuffer?

**Answer:** `slice()` creates a buffer sharing a region of the original buffer's content. `duplicate()` creates another buffer sharing the same content but with independent position/limit/mark state.

```text
Original Buffer
      |
      +---- slice() ----> shared content region
      |
      +---- duplicate() -> shared content, independent cursor
```

Because content can be shared, changes through one view can be visible through another.

---

# SECTION 6: NON-BLOCKING I/O & SELECTORS

### Q38. What is blocking vs non-blocking I/O?

**Answer:**

```text
Blocking I/O:
Thread → read() → waits

Non-blocking I/O:
Thread → Selector → ready Channel A / B / C
```

Non-blocking I/O allows a thread to manage multiple connections rather than waiting on one connection at a time.

---

### Q39. What is a Selector?

**Answer:** A `Selector` allows one or a small number of threads to monitor multiple selectable channels.

```java
Selector selector = Selector.open();

channel.configureBlocking(false);
channel.register(selector, SelectionKey.OP_READ);
```

The selector waits for channels that are ready for registered operations. Common operations: `OP_ACCEPT`, `OP_CONNECT`, `OP_READ`, `OP_WRITE`.

---

### Q40. What is SelectionKey?

**Answer:** A `SelectionKey` represents a channel's registration with a selector. It provides the channel, selector, interest operations, ready operations, and an attachment.

```java
SelectionKey key =
    channel.register(selector, SelectionKey.OP_READ);

key.attach(connectionContext);
```

---

### Q41. What is the difference between interestOps and readyOps?

**Answer:** **Interest operations** specify what the application wants to monitor.

```java
key.interestOps(SelectionKey.OP_READ |
                SelectionKey.OP_WRITE);
```

**Ready operations** indicate what the channel is currently ready to perform (`key.readyOps();`). Conceptually: interestOps = what I want to know about; readyOps = what is currently ready.

---

### Q42. Why is non-blocking I/O useful for high-concurrency network applications?

**Answer:** A traditional thread-per-connection design ties one thread to each connection. A selector-based model routes many connections through a single event loop. Benefits: fewer threads, lower thread-stack memory, less context switching, efficient handling of many mostly-idle connections.

**Senior caveat:** Non-blocking I/O does not mean all application work should execute on the event loop. CPU-heavy or blocking operations must be isolated.

---

### Q43. Can every Java Channel be registered with a Selector?

**Answer:** No. Selectors work with **selectable channels**, primarily network channels such as `SocketChannel`, `ServerSocketChannel`, `DatagramChannel`. A regular `FileChannel` is not used with a selector in the same way.

---

### Q44. What is the selector event-loop pattern?

**Answer:**

```java
while (running) {
    selector.select();

    for (SelectionKey key : selector.selectedKeys()) {

        if (key.isAcceptable()) {
            handleAccept(key);
        }

        if (key.isReadable()) {
            handleRead(key);
        }

        if (key.isWritable()) {
            handleWrite(key);
        }
    }

    selector.selectedKeys().clear();
}
```

The event loop should perform short, non-blocking operations.

---

# SECTION 7: PATH & FILES API

### Q45. What is the difference between File and Path?

**Answer:** `File` is the older filesystem abstraction from `java.io`. `Path` is the modern NIO.2 abstraction.

```java
File file = new File("data.txt");

Path path = Path.of("data.txt");
```

`Path` integrates naturally with `Files`, `FileSystem`, file attributes, symbolic links, and modern filesystem providers.

**Senior answer:** For new Java code I prefer Path and Files. I use File mainly when integrating with older APIs.

---

### Q46. What are the important Files API methods?

**Answer:** Common operations:

```java
Files.exists(path);
Files.isRegularFile(path);
Files.isDirectory(path);

Files.createFile(path);
Files.createDirectory(path);
Files.createDirectories(path);

Files.copy(source, target);
Files.move(source, target);
Files.deleteIfExists(path);

Files.readString(path);
Files.writeString(path, content);

Files.readAllBytes(path);
Files.readAllLines(path);
```

The right API depends on file size and processing requirements.

---

### Q47. createDirectory() vs createDirectories()?

**Answer:** `createDirectory()` creates one directory and expects its parent to exist.

```java
Files.createDirectory(Path.of("a/b"));
```

`createDirectories()` creates missing parent directories as necessary.

```java
Files.createDirectories(Path.of("a/b/c"));
```

For application startup/configuration directories, `createDirectories()` is often more convenient.

---

### Q48. How do you copy, move, and delete files?

**Answer:**

```java
Files.copy(source, target,
           StandardCopyOption.REPLACE_EXISTING);

Files.move(source, target,
           StandardCopyOption.REPLACE_EXISTING);

Files.deleteIfExists(path);
```

For supported filesystems:

```java
Files.move(source, target,
           StandardCopyOption.ATOMIC_MOVE);
```

**Senior caveat:** Atomic-move support is filesystem/provider dependent.

---

### Q49. What is Files.walk()?

**Answer:** `Files.walk()` recursively traverses a directory tree and returns a lazy `Stream<Path>`.

```java
try (Stream<Path> paths = Files.walk(root)) {
    paths.filter(Files::isRegularFile)
         .forEach(System.out::println);
}
```

Because the stream may hold filesystem resources, it should be closed.

---

### Q50. Files.walk() vs Files.walkFileTree()?

**Answer:**

| `Files.walk()` | `Files.walkFileTree()` |
|---|---|
| Stream-based | Visitor-based |
| Concise | More control |
| Functional processing | Detailed traversal callbacks |
| Good for filtering | Good for complex traversal/error handling |

`walkFileTree()` supports callbacks such as `preVisitDirectory()`, `visitFile()`, `visitFileFailed()`, `postVisitDirectory()`.

---

### Q51. How do symbolic links affect file operations?

**Answer:** A symbolic link points to another filesystem location.

```java
Files.isSymbolicLink(path);
```

Security-sensitive code should carefully decide whether links should be followed. Simply validating a path string may not be enough if a symbolic link can redirect access outside an allowed directory.

**Senior point:** For security-sensitive file access, I validate the resolved/real path and define explicit symbolic-link behavior rather than trusting the original path alone.

---

### Q52. How do you read file metadata?

**Answer:** NIO provides file attribute APIs.

```java
BasicFileAttributes attrs =
    Files.readAttributes(path, BasicFileAttributes.class);

System.out.println(attrs.size());
System.out.println(attrs.creationTime());
System.out.println(attrs.lastModifiedTime());
```

This avoids having to infer metadata from file contents.

---

# SECTION 8: ADVANCED FILE OPERATIONS

### Q53. What is a memory-mapped file?

**Answer:** A memory-mapped file maps a file region into memory using `FileChannel.map()`.

```java
try (FileChannel channel = FileChannel.open(path)) {

    MappedByteBuffer buffer =
        channel.map(
            FileChannel.MapMode.READ_ONLY,
            0,
            channel.size());
}
```

Potential benefits: efficient random access, useful for large files, can reduce explicit copying for suitable workloads.

**Senior caveat:** Memory mapping is workload-dependent. I would benchmark it against buffered/channel-based I/O rather than assuming it is always faster.

---

### Q54. What is FileLock?

**Answer:** `FileLock` provides file-region locking.

```java
try (FileChannel channel =
         FileChannel.open(path,
             StandardOpenOption.CREATE,
             StandardOpenOption.WRITE);
     FileLock lock = channel.lock()) {

    // protected operation
}
```

Locks can help coordinate access to files, but behavior depends on the underlying operating system/filesystem. Important: a `FileLock` is not a distributed lock for coordinating arbitrary application instances across a cluster.

---

### Q55. What is the difference between absolute and relative Path?

**Answer:** Absolute path: `/opt/app/data/file.txt`. Relative path: `data/file.txt`. Useful operations: `path.toAbsolutePath()`, `path.normalize()`, `path.resolve("child.txt")`, `path.relativize(other)`.

---

### Q56. What is normalize() vs toRealPath()?

**Answer:** `normalize()` removes redundant path components such as `.` and `..` syntactically.

```java
Path normalized = path.normalize();
```

`toRealPath()` resolves the path against the filesystem and can resolve symbolic links.

```java
Path real = path.toRealPath();
```

**Security point:** For path-validation security, understanding the difference is important. A normalized string is not necessarily the same as the actual filesystem location.

---

# SECTION 9: SENIOR-LEVEL SCENARIO QUESTIONS

### Q57. How would you process a 20 GB file without running out of memory?

**Answer:** I would use bounded, streaming processing: never load the entire file, use `BufferedReader`, `Files.lines()`, or `FileChannel` depending on the format, process records/chunks incrementally, use bounded queues if parallel processing is needed, apply backpressure, avoid unbounded collections, track progress and failures, and make processing restartable/idempotent where possible.

**Senior answer:** The main design principle is bounded memory. For a 20 GB file, `readAllBytes()` and `readAllLines()` are immediately suspicious.

---

### Q58. How would you efficiently copy a 50 GB file?

**Answer:** For large files, I would consider `FileChannel.transferTo()`.

```java
try (FileChannel source = FileChannel.open(sourcePath);
     FileChannel target = FileChannel.open(
         targetPath,
         StandardOpenOption.CREATE,
         StandardOpenOption.WRITE,
         StandardOpenOption.TRUNCATE_EXISTING)) {

    long position = 0;
    long size = source.size();

    while (position < size) {
        long transferred =
            source.transferTo(position, size - position, target);

        if (transferred <= 0) {
            break;
        }

        position += transferred;
    }
}
```

**Senior point:** Do not assume one `transferTo()` call transfers the entire requested range. Handle partial transfers.

---

### Q59. How do you prevent file descriptor leaks in a Spring Boot application?

**Answer:** Use try-with-resources; close streams/readers/writers/channels; close `Files.lines()` and directory streams; avoid retaining resources in long-lived objects; monitor OS-level file descriptors; review exception paths; use static analysis where available; check application/container limits.

A leak can eventually produce: `Too many open files`.

---

### Q60. How would you handle blocking I/O in a reactive/event-driven application?

**Answer:** Blocking I/O should not run on the event-loop thread. A safe architecture routes non-blocking work through the event loop and isolates blocking I/O to a bounded worker pool, which should itself be bounded and monitored.

**Senior answer:** The main concern is event-loop starvation. A single slow blocking operation can delay unrelated requests, so blocking work must be isolated.

---

### Q61. What are common Java I/O performance problems?

**Answer:** Reading one byte at a time, missing buffering, loading huge files into memory, calling `flush()` excessively, creating too many buffers, too many threads for blocking operations, blocking event-loop threads, resource leaks, wrong/default charset, ignoring filesystem and OS limits.

**Senior answer:** I measure before optimizing. I look at throughput, latency, CPU, allocation rate, memory, system calls, file descriptors, and GC rather than assuming a particular API is faster.

---

### Q62. How would you design a high-throughput file processing service?

**Answer:** A practical architecture: File Source → Reader/Channel → Bounded Queue → Worker Pool → Validation → Business Processing → Output/DB/Message. Important considerations: bounded queues, backpressure, batch processing, idempotency, retry strategy, dead-letter/error handling, metrics, checkpointing, graceful shutdown, resource cleanup.

**Senior answer:** I would optimize the complete pipeline rather than only the file read. Usually the bottleneck can be downstream processing, database writes, serialization, or contention.

---

# SECTION 10: COMMON INTERVIEW TRAPS

### Q63. Does clear() erase a ByteBuffer?

**Answer:** No. `clear()` only resets buffer state: `position = 0`, `limit = capacity`. The underlying bytes may still physically exist.

---

### Q64. Does flush() guarantee that data is physically persisted to disk?

**Answer:** Not necessarily. `flush()` generally pushes buffered application data to the underlying stream. For stronger durability semantics, filesystem/channel-specific mechanisms such as `FileChannel.force()` may be relevant.

```java
channel.force(true);
```

**Senior point:** Flushing application buffers and guaranteeing durable storage are different concerns.

---

### Q65. Does Files.exists() guarantee that a subsequent operation will succeed?

**Answer:** No. This is a classic time-of-check/time-of-use issue.

```java
if (Files.exists(path)) {
    Files.delete(path);
}
```

The filesystem state can change between the check and operation. **Better approach:** perform the operation and handle the resulting exception where appropriate.

---

### Q66. Is Java serialization the same as JSON serialization?

**Answer:** No. Java serialization is a Java-specific binary object serialization mechanism. JSON serialization represents data in a language-independent text format. For microservices and external APIs, JSON or schema-based formats such as Protobuf are generally more appropriate.

---

### Q67. Is NIO always faster than java.io?

**Answer:** No. NIO provides different capabilities and abstractions. For simple sequential I/O, traditional buffered streams can perform very well. NIO becomes particularly valuable when you need channels, buffers, selectors, advanced filesystem operations, random access, file transfer, or non-blocking network I/O.

**Senior answer:** I choose based on workload, not API age. Performance should be demonstrated through measurement.

---

# SECTION 11: QUICK-FIRE INTERVIEW QUESTIONS

- What does `read()` return at EOF? → `-1`.
- Why does `read()` return `int`? → To represent byte values plus `-1`.
- Byte stream vs character stream? → Bytes vs characters.
- Why use buffering? → Reduce underlying I/O operations.
- Does `flush()` close the stream? → No.
- Does `close()` normally flush output? → Yes.
- What does `try-with-resources` solve? → Automatic resource cleanup.
- What is a suppressed exception? → Exception thrown during resource cleanup while another exception is already primary.
- What does `transient` do? → Excludes a field from default serialization.
- Are static fields serialized? → No.
- What is `serialVersionUID`? → Serialization compatibility identifier.
- What is `Externalizable`? → Explicit serialization control.
- What is `FileChannel`? → Channel-based file I/O API.
- What is a `ByteBuffer`? → NIO data container used with channels.
- Buffer properties? → Position, limit, capacity, mark.
- What does `flip()` do? → Prepares written data for reading.
- What does `clear()` do? → Resets buffer for writing.
- Does `clear()` erase data? → No.
- What does `compact()` do? → Preserves unread data and prepares remaining space for writing.
- Heap vs direct buffer? → Heap uses JVM heap; direct uses native memory.
- What is a Selector? → Multiplexes selectable channels.
- What is SelectionKey? → Registration representing a channel's selector relationship.
- InterestOps vs readyOps? → Desired events vs currently ready events.
- Can FileChannel be registered with Selector? → Not as a selectable channel.
- `File` vs `Path`? → Prefer `Path` for modern code.
- `readAllLines()` vs `lines()`? → Eager collection vs lazy stream.
- Does `Files.lines()` need closing? → Yes.
- `walk()` vs `walkFileTree()`? → Stream-based vs visitor-based traversal.
- Is `FileLock` a distributed lock? → No.
- Does `flush()` guarantee disk durability? → Not necessarily.
- Is `transient` encryption? → No.
- Is NIO always faster? → No.
- Is Java native serialization safe for arbitrary untrusted input? → No.

---

# TOP 20 QUESTIONS TO PRIORITIZE

1. **Byte stream vs Character stream**
2. **InputStream vs Reader**
3. **Why does read() return int?**
4. **BufferedReader / BufferedInputStream and buffering**
5. **flush() vs close()**
6. **try-with-resources and suppressed exceptions**
7. **Serializable and serialVersionUID**
8. **transient keyword**
9. **Serializable vs Externalizable**
10. **Java deserialization security**
11. **Channels vs Streams**
12. **ByteBuffer position, limit, capacity, mark**
13. **flip() vs clear() vs rewind() vs compact()**
14. **Heap vs Direct ByteBuffer**
15. **Selector and SelectionKey**
16. **Blocking vs non-blocking I/O**
17. **File vs Path**
18. **Files.readAllLines() vs Files.lines()**
19. **Files.walk() vs walkFileTree()**
20. **How to process/copy very large files efficiently**

---

# SENIOR DEVELOPER INTERVIEW CLOSING ANSWER

### If the interviewer asks: "How do you decide which Java I/O API to use?"

**Answer:** "I start with the data and access pattern. For binary data or simple sequential operations, I use InputStream and OutputStream, normally with buffering. For text, I use Reader and Writer with an explicit charset such as UTF-8. For modern filesystem operations, I prefer Path and Files because they provide a richer API. When I need random access, file locking, memory mapping, or efficient channel-based transfers, I consider FileChannel and ByteBuffer. For high-concurrency network workloads, non-blocking channels and selectors can multiplex many connections efficiently. In production, I also focus on bounded memory, try-with-resources, resource ownership, correct charset handling, avoiding blocking operations on event-loop threads, and measuring performance rather than assuming NIO or any particular API is automatically faster."

---

# FINAL INTERVIEW CHECKLIST

Before an experience Java interview, make sure you can explain these without memorizing definitions:

### I/O
- [ ] Byte vs character streams
- [ ] InputStream vs Reader
- [ ] Buffering
- [ ] flush vs close
- [ ] try-with-resources
- [ ] Suppressed exceptions
- [ ] Charset/UTF-8

### Serialization
- [ ] Serializable
- [ ] serialVersionUID
- [ ] transient
- [ ] static fields
- [ ] Constructor behavior
- [ ] readObject/writeObject
- [ ] readResolve/writeReplace
- [ ] Externalizable
- [ ] Deserialization security

### NIO
- [ ] Channels
- [ ] FileChannel
- [ ] ByteBuffer
- [ ] position/limit/capacity/mark
- [ ] flip/clear/rewind/compact
- [ ] Heap vs direct buffer
- [ ] slice/duplicate
- [ ] transferTo/transferFrom
- [ ] Selector
- [ ] SelectionKey
- [ ] Blocking vs non-blocking

### Files & Path
- [ ] File vs Path
- [ ] Files API
- [ ] readAllLines vs lines
- [ ] walk vs walkFileTree
- [ ] File attributes
- [ ] Symbolic links
- [ ] normalize vs toRealPath
- [ ] Atomic move
- [ ] FileLock

### Production
- [ ] Process 20 GB+ files
- [ ] Avoid memory exhaustion
- [ ] Prevent file descriptor leaks
- [ ] Efficient large-file copy
- [ ] Blocking I/O in reactive systems
- [ ] Backpressure
- [ ] Performance measurement

**Target for senior interviews:** Do not only answer *"what is it?"*. Be prepared for *"why would you use it?", "what are the trade-offs?", "what can go wrong?", and "how would you implement it in production?"*