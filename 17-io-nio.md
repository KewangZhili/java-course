# Module 17 — File I/O and NIO.2

> Goal: read and write files, walk directories, handle binary and text data, choose the right API for the size and shape of the work. Understand the legacy `java.io` and the modern `java.nio.file`, and when to reach for low-level NIO channels.

---

## 1. Two generations of I/O APIs

Java has two layered I/O families:

- **`java.io`** (1995, "old I/O") — stream-oriented, blocking, `File` / `FileInputStream` / `BufferedReader`. Still everywhere because it's simple and fine for line-oriented work.
- **`java.nio` + `java.nio.file`** ("New I/O", 2002 + 2011) — buffer-/channel-oriented, with `Path`, `Files` utility class, FileSystem abstraction, async channels. The modern path for file work.

Day-to-day:
- For **small/medium files**: `Files.readString`, `Files.readAllLines`, `Files.lines`. Trivial.
- For **streaming or transformation**: `Files.newBufferedReader` / `newBufferedWriter` with try-with-resources.
- For **binary**: `Files.readAllBytes` / `newInputStream` / `newOutputStream`.
- For **filesystem operations**: `Path` + the `Files` utility class.
- For **really large or async**: NIO channels (`FileChannel`, `AsynchronousFileChannel`). Rare.

---

## 2. `Path` — the modern file reference

A `Path` represents a filesystem path (it does not require the file to exist). It replaces the old `java.io.File`.

```java
import java.nio.file.*;

Path p = Path.of("data", "input.txt");           // platform-aware separator
Path q = Paths.get("/etc/hosts");                 // legacy factory, identical
Path home = Path.of(System.getProperty("user.home"));

p.getFileName();           // "input.txt"
p.getParent();             // "data"
p.toAbsolutePath();
p.normalize();             // resolves "."/".." in the path string
p.resolve("sub.txt");      // "data/input.txt/sub.txt" — combine paths
p.relativize(other);
p.toString();
```

`Path` is immutable, hashable, comparable. `Path.of("a","b")` is preferred over building strings with `/`.

---

## 3. Reading whole files

For "I just want the file contents":

```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;

String text = Files.readString(p);                                // UTF-8 by default since Java 11
String text2 = Files.readString(p, StandardCharsets.UTF_16);

byte[] bytes = Files.readAllBytes(p);

List<String> lines = Files.readAllLines(p);                       // UTF-8
```

`readString`/`readAllBytes` load the entire file into memory. Don't use them on multi-GB files.

For larger files, use a **stream**:

```java
try (Stream<String> lines = Files.lines(p)) {                      // lazy, line-by-line
    long count = lines.filter(s -> !s.isBlank()).count();
}
// IMPORTANT: try-with-resources to close the underlying file
```

---

## 4. Writing whole files

```java
Files.writeString(p, "hello\n");                                   // UTF-8 by default since Java 11
Files.writeString(p, "hello\n", StandardOpenOption.APPEND);

Files.write(p, new byte[]{1, 2, 3});

Files.write(p, List.of("line 1", "line 2"));                       // UTF-8
```

By default, write modes truncate the file. Append with `StandardOpenOption.APPEND`. Create-only (fail if exists) with `CREATE_NEW`.

---

## 5. Buffered streaming (when you don't want to load everything)

For text:
```java
try (BufferedReader in = Files.newBufferedReader(p);
     BufferedWriter out = Files.newBufferedWriter(q)) {
    String line;
    while ((line = in.readLine()) != null) {
        out.write(transform(line));
        out.newLine();
    }
}
```

For binary:
```java
try (InputStream in = Files.newInputStream(p);
     OutputStream out = Files.newOutputStream(q)) {
    in.transferTo(out);          // since Java 9 — pipe entire stream
}
```

`transferTo` is the right one-liner for "copy all bytes from in to out".

`BufferedReader.readLine()` returns `null` at end-of-stream. (No exception, no `Optional`.) Don't forget to check it.

---

## 6. Filesystem operations

The `Files` static class is the API surface for filesystem actions:

```java
Files.exists(p);
Files.isDirectory(p);
Files.isRegularFile(p);
Files.size(p);                                          // bytes
Files.getLastModifiedTime(p);

Files.createDirectories(Path.of("data/sub/dir"));       // creates parents too
Files.createFile(p);                                    // fails if exists
Files.createTempFile("prefix", ".ext");
Files.createTempDirectory("scratch");

Files.copy(src, dst);
Files.copy(src, dst, StandardCopyOption.REPLACE_EXISTING);
Files.move(src, dst);
Files.delete(p);                                         // throws if missing
Files.deleteIfExists(p);

Files.list(dir);                                         // shallow Stream<Path> — close it!
Files.walk(dir);                                          // deep Stream<Path> — close it!
Files.find(dir, depth, (path, attrs) -> attrs.isRegularFile());
```

Streams returned by `list`, `walk`, `find`, `lines` *all* hold OS handles. Use them inside `try-with-resources`.

```java
try (var stream = Files.walk(root)) {
    long count = stream.filter(Files::isRegularFile).count();
}
```

---

## 7. Charsets — always be explicit (or rely on UTF-8 by default)

Pre-Java 18, the default charset varied by platform (UTF-8 on macOS/Linux, often Windows-1252 on Windows). Since Java 18, the JDK default is UTF-8 on all platforms (JEP 400).

Best practice: always pass `StandardCharsets.UTF_8` explicitly when in doubt. The standard charsets are constants in `java.nio.charset.StandardCharsets`:
- `UTF_8`
- `UTF_16` / `UTF_16BE` / `UTF_16LE`
- `US_ASCII`
- `ISO_8859_1`

```java
String s = new String(bytes, StandardCharsets.UTF_8);
byte[] b = "hello".getBytes(StandardCharsets.UTF_8);
```

---

## 8. The legacy `java.io` family — when you'll see it

Old code uses these. They still work and are still common:

```java
File f = new File("foo.txt");                    // legacy file reference
FileInputStream  in  = new FileInputStream(f);
FileOutputStream out = new FileOutputStream(f, true);   // append=true

BufferedReader br = new BufferedReader(new InputStreamReader(in, StandardCharsets.UTF_8));
PrintWriter   pw = new PrintWriter(out, /* autoFlush */ true, StandardCharsets.UTF_8);
```

Bridges between old and new:
```java
File   f = path.toFile();
Path   p = file.toPath();
```

Convention in new code: use `Path` and `Files`. Bridge to `File` only when an old API requires it.

### 8.1 The "decorator" pattern in `java.io`

Old streams compose by wrapping:
```
FileInputStream → BufferedInputStream → DataInputStream
```

```java
try (var in = new DataInputStream(new BufferedInputStream(new FileInputStream("data.bin")))) {
    int  v = in.readInt();
    long w = in.readLong();
}
```

This stack-of-wrappers style is why old `java.io` code looks dense. The wrappers add buffering, primitive readers, gzip, encryption. Each wrapper closes its delegate when closed.

---

## 9. Channels and buffers — the lower level

`java.nio` introduced `Channel` (a connection to a file/socket) and `Buffer` (a fixed-size container of bytes/chars/ints/...). For most application code, `Files.read*` and `try-with-resources` streams are enough. Channels matter for:

- Memory-mapped files (`FileChannel.map`).
- Asynchronous I/O (`AsynchronousFileChannel`).
- Networking (`SocketChannel`, `ServerSocketChannel`).
- Selector-based multiplexing (used by Netty internally).

Quick sketch (read a file via channel):
```java
try (FileChannel ch = FileChannel.open(p, StandardOpenOption.READ)) {
    ByteBuffer buf = ByteBuffer.allocate(8192);
    while (ch.read(buf) > 0) {
        buf.flip();                  // switch from write to read mode
        // ... consume buf ...
        buf.clear();
    }
}
```

Memory-mapped:
```java
try (FileChannel ch = FileChannel.open(p, StandardOpenOption.READ)) {
    MappedByteBuffer mm = ch.map(FileChannel.MapMode.READ_ONLY, 0, ch.size());
    // mm is a view into OS page cache — fast random access for big files
}
```

Useful when you need to scan multi-GB files at speed. Most application code never touches it.

---

## 10. Standard input / output

```java
System.out.println("hi");
System.err.println("oops");

// reading lines from stdin
try (var br = new BufferedReader(new InputStreamReader(System.in))) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println("got: " + line);
    }
}
```

`Scanner` is convenient but slow:
```java
var sc = new Scanner(System.in);
int n = sc.nextInt();
String s = sc.next();
```
For competitive programming, use `BufferedReader` (Module 29).

---

## 11. Watching directories (FileSystem events)

`WatchService` lets you wait for create/modify/delete events on a directory:

```java
try (WatchService ws = FileSystems.getDefault().newWatchService()) {
    Path dir = Path.of("/tmp/watched");
    dir.register(ws, StandardWatchEventKinds.ENTRY_CREATE,
                     StandardWatchEventKinds.ENTRY_MODIFY,
                     StandardWatchEventKinds.ENTRY_DELETE);
    while (true) {
        WatchKey k = ws.take();              // blocks
        for (WatchEvent<?> e : k.pollEvents()) {
            System.out.println(e.kind() + " " + e.context());
        }
        k.reset();
    }
}
```

Simple and platform-portable, with the caveat that on macOS the default implementation polls (slower than native FSEvents).

---

## 12. Summary: which method to call

| Task | API |
|---|---|
| Read whole file as String | `Files.readString(p)` |
| Read whole file as bytes | `Files.readAllBytes(p)` |
| Read line by line, large file | `try (var s = Files.lines(p)) { ... }` |
| Read binary, large | `Files.newInputStream(p)` + buffered |
| Write String | `Files.writeString(p, s)` |
| Write lines | `Files.write(p, lines)` |
| Append | `Files.writeString(p, s, StandardOpenOption.APPEND)` |
| Copy file | `Files.copy(src, dst)` |
| Walk directory | `try (var s = Files.walk(root)) { ... }` |
| Memory-map huge file | `FileChannel.map(...)` |
| Watch a directory | `WatchService` |

---

## 13. A worked example — recursively count word occurrences

```java
import java.nio.file.*;
import java.util.*;
import java.util.stream.*;

public class WordCount {
    public static Map<String, Long> count(Path root) throws Exception {
        try (Stream<Path> files = Files.walk(root)) {
            return files
                .filter(Files::isRegularFile)
                .filter(p -> p.toString().endsWith(".txt"))
                .flatMap(p -> {
                    try { return Files.lines(p); }            // lazy, closes when downstream is closed (caveat — see below)
                    catch (Exception e) { throw new RuntimeException(e); }
                })
                .flatMap(line -> Stream.of(line.split("\\s+")))
                .filter(w -> !w.isBlank())
                .map(String::toLowerCase)
                .collect(Collectors.groupingBy(w -> w, Collectors.counting()));
        }
    }

    public static void main(String[] args) throws Exception {
        Map<String, Long> counts = count(Path.of(args[0]));
        counts.entrySet().stream()
              .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
              .limit(10)
              .forEach(System.out::println);
    }
}
```

The caveat: `Files.lines` returns a `Stream<String>` that holds an open file. The outer try-with-resources closes the `Files.walk` stream but the inner `Files.lines` streams may leak unless the pipeline closes them transitively. For one-off scripts on small trees this is fine; for long-running programs, restructure to close each `lines` stream explicitly.

---

## 14. Try this

1. Implement `long countLines(Path p)` using `Files.lines`. Compare to `Files.readAllLines(p).size()` for memory on a large file.
2. Implement `void copyDir(Path src, Path dst)` recursively, preserving last-modified times. Use `Files.walk` and `Files.copy` with `COPY_ATTRIBUTES`.
3. Why does this print zero?
   ```java
   var lines = Files.lines(p);    // missing try-with-resources
   long n = lines.count();
   System.out.println(n);
   ```
   (Trick — actually it works, but the file handle leaks. Wrap with try-with-resources.)
4. Read a binary file 4096 bytes at a time with `Files.newInputStream` and a manual `byte[] buf = new byte[4096];` loop. Compute its SHA-256 with `MessageDigest.getInstance("SHA-256")`.

---

**Next:** [Module 18 — Concurrency basics](./18-concurrency-basics.md)
