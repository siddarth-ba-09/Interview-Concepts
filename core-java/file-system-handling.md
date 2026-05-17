# File System Handling in Java

---

## 1. Overview

Java has two generations of file I/O APIs:

| API | Package | Introduced | Status |
|-----|---------|-----------|--------|
| **Legacy I/O** (`File`, `FileInputStream`, `FileReader`) | `java.io` | Java 1.0 | Works; avoid for new code |
| **NIO.2** (`Path`, `Files`, `FileSystem`) | `java.nio.file` | Java 7 | **Preferred — use this** |

**Why NIO.2 replaced `java.io.File`:**
- `File` methods silently return `false` or `null` on failure instead of throwing exceptions — impossible to know why an operation failed.
- `File` has no support for symbolic links, file attributes, or file watching.
- `Files` utility class throws `IOException` with meaningful messages.
- `Path` is immutable and composable; `File` is mutable and clunky.

```
java.io.File           (legacy — string-based, error-silent)
java.nio.file.Path     (modern — immutable, OS-agnostic, rich API)
java.nio.file.Files    (utility class — all file operations)
java.nio.file.Paths    (factory for Path instances — Java 7/8)
Path.of(...)           (factory — Java 11+, preferred)
```

---

## 2. `Path` — The Modern File Reference

`Path` represents a location in the file system. It is just a **reference** — the path doesn't have to exist.

### 2.1 Creating Paths

```java
// Java 11+ — preferred
Path p1 = Path.of("/var/log/app/server.log");
Path p2 = Path.of("/var/log", "app", "server.log"); // components joined
Path p3 = Path.of("config", "application.yml");      // relative path

// Java 7/8
Path p4 = Paths.get("/var/log/app/server.log");

// From legacy File (interop)
File file = new File("/tmp/data.csv");
Path path = file.toPath();

// Back to File (legacy interop)
File back = path.toFile();
```

### 2.2 Path Operations (No I/O — Pure String Manipulation)

```java
Path p = Path.of("/home/user/docs/report.pdf");

p.getFileName();          // report.pdf
p.getParent();            // /home/user/docs
p.getRoot();              // /
p.getNameCount();         // 4  (/home, user, docs, report.pdf)
p.getName(2);             // docs
p.isAbsolute();           // true
p.toString();             // /home/user/docs/report.pdf

// Subpath (startIndex inclusive, endIndex exclusive)
p.subpath(1, 3);          // user/docs

// Resolve (append)
Path base = Path.of("/home/user");
base.resolve("docs/report.pdf");     // /home/user/docs/report.pdf
base.resolve(Path.of("/etc/hosts")); // /etc/hosts  (absolute wins)

// Relativize (compute relative path between two paths)
Path from = Path.of("/home/user");
Path to   = Path.of("/home/user/docs/report.pdf");
from.relativize(to);      // docs/report.pdf

// Normalize (remove . and .. components)
Path.of("/home/user/../user/./docs").normalize(); // /home/user/docs

// toAbsolutePath (resolve relative to CWD)
Path.of("config.yml").toAbsolutePath(); // /current/working/dir/config.yml

// toRealPath (resolve symlinks, verify existence — requires I/O)
Path real = Path.of("/var/log/app").toRealPath(); // throws IOException if not exists
```

---

## 3. `Files` Utility Class — All File Operations

### 3.1 Existence and Type Checks

```java
Path p = Path.of("/etc/hosts");

Files.exists(p);                            // true/false
Files.notExists(p);                         // NOT the same as !exists() — handles errors differently
Files.isRegularFile(p);                     // true if it's a file (not dir, symlink)
Files.isDirectory(p);                       // true if directory
Files.isSymbolicLink(p);                    // true if symlink
Files.isReadable(p);
Files.isWritable(p);
Files.isExecutable(p);
Files.isSameFile(p1, p2);                   // true if they refer to same actual file (resolves symlinks)
```

> `Files.notExists()` returns `true` only if the path is confirmed to not exist. `!Files.exists()` returns `true` if it doesn't exist OR if the check fails (e.g., permission denied). Use `notExists()` when you need to distinguish.

### 3.2 Reading Files

```java
Path p = Path.of("/etc/hosts");

// Read all bytes (small files only — loads entire file into memory)
byte[] bytes = Files.readAllBytes(p);

// Read all lines into a List<String> (small files only)
List<String> lines = Files.readAllLines(p);
List<String> lines = Files.readAllLines(p, StandardCharsets.UTF_8);

// Read as a single String (Java 11+)
String content = Files.readString(p);
String content = Files.readString(p, StandardCharsets.UTF_8);

// Stream lines lazily (large files — lines loaded on demand, MUST close stream)
try (Stream<String> stream = Files.lines(p, StandardCharsets.UTF_8)) {
    stream.filter(line -> line.contains("ERROR"))
          .forEach(System.out::println);
} // stream (and underlying reader) closed here

// Buffered reader for manual line processing
try (BufferedReader reader = Files.newBufferedReader(p, StandardCharsets.UTF_8)) {
    String line;
    while ((line = reader.readLine()) != null) {
        process(line);
    }
}

// Raw InputStream (binary files)
try (InputStream in = Files.newInputStream(p, StandardOpenOption.READ)) {
    byte[] buffer = new byte[8192];
    int bytesRead;
    while ((bytesRead = in.read(buffer)) != -1) {
        process(buffer, 0, bytesRead);
    }
}
```

### 3.3 Writing Files

```java
Path p = Path.of("/tmp/output.txt");

// Write bytes (creates or overwrites)
Files.write(p, "Hello World".getBytes(StandardCharsets.UTF_8));

// Write lines
List<String> lines = List.of("line1", "line2", "line3");
Files.write(p, lines, StandardCharsets.UTF_8);

// Write String (Java 11+)
Files.writeString(p, "Hello World", StandardCharsets.UTF_8);

// Append to existing file
Files.writeString(p, "\nNew line", StandardCharsets.UTF_8,
    StandardOpenOption.APPEND, StandardOpenOption.CREATE);

// Buffered writer (large content)
try (BufferedWriter writer = Files.newBufferedWriter(p, StandardCharsets.UTF_8,
        StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)) {
    writer.write("line one");
    writer.newLine();
    writer.write("line two");
}

// Raw OutputStream
try (OutputStream out = Files.newOutputStream(p,
        StandardOpenOption.CREATE, StandardOpenOption.APPEND)) {
    out.write(data);
}
```

### 3.4 `StandardOpenOption` Reference

| Option | Meaning |
|--------|---------|
| `READ` | Open for reading (default for input) |
| `WRITE` | Open for writing |
| `CREATE` | Create file if it doesn't exist |
| `CREATE_NEW` | Create file; fail if it already exists |
| `TRUNCATE_EXISTING` | Truncate file to zero length on open |
| `APPEND` | Append to end of file |
| `DELETE_ON_CLOSE` | Delete file when stream/channel is closed |
| `SYNC` | Every write flushed to physical storage immediately |
| `DSYNC` | Every data write flushed (metadata not guaranteed) |

### 3.5 Copy and Move

```java
Path src  = Path.of("/tmp/source.txt");
Path dest = Path.of("/tmp/dest.txt");

// Copy — fails if dest exists (use REPLACE_EXISTING to overwrite)
Files.copy(src, dest);
Files.copy(src, dest, StandardCopyOption.REPLACE_EXISTING);
Files.copy(src, dest, StandardCopyOption.COPY_ATTRIBUTES); // preserve timestamps, permissions

// Copy from InputStream to file (e.g., download to disk)
try (InputStream in = url.openStream()) {
    Files.copy(in, dest, StandardCopyOption.REPLACE_EXISTING);
}

// Copy file to OutputStream (e.g., serve file over HTTP)
try (OutputStream out = response.getOutputStream()) {
    Files.copy(src, out);
}

// Move (rename within same filesystem = near-instant; cross-filesystem = copy+delete)
Files.move(src, dest);
Files.move(src, dest, StandardCopyOption.REPLACE_EXISTING);
Files.move(src, dest, StandardCopyOption.ATOMIC_MOVE); // atomic on supported filesystems
```

### 3.6 Delete

```java
// Delete — throws NoSuchFileException if doesn't exist
Files.delete(p);

// Delete if exists — returns boolean, does not throw if missing
Files.deleteIfExists(p);

// Delete non-empty directory — must delete contents first
// Use walkFileTree (see Section 5)
```

---

## 4. File Metadata and Attributes

### 4.1 Basic Attributes

```java
Path p = Path.of("/var/log/syslog");

Files.size(p);                         // file size in bytes
Files.getLastModifiedTime(p);          // FileTime object
Files.isHidden(p);                     // starts with . on Unix

// Read all basic attributes in one syscall (efficient)
BasicFileAttributes attrs = Files.readAttributes(p, BasicFileAttributes.class);
attrs.size();
attrs.creationTime();
attrs.lastModifiedTime();
attrs.lastAccessTime();
attrs.isRegularFile();
attrs.isDirectory();
attrs.isSymbolicLink();
attrs.isOther();                        // device file, etc.
```

### 4.2 POSIX Attributes (Unix/Linux/macOS)

```java
PosixFileAttributes posix = Files.readAttributes(p, PosixFileAttributes.class);
posix.owner().getName();               // file owner username
posix.group().getName();               // file group
posix.permissions();                   // Set<PosixFilePermission>

// Set permissions
Set<PosixFilePermission> perms = PosixFilePermissions.fromString("rw-r--r--");
Files.setPosixFilePermissions(p, perms);
```

### 4.3 Setting Metadata

```java
// Update last-modified time
Files.setLastModifiedTime(p, FileTime.from(Instant.now()));

// Set owner (requires privilege)
UserPrincipalLookupService lookup = FileSystems.getDefault().getUserPrincipalLookupService();
UserPrincipal owner = lookup.lookupPrincipalByName("appuser");
Files.setOwner(p, owner);
```

---

## 5. Directory Operations

### 5.1 Create Directories

```java
// Create single directory — fails if parent doesn't exist
Files.createDirectory(Path.of("/tmp/mydir"));

// Create full path (like mkdir -p)
Files.createDirectories(Path.of("/tmp/a/b/c/d"));

// Create with permissions
FileAttribute<Set<PosixFilePermission>> perms =
    PosixFilePermissions.asFileAttribute(PosixFilePermissions.fromString("rwxr-xr-x"));
Files.createDirectory(Path.of("/tmp/secure"), perms);
```

### 5.2 List Directory Contents

```java
Path dir = Path.of("/var/log");

// List immediate children (non-recursive) — MUST close
try (Stream<Path> entries = Files.list(dir)) {
    entries.filter(Files::isRegularFile)
           .forEach(System.out::println);
}

// List with glob pattern matching
try (DirectoryStream<Path> stream = Files.newDirectoryStream(dir, "*.log")) {
    for (Path entry : stream) {
        System.out.println(entry.getFileName());
    }
}
```

### 5.3 Walk the File Tree (Recursive)

#### `Files.walk()` — Simple Stream-Based

```java
// Walk recursively, max depth unlimited
try (Stream<Path> stream = Files.walk(Path.of("/home/user/project"))) {
    stream.filter(Files::isRegularFile)
          .filter(p -> p.toString().endsWith(".java"))
          .forEach(System.out::println);
}

// Limit depth
try (Stream<Path> stream = Files.walk(dir, 2)) { // max 2 levels deep
    // ...
}
```

#### `Files.walkFileTree()` — Visitor Pattern (Full Control)

```java
// Delete a directory and all its contents
Files.walkFileTree(Path.of("/tmp/old-data"), new SimpleFileVisitor<>() {

    @Override
    public FileVisitResult visitFile(Path file, BasicFileAttributes attrs)
            throws IOException {
        Files.delete(file);
        return FileVisitResult.CONTINUE;
    }

    @Override
    public FileVisitResult postVisitDirectory(Path dir, IOException exc)
            throws IOException {
        if (exc != null) throw exc;
        Files.delete(dir);
        return FileVisitResult.CONTINUE;
    }

    @Override
    public FileVisitResult visitFileFailed(Path file, IOException exc)
            throws IOException {
        System.err.println("Failed to visit: " + file + " — " + exc.getMessage());
        return FileVisitResult.CONTINUE; // or TERMINATE to abort
    }
});
```

`FileVisitResult` options:

| Value | Meaning |
|-------|---------|
| `CONTINUE` | Continue traversal |
| `TERMINATE` | Stop traversal immediately |
| `SKIP_SUBTREE` | Skip this directory's children (only in `preVisitDirectory`) |
| `SKIP_SIBLINGS` | Skip remaining siblings of this entry |

### 5.4 `Files.find()` — Walk with Filter

```java
// Find all .log files larger than 10MB, modified in the last 7 days
Instant weekAgo = Instant.now().minus(7, ChronoUnit.DAYS);

try (Stream<Path> results = Files.find(
        Path.of("/var/log"),
        Integer.MAX_VALUE, // max depth
        (path, attrs) -> attrs.isRegularFile()
            && path.toString().endsWith(".log")
            && attrs.size() > 10 * 1024 * 1024
            && attrs.lastModifiedTime().toInstant().isAfter(weekAgo))) {
    results.forEach(System.out::println);
}
```

---

## 6. Temporary Files and Directories

```java
// Create temp file in system temp dir (e.g., /tmp/prefix12345678suffix.txt)
Path tmp = Files.createTempFile("prefix-", ".txt");

// Create in specific directory
Path tmp2 = Files.createTempFile(Path.of("/var/tmp"), "report-", ".csv");

// Create temp directory
Path tmpDir = Files.createTempDirectory("workdir-");

// Auto-delete on JVM exit (best-effort — not guaranteed on crash)
tmp.toFile().deleteOnExit();

// Better: auto-delete with try-with-resources using DELETE_ON_CLOSE
try (OutputStream out = Files.newOutputStream(tmp,
        StandardOpenOption.DELETE_ON_CLOSE)) {
    out.write(data);
} // file deleted when stream closes
```

---

## 7. File Watching — `WatchService`

`WatchService` monitors a directory for file system events (create, modify, delete) without polling.

```java
WatchService watcher = FileSystems.getDefault().newWatchService();

Path configDir = Path.of("/etc/myapp");

// Register directory for events
configDir.register(watcher,
    StandardWatchEventKinds.ENTRY_CREATE,
    StandardWatchEventKinds.ENTRY_MODIFY,
    StandardWatchEventKinds.ENTRY_DELETE);

// Poll loop (typically on a background thread)
Thread watchThread = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        WatchKey key;
        try {
            key = watcher.take(); // blocks until an event arrives
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            break;
        }

        for (WatchEvent<?> event : key.pollEvents()) {
            WatchEvent.Kind<?> kind = event.kind();

            if (kind == StandardWatchEventKinds.OVERFLOW) {
                // Events lost due to OS buffer overflow — do a full rescan
                continue;
            }

            @SuppressWarnings("unchecked")
            WatchEvent<Path> pathEvent = (WatchEvent<Path>) event;
            Path changed = configDir.resolve(pathEvent.context());

            System.out.println(kind.name() + ": " + changed);

            if (kind == StandardWatchEventKinds.ENTRY_MODIFY
                    && changed.endsWith("application.yml")) {
                reloadConfig(); // hot-reload config on change
            }
        }

        boolean valid = key.reset(); // re-register for next events
        if (!valid) break;           // directory was deleted
    }
}, "file-watcher");

watchThread.setDaemon(true);
watchThread.start();
```

**`WatchService` limitations:**
- Monitors one directory level — not recursive (register subdirs explicitly).
- `OVERFLOW` events mean the OS dropped events; always do a full rescan when it occurs.
- On macOS, the native implementation polls internally (high CPU); on Linux it uses `inotify` (efficient).

---

## 8. File Locking — `FileChannel` and `FileLock`

Prevents multiple processes (not threads — see note) from writing to the same file concurrently.

```java
try (FileChannel channel = FileChannel.open(
        Path.of("/var/lock/myapp.lock"),
        StandardOpenOption.CREATE,
        StandardOpenOption.WRITE)) {

    // Try to acquire exclusive lock (non-blocking)
    FileLock lock = channel.tryLock();
    if (lock == null) {
        System.out.println("Another process holds the lock — exiting");
        return;
    }

    try {
        // Only one process holds this lock at a time
        doExclusiveWork();
    } finally {
        lock.release();
    }
}
```

```java
// Shared lock (multiple readers allowed, blocks exclusive writers)
FileLock sharedLock = channel.lock(0, Long.MAX_VALUE, true); // true = shared

// Exclusive lock (blocks all other lockers)
FileLock exclusiveLock = channel.lock(); // or lock(0, Long.MAX_VALUE, false)
```

> **Critical:** `FileLock` is between **processes** — it uses OS-level advisory locking. It does NOT protect against concurrent access by multiple **threads** within the same JVM. For intra-JVM thread safety, use `synchronized` or `ReentrantReadWriteLock` on a shared object.

---

## 9. Memory-Mapped Files — `MappedByteBuffer`

Maps a file region directly into virtual memory. Reading/writing the buffer directly reads/writes the file — no explicit I/O calls. Ideal for large files processed randomly.

```java
Path p = Path.of("/data/large-dataset.bin");

try (FileChannel channel = FileChannel.open(p, StandardOpenOption.READ,
                                                StandardOpenOption.WRITE)) {

    // Map entire file into memory (read-write)
    MappedByteBuffer buffer = channel.map(
        FileChannel.MapMode.READ_WRITE,
        0,                   // start position
        channel.size()       // length
    );

    // Read
    buffer.position(1024);
    byte b = buffer.get();

    // Write — changes are reflected in file (OS manages flush timing)
    buffer.position(2048);
    buffer.put((byte) 42);

    // Force flush to disk immediately
    buffer.force();
}
```

**Use cases:** Log file readers, database storage engines, large binary file processing.

**Caveat:** `MappedByteBuffer` is not released until GC collects it — the file cannot be deleted on Windows while mapped. On JVM crashes, in-flight writes may or may not be flushed. Prefer `force()` for durability.

---

## 10. `java.io.File` — Legacy API (Know It; Avoid It)

```java
File file = new File("/tmp/data.txt");

file.exists();
file.isFile();
file.isDirectory();
file.getName();          // "data.txt"
file.getParent();        // "/tmp"
file.getAbsolutePath();
file.length();           // size in bytes
file.lastModified();     // epoch milliseconds
file.canRead();
file.canWrite();

// SILENT FAILURE — these return false instead of throwing
boolean created = file.createNewFile(); // returns false if already exists — no exception
boolean deleted = file.delete();        // returns false if fails — no exception
boolean renamed = file.renameTo(dest);  // returns false on failure — no exception!

// List directory
String[] names = file.list();           // null if not dir or I/O error — no exception
File[] children = file.listFiles();     // null on failure

// Create directories
file.mkdirs(); // returns false on failure, no reason given
```

**Why to avoid:** All mutation methods return `boolean` on failure. There is no way to know if `delete()` failed because the file didn't exist, because of a permission error, because it's a non-empty directory, or because the path is locked. `Files.delete()` throws `IOException` with the actual reason.

---

## 11. Character Encoding — Always Specify It

```java
// WRONG — uses platform default encoding (varies by OS, JVM flags, locale)
new FileReader("/tmp/data.txt")
new FileWriter("/tmp/output.txt")
Files.readAllLines(path)               // platform default

// CORRECT — always specify charset explicitly
new InputStreamReader(new FileInputStream(path), StandardCharsets.UTF_8)
new OutputStreamWriter(new FileOutputStream(path), StandardCharsets.UTF_8)
Files.readAllLines(path, StandardCharsets.UTF_8)
Files.readString(path, StandardCharsets.UTF_8)
Files.writeString(path, content, StandardCharsets.UTF_8)
```

Platform default can be `UTF-8` on Linux, `Cp1252` (Windows-1252) on Windows, or `MacRoman` on older macOS. Hardcode `StandardCharsets.UTF_8` everywhere.

---

## 12. Production Use Cases & Best Practices

### 12.1 Config File Hot-Reload

Use `WatchService` to monitor the config directory. On `ENTRY_MODIFY` for the config file, re-read and re-apply settings without restarting the application. This is how Spring Cloud Config and Kubernetes ConfigMap updates work under the hood.

### 12.2 Atomic File Write (Write-Then-Replace Pattern)

Never write directly to the final file path for important data. A crash mid-write leaves a corrupt file:

```java
Path target = Path.of("/data/config.json");
Path temp   = Files.createTempFile(target.getParent(), ".tmp-config-", ".json");

try {
    Files.writeString(temp, jsonContent, StandardCharsets.UTF_8,
        StandardOpenOption.SYNC); // ensure data hits disk before move
    Files.move(temp, target,
        StandardCopyOption.REPLACE_EXISTING,
        StandardCopyOption.ATOMIC_MOVE); // atomic on supported filesystems
} catch (Exception e) {
    Files.deleteIfExists(temp); // clean up temp on failure
    throw e;
}
```

`ATOMIC_MOVE` ensures readers either see the old file or the new file — never a half-written file. Supported on most Linux/macOS filesystems (ext4, APFS, XFS) for same-filesystem moves.

### 12.3 Large File Processing with `Files.lines()`

```java
// Process a 10GB log file without loading it into memory
try (Stream<String> lines = Files.lines(Path.of("/logs/access.log"), StandardCharsets.UTF_8)) {
    Map<Integer, Long> statusCounts = lines
        .filter(l -> l.startsWith("2024"))
        .map(l -> extractStatusCode(l))
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
}
```

Always use `try-with-resources` — `Files.lines()` returns a lazy stream backed by an open file handle. Forgetting to close it leaks the file descriptor.

### 12.4 CSV Import Pipeline (Memory-Efficient)

```java
@Transactional
public void importProducts(Path csvFile) throws IOException {
    try (BufferedReader reader = Files.newBufferedReader(csvFile, StandardCharsets.UTF_8)) {
        reader.readLine(); // skip header

        String line;
        List<Product> batch = new ArrayList<>(500);
        while ((line = reader.readLine()) != null) {
            batch.add(parseLine(line));
            if (batch.size() == 500) {
                productRepository.saveAll(batch);
                batch.clear();
                entityManager.flush();
                entityManager.clear(); // avoid Hibernate 1st-level cache bloat
            }
        }
        if (!batch.isEmpty()) productRepository.saveAll(batch);
    }
}
```

### 12.5 Secure Temporary Files

```java
// Create temp file with restricted permissions (owner read/write only)
FileAttribute<Set<PosixFilePermission>> ownerOnly =
    PosixFilePermissions.asFileAttribute(
        EnumSet.of(PosixFilePermission.OWNER_READ, PosixFilePermission.OWNER_WRITE));

Path tmpFile = Files.createTempFile("sensitive-", ".tmp", ownerOnly);
// File is rw------- — other users cannot read it
```

### 12.6 Path Traversal Attack Prevention (Security)

Never allow user input to directly construct a file path:

```java
// VULNERABLE — attacker sends "../../etc/passwd" as filename
Path file = baseDir.resolve(userInputFilename);
Files.readString(file); // reads /etc/passwd!

// SAFE — normalize and verify the resolved path is within the base directory
Path resolved = baseDir.resolve(userInputFilename).normalize();
if (!resolved.startsWith(baseDir.normalize())) {
    throw new SecurityException("Path traversal attempt detected: " + userInputFilename);
}
Files.readString(resolved);
```

---

## 13. Common Pitfalls & Gotchas

### Forgetting to Close `Files.lines()` / `Files.walk()` / `Files.list()`

All return lazy `Stream<Path>` or `Stream<String>` backed by open file descriptors. Not closing them leaks file descriptors — eventually causes `Too many open files` error under load.

```java
// WRONG — stream and file handle leaked if exception occurs
Stream<String> lines = Files.lines(path);
lines.forEach(process);

// CORRECT
try (Stream<String> lines = Files.lines(path)) {
    lines.forEach(process);
}
```

---

### `Files.exists()` TOCTOU Race Condition

```java
// WRONG — race condition between check and use (Time-Of-Check to Time-Of-Use)
if (!Files.exists(target)) {
    Files.createFile(target); // another thread may have created it between check and create
}

// CORRECT — use CREATE_NEW which atomically creates or fails
try {
    Files.writeString(target, content, StandardOpenOption.CREATE_NEW);
} catch (FileAlreadyExistsException e) {
    // Handle existing file
}
```

---

### `Files.copy()` Does Not Copy Recursively

`Files.copy()` copies a single file or an empty directory. It throws `DirectoryNotEmptyException` on a populated directory. Use `Files.walkFileTree()` for recursive copy:

```java
Path src = Path.of("/src/dir");
Path dst = Path.of("/dst/dir");

Files.walkFileTree(src, new SimpleFileVisitor<>() {
    @Override
    public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs)
            throws IOException {
        Files.createDirectories(dst.resolve(src.relativize(dir)));
        return FileVisitResult.CONTINUE;
    }

    @Override
    public FileVisitResult visitFile(Path file, BasicFileAttributes attrs)
            throws IOException {
        Files.copy(file, dst.resolve(src.relativize(file)),
            StandardCopyOption.REPLACE_EXISTING);
        return FileVisitResult.CONTINUE;
    }
});
```

---

### `File.renameTo()` Fails Silently Across Filesystems

`File.renameTo()` returns `false` when moving across filesystems (e.g., `/tmp` → `/home`). No exception, no reason. Use `Files.move()` — it falls back to copy+delete and throws `IOException` on failure.

---

### `FileOutputStream` Truncates on Creation

```java
// TRUNCATES the file immediately, even before writing anything
FileOutputStream fos = new FileOutputStream("/important/data.bin");
// If the process crashes here, data.bin is now empty
```

Use the write-then-replace pattern (temp file + `Files.move(ATOMIC_MOVE)`) for important data.

---

### Hardcoded Path Separators

```java
// WRONG — breaks on Windows (uses \)
String path = "/home/user" + "/" + "file.txt";

// CORRECT — use Path.of() for composition
Path path = Path.of("home", "user", "file.txt");
// or
Path path = Path.of("home/user").resolve("file.txt"); // Path handles separators
```

---

## 14. Interview Questions — Standard

**Q1: What is the main difference between `java.io.File` and `java.nio.file.Path`?**
`File` is the legacy API where failure methods return `false`/`null` silently — no exception, no reason. `Path`/`Files` (NIO.2, Java 7+) throw `IOException` with meaningful messages on failures. `Path` also supports symbolic links, file attributes, atomic moves, file watching, and directory streams. Always use NIO.2 for new code.

**Q2: What happens if you forget to close a `Stream<String>` returned by `Files.lines()`?**
The `Stream<String>` from `Files.lines()` is backed by an open `BufferedReader` and file descriptor. Not closing it leaks the file descriptor. Under load, this causes `java.io.IOException: Too many open files` when the process's file descriptor limit (e.g., `ulimit -n 1024`) is reached. Always wrap in `try-with-resources`.

**Q3: What is `StandardCopyOption.ATOMIC_MOVE` and when would you use it?**
`ATOMIC_MOVE` instructs the OS to rename the file in a single atomic operation (a single filesystem rename syscall). Readers see either the old file or the new file — never a partially written state. Used in the write-to-temp-then-move pattern to ensure data durability. If the target is on a different filesystem, the JVM falls back to copy+delete (non-atomic) and may throw `AtomicMoveNotSupportedException`.

**Q4: What is the difference between `Files.exists()` and `Files.notExists()`?**
`Files.exists(p)` returns `true` if the path exists, `false` if it doesn't OR if existence cannot be determined (e.g., permission denied). `Files.notExists(p)` returns `true` only if the path is confirmed to not exist — it returns `false` if the check itself fails. Use `notExists()` when you need to distinguish "definitely doesn't exist" from "unknown."

**Q5: How does `WatchService` work and what are its limitations?**
`WatchService` registers directories for `ENTRY_CREATE`, `ENTRY_MODIFY`, `ENTRY_DELETE` events. On Linux it uses the `inotify` kernel mechanism (efficient); on macOS it uses polling (high CPU). Limitations: it only monitors one directory level (not recursive), `OVERFLOW` events mean the OS dropped events (requiring a full rescan), and it runs on a separate thread that must be managed manually.

**Q6: What is a `FileLock` and does it protect against concurrent threads?**
`FileLock` is an OS-level advisory lock acquired via `FileChannel.lock()`. It provides mutual exclusion between OS **processes** — useful for ensuring only one JVM instance processes a file. It does NOT protect against concurrent access by multiple **threads** within the same JVM. For intra-JVM thread safety, use `synchronized` blocks or `ReentrantReadWriteLock`.

---

## 15. Interview Questions — Tricky / Scenario-Based

**Q1: You process a 50GB log file with `Files.readAllLines()`. What happens?**
`Files.readAllLines()` loads the entire file into a `List<String>` in heap memory. A 50GB file will throw `OutOfMemoryError`. The correct approach is `Files.lines()` which returns a lazy `Stream<String>`, reading lines on demand from disk. The entire file is never in memory at once.

**Q2: Two threads call `Files.createFile(path)` simultaneously. What happens?**
`Files.createFile()` uses `CREATE_NEW` semantics — it atomically creates the file or throws `FileAlreadyExistsException`. One thread succeeds; the other gets the exception. This is the safe way to implement leader election or single-instance locking at the filesystem level.

**Q3: You write a config file with `Files.writeString()`. The JVM crashes mid-write. What does the file contain?**
It contains whatever was written before the crash — potentially partial/corrupt data. The file was opened and truncated at the start of the write. For critical files, always use the write-to-temp-then-atomic-move pattern: write fully to a temp file, then `Files.move()` with `ATOMIC_MOVE` to the target. A crash mid-write leaves the temp file (harmless), and the target is either the complete old or complete new version.

**Q4: `Files.walk()` returns a stream that includes the starting directory itself. You filter for regular files but also get zero results from empty subdirectories. Is this expected?**
Yes, this is expected and correct. `Files.walk()` emits every path in the tree, starting with the root directory itself. Filtering with `Files::isRegularFile` correctly excludes the root and all subdirectories, leaving only files. Empty subdirectories are directories, not regular files, so they are filtered out. Use `Files.find()` with a `BiPredicate` for more complex filtering including attribute checks in one pass.

**Q5: Why does `file.renameTo(dest)` return `false` when moving from `/tmp` to `/home/user`?**
`/tmp` and `/home` are often on different filesystems (or at minimum different mount points). A `rename()` syscall only works within the same filesystem. Across filesystems, a copy-then-delete is needed. `File.renameTo()` silently returns `false` when the underlying OS rename fails for any reason. `Files.move()` handles cross-filesystem moves (falls back to copy+delete) and throws a meaningful `IOException` if it still fails.

---

## 16. Quick Revision Summary

- **Always use NIO.2** (`Path`, `Files`) over `java.io.File` for new code — `File` silently returns `false`/`null` on failure; `Files` throws meaningful `IOException`.
- `Path` is an immutable reference — it doesn't mean the path exists. Use `Files.exists()` to check.
- `Files.lines()`, `Files.walk()`, `Files.list()` return **lazy streams backed by open file handles** — always wrap in `try-with-resources`.
- **Always specify charset explicitly** (`StandardCharsets.UTF_8`) — never rely on platform default.
- For durability: write to a **temp file + `ATOMIC_MOVE`** to target — prevents corrupt files on crash.
- `Files.readAllLines()` / `Files.readAllBytes()` load the entire file into memory — only for small files.
- `Files.copy()` is NOT recursive; use `Files.walkFileTree()` for recursive copy or delete.
- `FileLock` is inter-process locking, not intra-JVM thread locking.
- `WatchService` is event-driven (not polling in JVM); handle `OVERFLOW` events with a full rescan.
- Path traversal attacks: always `normalize()` and `startsWith(baseDir)` before serving user-provided paths.
- `StandardCopyOption.ATOMIC_MOVE` ensures readers see either old or new file — never partial.
- `Files.createFile()` is atomic (`CREATE_NEW`) — use it for single-instance and leader-election patterns.
- `Files.notExists()` ≠ `!Files.exists()` — the former confirms absence; the latter is ambiguous on error.
