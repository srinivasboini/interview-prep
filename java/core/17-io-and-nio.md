# Java I/O and NIO: Comprehensive Guide

## Overview

Java provides two distinct I/O paradigms: traditional blocking I/O (java.io) and non-blocking NIO (java.nio). While classic I/O uses thread-per-connection model, NIO enables highly scalable applications through selectors and non-blocking operations. Understanding both is critical for senior engineers—legacy systems often rely on classic I/O, while modern high-performance systems leverage NIO. Java 11+ adds convenience methods that make common operations cleaner.

**Why this matters in interviews**: Interviewers assess understanding of I/O bottlenecks, scalability trade-offs, and practical decisions about which approach to use. Banking systems handle millions of transactions requiring solid I/O design.

---

## Part 1: Traditional Java I/O (java.io)

### Key Concepts

#### Stream Hierarchy
Java I/O uses a byte stream and character stream model:

```
InputStream (abstract)
├── ByteArrayInputStream
├── FileInputStream
├── FilterInputStream
│   └── BufferedInputStream
├── PipedInputStream
└── SequenceInputStream

OutputStream (abstract)
├── ByteArrayOutputStream
├── FileOutputStream
├── FilterOutputStream
│   └── BufferedOutputStream
├── PipedOutputStream
└── PrintStream
```

Readers and Writers are character-oriented (UTF-16 internally, can wrap byte streams):
```
Reader/Writer (character streams)
├── FileReader / FileWriter
├── InputStreamReader / OutputStreamWriter (byte-to-char bridge)
├── BufferedReader / BufferedWriter
└── StringReader / StringWriter
```

#### Stream Characteristics
- **Blocking**: Reading/writing threads block until data available or written
- **Sequential**: Must process data in order
- **Unbuffered by default**: Consider BufferedInputStream/BufferedReader for performance
- **1-way**: Either read OR write, not both on same stream

### Practical Implementation

```java
// Traditional blocking read - thread blocks until data available
public class FileReadExample {
    /**
     * Simple file reading - demonstrates blocking I/O.
     * Thread blocks at read() until 1 byte available or EOF.
     * Inefficient for large files or high-concurrency scenarios.
     */
    public static void readFileBlocking(String filePath) throws IOException {
        try (FileInputStream fis = new FileInputStream(filePath)) {
            int byteRead;
            while ((byteRead = fis.read()) != -1) {
                // Process single byte - slow!
                processByte((byte) byteRead);
            }
        }
    }

    /**
     * Buffered reading - reads chunks into buffer.
     * Much more efficient: single thread waits for buffer fill, not per-byte.
     * Standard approach for file I/O in production.
     */
    public static void readFileBuffered(String filePath) throws IOException {
        try (BufferedInputStream bis = new BufferedInputStream(
                new FileInputStream(filePath), 8192)) {
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = bis.read(buffer)) != -1) {
                processBuffer(buffer, bytesRead);
            }
        }
    }

    /**
     * Character-level reading with encoding support.
     * InputStreamReader converts bytes to chars based on charset.
     * BufferedReader adds line-reading convenience.
     */
    public static void readFileAsText(String filePath) throws IOException {
        try (BufferedReader br = new BufferedReader(
                new InputStreamReader(
                        new FileInputStream(filePath), StandardCharsets.UTF_8))) {
            String line;
            while ((line = br.readLine()) != null) {
                processLine(line);
            }
        }
    }

    /**
     * Writing with buffering and auto-flush.
     * BufferedWriter queues data, flushes periodically or explicitly.
     * PrintWriter adds println/printf convenience methods.
     */
    public static void writeFileBuffered(String filePath, List<String> lines)
            throws IOException {
        try (BufferedWriter bw = new BufferedWriter(
                new OutputStreamWriter(
                        new FileOutputStream(filePath), StandardCharsets.UTF_8))) {
            for (String line : lines) {
                bw.write(line);
                bw.newLine();  // Platform-specific line separator
            }
            bw.flush();  // Ensure data written to disk
        }
    }
}
```

### Serialization

```java
/**
 * Java Serialization - converts objects to byte stream.
 * Critical for distributed systems, but has security/performance trade-offs.
 */
public class SerializationExample {
    // Must implement Serializable marker interface
    static class Contract implements Serializable {
        private static final long serialVersionUID = 1L;
        private String id;
        private BigDecimal amount;
        private transient LocalDateTime createdAt;  // NOT serialized

        // ... getters/setters
    }

    public static void serializeObject(Contract contract, String filePath)
            throws IOException {
        try (ObjectOutputStream oos = new ObjectOutputStream(
                new FileOutputStream(filePath))) {
            oos.writeObject(contract);
        }
    }

    public static Contract deserializeObject(String filePath)
            throws IOException, ClassNotFoundException {
        try (ObjectInputStream ois = new ObjectInputStream(
                new FileInputStream(filePath))) {
            return (Contract) ois.readObject();
        }
    }
}
```

**Serialization Considerations**:
- `serialVersionUID`: Enables versioning; omitting allows incompatible changes to break deserialization
- `transient` fields: Excluded from serialization (e.g., temporary state, timestamps)
- Security risk: Untrusted serialized data can execute arbitrary code
- Modern alternative: JSON (via Jackson/Gson) preferred for inter-system communication

### Diagram 1: Traditional I/O Stream Chain

```
┌─────────────────────────────────────────────────────────────┐
│ Application Code                                            │
│ (high-level operations: read line, read object)            │
└──────────────────┬──────────────────────────────────────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
   BufferedReader         ObjectInputStream
   (buffers chars)        (reconstructs objects)
        │                     │
   InputStreamReader      ObjectInputStream internals
   (bytes→chars)          (bytes→object graph)
        │                     │
   BufferedInputStream    BufferedInputStream
   (buffers bytes)        (buffers bytes)
        │                     │
   FileInputStream        FileInputStream
   (OS file handle)       (OS file handle)
        │                     │
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────┐
        │  Kernel Buffer      │
        │  (page cache)       │
        └──────────┬──────────┘
                   │
        ┌──────────▼──────────┐
        │  Disk Storage       │
        └─────────────────────┘
```

---

## Part 2: NIO (java.nio) - Non-Blocking I/O

### Core Concepts

#### The Problem NIO Solves
Traditional I/O with thread-per-connection scales poorly:
- 1,000 concurrent connections = 1,000+ threads
- Thread overhead: ~1MB stack memory per thread
- Context switching costs at OS level
- Blocking on read/write wastes resources

NIO enables **thousands of connections per thread** through multiplexing.

#### Core Components

**1. Buffers**: Memory containers for data transfer
```
Buffer (abstract)
├── ByteBuffer (most common)
│   ├── Direct: Allocated outside JVM heap (DMA-friendly)
│   └── Non-direct: On-heap (GC managed)
├── CharBuffer
├── IntBuffer
└── ... (other primitive types)
```

**2. Channels**: Bidirectional, non-blocking I/O endpoints
```
Channel (interface)
├── FileChannel (file I/O, seekable)
├── SocketChannel (TCP networking, non-blocking)
├── ServerSocketChannel (accepts TCP connections)
├── DatagramChannel (UDP)
└── Pipe.SinkChannel / Pipe.SourceChannel
```

**3. Selectors**: Multiplexer allowing single thread to monitor multiple channels
```
Selector
├── Register channels with interest set (READ, WRITE, ACCEPT, CONNECT)
├── select() blocks until events occur
├── Returns ready channels for processing
└── Enables high-concurrency with few threads
```

### Practical NIO Implementation

```java
/**
 * NIO FileChannel - modern approach to file I/O.
 * More efficient than traditional streams for certain operations.
 */
public class NIOFileChannelExample {

    /**
     * Reading file via FileChannel.
     * Direct buffer allocation for better OS interaction.
     */
    public static void readFileWithChannel(String filePath) throws IOException {
        try (RandomAccessFile file = new RandomAccessFile(filePath, "r");
             FileChannel channel = file.getChannel()) {

            // Allocate direct buffer for optimal performance
            ByteBuffer buffer = ByteBuffer.allocateDirect(16 * 1024);  // 16KB

            while (channel.read(buffer) > 0) {
                buffer.flip();  // Switch from write-mode to read-mode
                while (buffer.hasRemaining()) {
                    byte b = buffer.get();
                    processByte(b);
                }
                buffer.clear();  // Prepare for next read
            }
        }
    }

    /**
     * Zero-copy file transfer between channels.
     * Kernel directly transfers data: source → destination.
     * Optimal for file copying, proxying scenarios.
     */
    public static void copyFileWithZeroCopy(String source, String dest)
            throws IOException {
        try (FileChannel sourceChannel = new RandomAccessFile(source, "r")
                    .getChannel();
             FileChannel destChannel = new RandomAccessFile(dest, "rw")
                    .getChannel()) {

            sourceChannel.transferTo(0, sourceChannel.size(), destChannel);
            // No application-level buffer: kernel copies directly
        }
    }

    /**
     * File memory mapping - maps file into virtual memory.
     * Enables random access with OS-level efficiency.
     */
    public static void readFileMappedMemory(String filePath) throws IOException {
        try (RandomAccessFile file = new RandomAccessFile(filePath, "r");
             FileChannel channel = file.getChannel()) {

            MappedByteBuffer buffer = channel.map(
                    FileChannel.MapMode.READ_ONLY, 0, channel.size());

            while (buffer.hasRemaining()) {
                byte b = buffer.get();
                processByte(b);
            }
        }
    }
}
```

### NIO Selector-Based Server

```java
/**
 * Non-blocking TCP server using Selector.
 * Handles thousands of concurrent connections with few threads.
 */
public class NonBlockingServer {
    private Selector selector;
    private ServerSocketChannel serverChannel;

    public void start(int port) throws IOException {
        selector = Selector.open();
        serverChannel = ServerSocketChannel.open();
        serverChannel.configureBlocking(false);
        serverChannel.bind(new InetSocketAddress(port));

        // Register for ACCEPT events (new connection attempts)
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);

        System.out.println("Server listening on port " + port);
        eventLoop();
    }

    /**
     * Main event loop - single thread monitors all registered channels.
     * select() blocks until one or more channels ready, then returns immediately.
     */
    private void eventLoop() throws IOException {
        while (true) {
            // Block until channels ready or timeout
            selector.select();  // Could also use select(timeout)

            Set<SelectionKey> readyKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = readyKeys.iterator();

            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove();  // Must remove from set

                try {
                    if (key.isAcceptable()) {
                        handleAccept(key);
                    } else if (key.isReadable()) {
                        handleRead(key);
                    } else if (key.isWritable()) {
                        handleWrite(key);
                    }
                } catch (IOException e) {
                    key.cancel();
                    key.channel().close();
                }
            }
        }
    }

    private void handleAccept(SelectionKey key) throws IOException {
        ServerSocketChannel server = (ServerSocketChannel) key.channel();
        SocketChannel client = server.accept();

        if (client != null) {
            client.configureBlocking(false);
            // Register for READ events
            client.register(selector, SelectionKey.OP_READ);
            System.out.println("Client connected: " + client.getRemoteAddress());
        }
    }

    private void handleRead(SelectionKey key) throws IOException {
        SocketChannel client = (SocketChannel) key.channel();
        ByteBuffer buffer = ByteBuffer.allocate(1024);

        int bytesRead = client.read(buffer);
        if (bytesRead == -1) {
            // Client disconnected
            client.close();
            key.cancel();
        } else if (bytesRead > 0) {
            buffer.flip();
            // Echo back
            client.write(buffer);
            buffer.clear();
        }
    }

    private void handleWrite(SelectionKey key) throws IOException {
        // Implement write logic if needed
    }
}
```

### Buffer Operations in Detail

```java
/**
 * ByteBuffer is the workhorse of NIO.
 * Understanding buffer states and operations is crucial.
 */
public class BufferOperations {

    public static void bufferStates() {
        ByteBuffer buffer = ByteBuffer.allocate(10);

        // WRITE mode: position=0, limit=capacity(10), capacity=10
        buffer.put("Hello".getBytes(StandardCharsets.UTF_8));
        // After put: position=5

        // Transition to READ mode
        buffer.flip();  // position=0, limit=5, capacity=10

        // READ operations
        while (buffer.hasRemaining()) {
            byte b = buffer.get();  // position increments
        }
        // After reading: position=5

        // Reuse buffer
        buffer.clear();  // position=0, limit=capacity(10)

        // OR rewind to re-read
        buffer.flip();   // position=0, limit=5
        byte[] data = new byte[5];
        buffer.get(data);  // bulk get
    }

    /**
     * Direct vs non-direct buffers.
     * Direct: Off-heap memory, better for I/O but harder for GC.
     * Non-direct: On-heap, easier GC but potentially slower I/O.
     */
    public static void bufferAllocation() {
        // Non-direct: on-heap, managed by GC
        ByteBuffer nonDirect = ByteBuffer.allocate(1024);

        // Direct: off-heap, survives GC, better for long-lived I/O
        ByteBuffer direct = ByteBuffer.allocateDirect(1024);

        System.out.println("Direct: " + direct.isDirect());      // true
        System.out.println("NonDirect: " + nonDirect.isDirect()); // false
    }

    /**
     * Scatter-gather (vectored I/O): Read/write multiple buffers atomically.
     * Useful for reading structured data (headers + payload).
     */
    public static void scatterGatherIO(FileChannel channel) throws IOException {
        // Gather: write multiple buffers to channel in order
        ByteBuffer header = ByteBuffer.wrap("HEADER".getBytes());
        ByteBuffer payload = ByteBuffer.wrap("PAYLOAD".getBytes());
        ByteBuffer[] buffers = {header, payload};

        channel.write(buffers);  // Writes all sequentially

        // Scatter: read from channel into multiple buffers
        ByteBuffer[] readBuffers = new ByteBuffer[2];
        readBuffers[0] = ByteBuffer.allocate(6);   // Header size
        readBuffers[1] = ByteBuffer.allocate(100); // Payload size

        channel.read(readBuffers);  // Fills buffers sequentially
    }
}
```

### Diagram 2: NIO Selector Architecture

```
Thread 1 (Event Loop)
│
├─ Selector.select()
│  (blocks until events)
│
├─ selectedKeys = selector.selectedKeys()
│
├─ for each SelectionKey:
│  │
│  ├─ if (key.isAcceptable())
│  │  └─ Accept new connection
│  │
│  ├─ if (key.isReadable())
│  │  └─ Read from SocketChannel into ByteBuffer
│  │     └─ Process application logic
│  │
│  └─ if (key.isWritable())
│     └─ Write response from ByteBuffer to SocketChannel
│
│
Registered Channels (thousands possible):
├─ ServerSocketChannel (listening for ACCEPT)
├─ SocketChannel 1 (registered for READ|WRITE)
├─ SocketChannel 2 (registered for READ|WRITE)
├─ SocketChannel 3 (registered for READ)
└─ ... SocketChannel N
```

---

## Part 3: Java 7+ Modern API (Path, Files, WatchService)

### Path and Files

Java 7 introduced `java.nio.file` package, providing higher-level abstractions:

```java
/**
 * Path: Represents file system path (replaces File).
 * Files: Utility methods for common operations.
 */
public class ModernFileOperations {

    public static void pathOperations() {
        Path path = Paths.get("/data/contracts", "2024", "Q4.csv");

        // Useful operations
        Path fileName = path.getFileName();              // Q4.csv
        Path parent = path.getParent();                  // /data/contracts/2024
        Path root = path.getRoot();                      // /
        Path resolved = Paths.get("/data").resolve("contracts");
        Path relative = Paths.get("/data").relativize(path);

        System.out.println("Normalized: " + path.normalize());
        System.out.println("Absolute: " + path.toAbsolutePath());
    }

    /**
     * Files utility class - streamlined operations.
     * Handles resource cleanup automatically.
     */
    public static void filesOperations(String filePath) throws IOException {
        Path path = Paths.get(filePath);

        // Check existence
        Files.exists(path);
        Files.notExists(path);
        Files.isDirectory(path);
        Files.isRegularFile(path);

        // Create/Delete
        Files.createFile(path);
        Files.createDirectory(Paths.get("/new/dir"));
        Files.createDirectories(Paths.get("/new/nested/dir"));  // Creates parents
        Files.deleteIfExists(path);

        // Copy
        Files.copy(path, Paths.get("/backup/file.txt"),
                StandardCopyOption.REPLACE_EXISTING);

        // Move/Rename
        Files.move(path, Paths.get("/archive/file.txt"),
                StandardCopyOption.ATOMIC_MOVE);

        // Attributes
        BasicFileAttributes attrs = Files.readAttributes(path,
                BasicFileAttributes.class);
        System.out.println("Size: " + attrs.size());
        System.out.println("Created: " + attrs.creationTime());
        System.out.println("Modified: " + attrs.lastModifiedTime());
    }
}
```

### File Tree Walking

```java
/**
 * Walking file trees - crucial for batch processing, backups, indexing.
 */
public class FileTreeWalking {

    /**
     * Simple traversal with FileVisitor.
     * Custom visitor implements pre/post-processing logic.
     */
    public static void walkWithVisitor(Path startPath) throws IOException {
        Files.walkFileTree(startPath, new SimpleFileVisitor<Path>() {

            @Override
            public FileVisitResult preVisitDirectory(Path dir,
                    BasicFileAttributes attrs) {
                System.out.println("Entering: " + dir);
                return FileVisitResult.CONTINUE;
            }

            @Override
            public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) {
                // Process file
                if (file.toString().endsWith(".csv")) {
                    System.out.println("Found CSV: " + file);
                }
                return FileVisitResult.CONTINUE;
            }

            @Override
            public FileVisitResult postVisitDirectory(Path dir, IOException exc) {
                System.out.println("Exiting: " + dir);
                return FileVisitResult.CONTINUE;
            }

            @Override
            public FileVisitResult visitFileFailed(Path file, IOException exc) {
                // Handle permission errors, etc.
                System.err.println("Error accessing: " + file);
                return FileVisitResult.CONTINUE;  // Keep walking
            }
        });
    }

    /**
     * Stream-based walking - functional approach.
     * More concise for filtering/transformation.
     */
    public static void walkWithStream(Path startPath) throws IOException {
        try (Stream<Path> paths = Files.walk(startPath, FileVisitOption.FOLLOW_LINKS)) {
            paths.filter(Files::isRegularFile)
                    .filter(p -> p.toString().endsWith(".csv"))
                    .forEach(System.out::println);
        }
    }

    /**
     * Find files matching predicate.
     * Efficient: stops early when limit reached.
     */
    public static void findFiles(Path startPath) throws IOException {
        try (Stream<Path> paths = Files.find(startPath, 5,  // max depth
                (path, attrs) -> attrs.isRegularFile() &&
                        path.toString().contains("contract"))) {
            paths.forEach(System.out::println);
        }
    }
}
```

### WatchService

```java
/**
 * WatchService: Monitor file system for changes (creation, modification, deletion).
 * Critical for hot-reload configuration, log file monitoring, etc.
 */
public class FileSystemWatching {

    public static void watchDirectory(String dirPath) throws IOException, InterruptedException {
        Path path = Paths.get(dirPath);
        WatchService watchService = FileSystems.getDefault().newWatchService();

        // Register for events
        path.register(watchService,
                StandardWatchEventKinds.ENTRY_CREATE,
                StandardWatchEventKinds.ENTRY_MODIFY,
                StandardWatchEventKinds.ENTRY_DELETE);

        System.out.println("Watching: " + path);

        // Event processing loop
        while (true) {
            WatchKey key = watchService.take();  // Blocks until event

            for (WatchEvent<?> event : key.pollEvents()) {
                WatchEvent.Kind<?> kind = event.kind();
                Path filename = (Path) event.context();

                if (kind == StandardWatchEventKinds.ENTRY_CREATE) {
                    System.out.println("Created: " + filename);
                } else if (kind == StandardWatchEventKinds.ENTRY_MODIFY) {
                    System.out.println("Modified: " + filename);
                } else if (kind == StandardWatchEventKinds.ENTRY_DELETE) {
                    System.out.println("Deleted: " + filename);
                }
            }

            // Reset key for next events
            boolean valid = key.reset();
            if (!valid) {
                System.out.println("Watch key invalid, stopping.");
                break;
            }
        }
    }
}
```

---

## Part 4: Java 11+ Convenience Methods

Modern Java adds syntactic sugar for common operations:

```java
/**
 * Java 11+ convenience methods.
 * Eliminates boilerplate for common file operations.
 */
public class ModernConvenience {

    /**
     * Files.readString - read entire file as String in one call.
     * Before Java 11: Required BufferedReader + StringBuilder loop.
     */
    public static void readStringModern(String filePath) throws IOException {
        Path path = Paths.get(filePath);

        // Single call, handles charset (UTF-8 default)
        String content = Files.readString(path);

        // With explicit charset
        String contentLatin1 = Files.readString(path, StandardCharsets.ISO_8859_1);

        processContent(content);
    }

    /**
     * Files.writeString - write String to file in one call.
     * Options: CREATE, APPEND, TRUNCATE_EXISTING, etc.
     */
    public static void writeStringModern(String filePath, String content)
            throws IOException {
        Path path = Paths.get(filePath);

        Files.writeString(path, content);  // CREATE + TRUNCATE

        // Append
        Files.writeString(path, "\nAppended content\n",
                StandardOpenOption.APPEND);

        // With charset
        Files.writeString(path, content, StandardCharsets.UTF_16);
    }

    /**
     * Files.readAllLines - read file as List<String>.
     * More memory-efficient than readString for large files.
     */
    public static void readAllLinesModern(String filePath) throws IOException {
        Path path = Paths.get(filePath);

        List<String> lines = Files.readAllLines(path);

        // With explicit charset
        List<String> linesUTF16 = Files.readAllLines(path,
                StandardCharsets.UTF_16);

        // Stream-based approach (lazy, memory-efficient)
        try (Stream<String> lines2 = Files.lines(path)) {
            lines2.filter(line -> !line.isEmpty())
                    .forEach(System.out::println);
        }
    }

    /**
     * Files.mismatch (Java 12+) - compare two files byte-by-byte.
     * Returns position of first difference, or -1 if identical.
     * Efficient: stops at first mismatch.
     */
    public static void compareFiles(String file1, String file2) throws IOException {
        Path path1 = Paths.get(file1);
        Path path2 = Paths.get(file2);

        long mismatchPos = Files.mismatch(path1, path2);

        if (mismatchPos == -1) {
            System.out.println("Files are identical");
        } else {
            System.out.println("First difference at byte: " + mismatchPos);
        }
    }
}
```

### Diagram 3: I/O API Evolution

```
┌─────────────────────────────────────────────────────────┐
│ Java I/O API Timeline                                   │
└─────────────────────────────────────────────────────────┘

Java 1.0-1.3  Java 1.4     Java 1.7         Java 11+
  |             |            |                 |
  |             |            |                 |
  | java.io     | java.nio   | java.nio.file  | Convenience
  |             |            |                |
  |             |            |                |
Stream      Channel      Path/Files      readString/
Reader/     Buffer       WatchService    writeString
Writer      Selector     Better APIs     mismatch

Legacy      High-perf    Cross-platform  Developer
systems     networking   file handling   productivity

Blocking    Non-blocking Modern NIO      Syntactic
threads     selectors    abstractions    sugar
```

---

## Interview Questions & Answers

### Q1: When would you use NIO vs Traditional I/O?

**Answer**:
- **Traditional I/O**: Fewer connections, simpler logic, readability matters. Files, system processes, logging (blocking fine here).
  - Example: Reading config files, writing logs, simple file processing.

- **NIO**: Many concurrent connections, low latency, throughput critical.
  - Example: Web servers (thousands of clients), messaging brokers, proxies, banking APIs.

**Banking context**: Payment processing APIs receive thousands of concurrent requests. NIO allows handling them with few threads; traditional I/O would require 1000+ threads (infeasible).

---

### Q2: Explain BufferedReader vs BufferedInputStream - when use each?

**Answer**:
- **BufferedInputStream**: Handles raw bytes. Use when: binary data (images, serialized objects), you need fine-grained byte control.
- **BufferedReader**: Wraps bytes as characters with encoding awareness. Use when: text files, line-oriented processing, human-readable content.

```
File bytes → BufferedInputStream → byte[] → your code
File bytes → BufferedInputStream → InputStreamReader → BufferedReader → String lines
```

Key insight: All reach same underlying bytes, but BufferedReader provides encoding translation and line convenience.

---

### Q3: What's the buffer lifecycle in NIO? (Common mistake)

**Answer**: Three states—memorize this:

```
WRITE mode:    position=0, limit=capacity
               put() increments position

FLIP:          position=0, limit=position(old)
               switches to READ mode

READ mode:     get() increments position
               hasRemaining() checks position < limit

CLEAR:         position=0, limit=capacity
               ready for next write
```

**Common mistake**: Forgetting `flip()` after filling buffer—then `read()` returns 0 (position at end).

---

### Q4: Direct ByteBuffer vs heap—trade-offs?

**Answer**:

| Aspect | Direct | Heap |
|--------|--------|------|
| Memory | Off-heap, survives GC | On-heap, GC-managed |
| I/O Performance | Better (DMA-friendly) | Slower (JVM copy overhead) |
| GC Pause | No, but harder to deallocate | Automatic |
| Use Case | Long-lived, I/O-heavy | Short-lived, low frequency |
| Allocation Cost | Higher | Lower |

**Banking rule-of-thumb**: Use direct for server NIO (persistent channels), heap for short-term buffers.

---

### Q5: Why does Selector enable high concurrency?

**Answer**: Multiplexing—single thread monitors many channels via OS-level mechanisms (epoll on Linux, kqueue on macOS).

```
Traditional I/O (thread-per-connection):
1000 clients = 1000 threads
Thread overhead: 1000 × 1MB stack = 1GB+ memory
Context switching: expensive

NIO (multiplexing):
1000 clients = 1 selector thread
Selector.select() blocks, OS wakes when any channel ready
Single thread handles all via event loop
Memory: minimal, CPU efficient
```

OS can efficiently check thousands of file descriptors—JVM just uses that capability.

---

### Q6: How does zero-copy work (transferTo)?

**Answer**: `FileChannel.transferTo()` tells OS: "Copy bytes from this file descriptor to this socket descriptor." Kernel handles it directly without passing through application buffer.

```
Traditional:
File → [Kernel buffer] → [App JVM heap] → [Kernel buffer] → Network

Zero-copy:
File → [Kernel buffer] → Network (direct, bypasses JVM)
```

For high-volume file transfers (backup, streaming), saves significant CPU.

---

### Q7: What happens when you serialize a transient field?

**Answer**: It's skipped—not written to stream. On deserialization, field gets default value (0, null, false, etc.). Use case: `private transient LocalDateTime createdAt`—creation time computed at deserialization, not stored.

**Risk**: If you later remove `transient`, old serialized objects won't have this field, causing compatibility issues.

---

### Q8: Files.readString vs BufferedReader—when each?

**Answer**:
- **Files.readString()**: Entire file fits in memory, simplicity preferred, Java 11+ available.
- **BufferedReader + loop**: File may be huge, memory constrained, or streaming processing needed.

```java
// 1MB JSON config
String config = Files.readString(Paths.get("config.json"));

// 1GB log file
try (BufferedReader br = new BufferedReader(new FileReader("huge.log"))) {
    String line;
    while ((line = br.readLine()) != null) {
        processLine(line);  // Doesn't load entire file
    }
}
```

---

### Q9: WatchService real-world use?

**Answer**: Monitor configuration directories for hot-reload (no restart needed).

```java
Path configDir = Paths.get("/config");
watchService.register(configDir, ENTRY_MODIFY);

// When file changes, reload config without restarting app
// Used in banking systems for runtime policy updates
```

Tradeoff: WatchService isn't real-time (OS event delays); polling may be more reliable for critical config.

---

### Q10: Why must you call key.remove() in selector loop?

**Answer**: `selector.selectedKeys()` returns a Set that persists across loops. If you don't remove a processed key, it stays in the set, causing the same channel to process repeatedly.

```java
Set<SelectionKey> readyKeys = selector.selectedKeys();
Iterator<SelectionKey> iterator = readyKeys.iterator();

while (iterator.hasNext()) {
    SelectionKey key = iterator.next();
    iterator.remove();  // MUST remove, or re-processes next loop
    handleEvent(key);
}
```

---

### Q11: Path.normalize() vs toAbsolutePath()?

**Answer**:
- **normalize()**: Removes redundant parts (`.`, `..`).
  ```
  /data/./contracts/../archive → /data/archive
  ```
- **toAbsolutePath()**: Makes absolute (prepends current working directory if relative).
  ```
  contracts/file.csv → /home/user/contracts/file.csv
  ```

Use both: `path.toAbsolutePath().normalize()`

---

### Q12: How does scatter-gather improve performance?

**Answer**: Instead of copying bytes multiple times between buffers, gather writes multiple buffers atomically to channel (single system call).

```
Traditional:
Write header → syscall
Write payload → syscall
Write footer → syscall
(3 system calls, overhead)

Scatter-gather:
ByteBuffer[] {header, payload, footer}
channel.write(buffers)  // Single syscall, all at once
```

Critical for structured data (network protocols, file formats).

---

## Gotchas & Best Practices

### Common Pitfalls

1. **Forgetting flip()**: Buffer in write mode, then try to read—position at end, zero bytes available.

2. **Not removing selectedKeys**: Event loop reprocesses same keys infinitely.

3. **Blocking on selector.select()**: If you do CPU work in event loop, all clients stall. Keep handlers brief; offload to thread pool.

4. **Leaking resources**: Try-with-resources is essential. Unclosed channels, selectors, streams waste OS file descriptors.

5. **Direct buffer bloat**: Allocating massive direct buffers can lead to off-heap memory exhaustion (no GC to reclaim).

6. **Mixing byte and char I/O**: Choose one approach, don't switch between FileInputStream and FileReader on same file.

### Best Practices

- **Use try-with-resources** for all I/O (auto-closes).
- **Buffer appropriately**: 8-16KB for most file I/O, larger for sequential streaming.
- **NIO for server, traditional I/O for simple files**: NIO complexity justified only for concurrency scenarios.
- **Charset awareness**: Always specify UTF-8 explicitly (or your target charset), never assume platform default.
- **Monitor file descriptor limits**: `ulimit -n` on Unix; process can't open more than limit.
- **Consider memory-mapped files** for random access to huge files (>=100MB).
- **Async I/O with frameworks**: Spring WebFlux, Netty handle NIO complexity; rarely use Selector directly in modern code.

---

## References

- **Official Docs**:
  - [java.io Documentation](https://docs.oracle.com/javase/21/docs/api/java.base/java/io/package-summary.html)
  - [java.nio Documentation](https://docs.oracle.com/javase/21/docs/api/java.base/java/nio/package-summary.html)
  - [java.nio.file Documentation](https://docs.oracle.com/javase/21/docs/api/java.base/java/nio/file/package-summary.html)

- **Deep Dives**:
  - "Java NIO" by Ron Hitchens (classic, free online)
  - "Netty in Action" (commercial), covers production NIO patterns
  - [Linux man pages for epoll/kqueue](https://man7.org/linux/man-pages/man7/epoll.7.html)

- **Real-world Frameworks**:
  - Spring WebFlux (reactive streams over NIO)
  - Netty (high-performance NIO framework)
  - Vert.x (async I/O toolkit)

---

## Summary

**Traditional I/O** remains the go-to for simplicity and is sufficient for most file operations. **NIO** is essential for building scalable networked systems and understanding modern frameworks like Spring WebFlux and Netty. **Java 11+ convenience methods** reduce boilerplate and should be preferred when appropriate. Senior engineers recognize the trade-offs and choose the right tool: traditional I/O for config/logging, NIO for servers, streaming APIs for memory efficiency.
