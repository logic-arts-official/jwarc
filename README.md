# jwarc [![](https://maven-badges.herokuapp.com/maven-central/org.netpreserve/jwarc/badge.svg?style=flat)](https://maven-badges.herokuapp.com/maven-central/org.netpreserve/jwarc) [![](https://www.javadoc.io/badge/org.netpreserve/jwarc.svg)](https://www.javadoc.io/doc/org.netpreserve/jwarc)

A high-performance Java library for reading and writing WARC (Web ARChive) files. This library includes a high-level API modeling
the standard record types as individual classes with typed accessors. The API is extensible and you can register
extension record types and accessors for extension header fields.

## Documentation

- **[API Reference](API-REFERENCE.md)** - Quick reference guide for common APIs
- **[Architecture](ARCHITECTURE.md)** - Design and implementation details
- **[Contributing](CONTRIBUTING.md)** - Guide for developers and contributors
- **[Changelog](CHANGELOG.md)** - Version history and release notes
- **[Backlog](BACKLOG.md)** - Future features and enhancements roadmap
- **[JavaDoc](https://www.javadoc.io/doc/org.netpreserve/jwarc)** - Complete API documentation

## Features

- **High-level typed API** for WARC record types
- **High-performance parsing** using Ragel finite state machine
- **Multiple compression formats**: GZIP, Zstandard (zstd), and Brotli
- **Protocol support**: HTTP, HTTPS, Gemini
- **Lenient parsing** modes for handling non-compliant records
- **NIO-based I/O** with minimal data copying and shared buffers
- **Filtering and validation** with expression language
- **Command-line tools** for common WARC operations
- **ARC format support** with automatic detection and parsing

## Quick Start

### Basic Usage Example

```java
try (WarcReader reader = new WarcReader(FileChannel.open(Paths.get("example.warc")))) {
    for (WarcRecord record : reader) {
        if (record instanceof WarcResponse && record.contentType().base().equals(MediaType.HTTP)) {
            WarcResponse response = (WarcResponse) record;
            System.out.println(response.http().status() + " " + response.target());
        }
    }
}
```

## Quick Developer Setup

Want to contribute or build from source? Get up and running in minutes:

### Prerequisites

- **JDK 8 or later** ([Eclipse Temurin](https://adoptium.net/) or [Oracle JDK](https://www.oracle.com/java/technologies/downloads/) recommended)
  - Java 25 recommended for new development
  - JDK 8 minimum for runtime compatibility
- **Apache Maven 3.6+** ([Download](https://maven.apache.org/download.cgi))
- **Git** ([Download](https://git-scm.com/downloads))

### Get Up and Running

```bash
# 1. Clone the repository
git clone https://github.com/iipc/jwarc.git
cd jwarc

# 2. Build the project (includes running tests)
mvn clean package

# 3. Run the command-line tool
java -jar target/jwarc-0.32.1-SNAPSHOT.jar

# 4. Or run a specific command
java -jar target/jwarc-0.32.1-SNAPSHOT.jar ls test-resources/*.warc
```

### Verify Your Setup

```bash
# Run all tests
mvn test

# Build without tests (faster)
mvn package -DskipTests

# Generate JavaDoc
mvn javadoc:javadoc

# Check for dependency updates
mvn versions:display-dependency-updates
```

### Common Issues

**Issue**: `java.lang.UnsupportedClassVersionError`  
**Solution**: Your Java version is too old. Install JDK 8 or later.

**Issue**: Maven not found  
**Solution**: Add Maven to your PATH or use `./mvnw` wrapper if available.

**Issue**: Tests fail during build  
**Solution**: Try `mvn clean test` to ensure a fresh build. Check Java version compatibility.

### Next Steps

- Read the [Contributing Guide](CONTRIBUTING.md) for detailed development guidelines
- Check the [Architecture](ARCHITECTURE.md) document to understand the codebase
- Browse [open issues](https://github.com/iipc/jwarc/issues) for contribution ideas
- Review the [Backlog](BACKLOG.md) for upcoming features

## Technical Details

It uses a finite state machine parser generated from a strict [grammar](https://github.com/iipc/jwarc/blob/master/src/org/netpreserve/jwarc/WarcParser.rl)
using [Ragel](http://www.colm.net/open-source/ragel/). There is an optional lenient mode which can handle some forms of non-compliant WARC records.
ARC and HTTP parsing is lenient by default.

Gzipped records are automatically decompressed. The library also supports **Zstandard (zstd)** compression for reading and **Brotli** 
decompression (via optional `org.brotli:dec` dependency). The parser interprets ARC/1.1 records as if they are a WARC dialect and
populates the appropriate WARC headers.

All I/O is performed using NIO and an effort is made to minimize data copies and share buffers whenever feasible.
Direct buffers and even memory-mapped files can be used with uncompressed WARCs.

## Getting it

### As a Library

To use as a library add jwarc as a dependency from [Maven Central](https://maven-badges.herokuapp.com/maven-central/org.netpreserve/jwarc).

**Maven:**
```xml
<dependency>
    <groupId>org.netpreserve</groupId>
    <artifactId>jwarc</artifactId>
    <version>0.32.0</version>
</dependency>
```

**Gradle:**
```gradle
implementation 'org.netpreserve:jwarc:0.32.0'
```

### As a Command-Line Tool

To use as a command-line tool install [Java 8 or later](https://adoptopenjdk.net/), download
the latest [release jar](https://github.com/iipc/jwarc/releases) and run it using:
 
    java -jar jwarc-{version}.jar

### Building from Source

If you would prefer to build it from source install [JDK 8+](https://adoptopenjdk.net/) and 
[Maven](https://maven.apache.org/) and then run:

    mvn package

See [CONTRIBUTING.md](CONTRIBUTING.md) for more details on building and developing jwarc.

## Dependencies

### Required

- **Java 8+** (Java 25 recommended for new development)
- **zstd-jni**: For Zstandard compression support

### Optional

- **org.brotli:dec**: For Brotli decompression support (not included by default)

Add Brotli support to your Maven project:
```xml
<dependency>
    <groupId>org.brotli</groupId>
    <artifactId>dec</artifactId>
    <version>0.1.2</version>
</dependency>
```

## Examples

### Saving a remote resource

```java
try (WarcWriter writer = new WarcWriter(System.out)) {
    writer.fetch(URI.create("http://example.org/"));
}
```

### Writing records

```java
// write a warcinfo record
// date and record id will be populated automatically if unset
writer.write(new Warcinfo.Builder()
    .fields("software", "my-cool-crawler/1.0",
            "robots", "obey")
    .build());

// we can also supply a specific date
Instant captureDate = Instant.now();

// write a request but keep a copy of it to reference later
WarcRequest request = new WarcRequest.Builder()
    .date(captureDate)
    .target(uri)
    .contentType("application/http")
    .body(bodyStream, bodyLength)
    .build();
writer.write(request);

// write a response referencing the request
WarcResponse response = new WarcResponse.Builder()
    .date(captureDate)
    .target(uri)
    .contentType("application/http")
    .body("HTTP/1.0 200 OK\r\n...".getBytes())
    .concurrentTo(request.id())
    .build();
writer.write(response);
```

### Filter expressions

The [WarcFilter](https://www.javadoc.io/page/org.netpreserve/jwarc/latest/org/netpreserve/jwarc/WarcFilter.html) class
provides a simple filter expression language for matching WARC records. For example here's a moderately complex filter
which matches all records that are not image resources or image responses:

     !((warc-type == "resource" && content-type =~ "image/.*") || 
       (warc-type == "response" && http:content-type =~ "image/.*")) 

WarcFilter implements `Predicate<WarcRecord>` and can be used conveniently with streams of records:

```java
long errorCount = warcReader.records().filter(WarcFilter.compile(":status >= 400")).count();
``` 

Their real power though is as a building block for user-supplied options.

## Protocol Support

### HTTP/HTTPS

Full support for parsing HTTP requests and responses, including:
- Standard and custom headers
- Chunked transfer encoding
- Multiple content encodings (gzip, deflate, brotli)
- Lenient parsing for handling non-compliant HTTP

### Gemini Protocol

jwarc includes support for the [Gemini protocol](https://gemini.circumlunar.space/), a lightweight alternative to HTTP:

- Parse Gemini requests and responses with `GeminiRequest` and `GeminiResponse`
- Automatic mapping of Gemini status codes to HTTP equivalents
- Full support for Gemini text/gemini content type

Example:
```java
GeminiResponse response = GeminiResponse.parse(channel, buffer);
System.out.println("Status: " + response.status());
System.out.println("HTTP equivalent: " + response.statusHttpEquivalent());
System.out.println("Meta: " + response.meta());
``` 

### Command-line tools

jwarc also includes a comprehensive set of command-line tools which serve as [examples](src/org/netpreserve/jwarc/tools/). 

#### Available Commands

List available commands:

    java -jar jwarc.jar

Get help for a specific command:

    java -jar jwarc.jar <command> --help

#### Tool Examples

**Capture a URL** (without subresources):

    java -jar jwarc.jar fetch http://example.org/ > example.warc

**List records** in a WARC file:

    java -jar jwarc.jar ls example.warc

**Create a CDX file:**

    java -jar jwarc.jar cdx example.warc > records.cdx

**Extract a record** by offset:

    java -jar jwarc.jar extract example.warc 1234 > record.warc

**Filter records** by criteria:

    java -jar jwarc.jar filter ':status == 200 && http:content-type =~ "text/html(;.*)?"' example.warc > pages.warc

**Deduplicate records:**

    java -jar jwarc.jar dedupe --cache-size 10000 example.warc > deduped.warc

**Validate WARC files:**

    java -jar jwarc.jar validate example.warc

**Print statistics:**

    java -jar jwarc.jar stats example.warc

**Run a replay proxy and web server:**

    export PORT=8080
    java -jar jwarc.jar serve example.warc

**Replay each page** and use [headless Chrome](https://developers.google.com/web/updates/2017/04/headless-chrome)
to render a screenshot:

    export BROWSER=/opt/google/chrome/chrome
    java -jar jwarc.jar screenshot example.warc > screenshots.warc

**Run a recording proxy** server. This will generate self-signed SSL certificates so you will
need to turn off TLS verification in the client. For Chrome/Chromium use the `--ignore-certificate-errors`
command-line option:

    export PORT=8080
    java -jar jwarc.jar recorder > example.warc

    chromium --proxy-server=http://localhost:8080 --ignore-certificate-errors

**Record a command** that obeys the http(s)_proxy and CURL_CA_BUNDLE environment variables:

    java -jar jwarc.jar recorder -o example.warc curl http://example.org/

**Capture a page** by recording headless Chrome:

    export BROWSER=/opt/google/chrome/chrome
    java -jar jwarc.jar record > example.warc

**Save replayed pages** as WARC records:

    java -jar jwarc.jar saveback http://localhost:8080 > replay.warc 

## API Quick Reference

See the [javadoc](https://www.javadoc.io/doc/org.netpreserve/jwarc) for complete documentation or the [API-REFERENCE.md](API-REFERENCE.md) for a concise quick reference guide.

### [WarcReader](https://www.javadoc.io/page/org.netpreserve/jwarc/latest/org/netpreserve/jwarc/WarcReader.html)

```java
              new WarcReader(stream|path|channel);                // opens a WARC file for reading
                  reader.close();                                 // closes the underlying channel
(WarcCompression) reader.compression();                           // type of compression: NONE or GZIP
       (Iterator) reader.iterator();                              // an iterator over the records
     (WarcRecord) reader.next();                                  // reads the next record
                  reader.registerType("myrecord", MyRecord::new); // registers a new record type
                  reader.setLenient(true);                        // enables lenient parsing mode
```

### [WarcWriter](https://www.javadoc.io/page/org.netpreserve/jwarc/latest/org/netpreserve/jwarc/WarcWriter.html)

```java
                new WarcWriter(channel, NONE|GZIP);    // opens a WARC file for writing
                    writer.fetch(uri);                 // downloads a resource recording the request and response
             (long) writer.position();                 // byte position the next record will be written to
                    writer.write(record);              // adds a record to the WARC file
```
        
### Record types

    Message
      HttpMessage
        HttpRequest
        HttpResponse
      WarcRecord
        Warcinfo            (warcinfo)
        WarcTargetRecord
          WarcContinuation  (continuation)
          WarcConversion    (conversion)
          WarcCaptureRecord
            WarcMetadata    (metadata)
            WarcRequest     (request)
            WarcResource    (resource)
            WarcResponse    (response)
            WarcRevisit     (revisit)

#### [Message](https://www.javadoc.io/page/org.netpreserve/jwarc/latest/org/netpreserve/jwarc/Message.html)

The basic building block of both HTTP protocol and WARC file format is a message consisting of set of named header
fields and a body. Header field names are case-insensitvie and may have multiple values.

```java
             (BodyChannel) message.body();                     // the message body as a ReadableByteChannel
                    (long) message.body().position();          // the next byte position to read from
                     (int) message.body().read(byteBuffer);    // reads a sequence of bytes from the body
                    (long) message.body().size();              // the length in bytes of the body
             (InputStream) message.body().stream();            // views the body as an InputStream
                  (String) message.contentType();              // the media type of the body
                 (Headers) message.headers();                  // the header fields
            (List<String>) message.headers().all("Cookie");    // all values of a header
                 (boolean) message.headers().contains("TE", "deflate"); // tests if a value is present
        (Optional<String>) message.headers().first("Cookie");  // the first value of a header
(Map<String,List<String>>) message.headers().map();            // views the header fields as a map
        (Optional<String>) message.headers().sole("Location"); // throws if header has multiple values
         (ProtocolVersion) message.version();                  // the protocol version (e.g. HTTP/1.0 or WARC/1.1)
```

#### [WarcRecord](https://www.javadoc.io/page/org.netpreserve/jwarc/latest/org/netpreserve/jwarc/WarcRecord.html)

Methods available on all WARC records:

```java
  (Optional<Digest>) record.blockDigest();   // value of hash function applied to bytes of body
           (Instant) record.date();          // instant that data capture began
               (URI) record.id();            // globally unique record identifier
    (Optional<Long>) record.segmentNumber(); // position of this record in segmentated series
   (TuncationReason) record.truncated();     // reason record was truncated; or else NOT_TRUNCATED
            (String) record.type();          // "warcinfo", "request", "response" etc
```

#### [Warcinfo](https://www.javadoc.io/page/org.netpreserve/jwarc/latest/org/netpreserve/jwarc/Warcinfo.html)

```java
            (Headers) warcinfo.fields();   // parses the body as application/warc-fields
   (Optional<String>) warcinfo.filename(); // filename of the containing WARC
```

#### [WarcTargetRecord](https://www.javadoc.io/page/org.netpreserve/jwarc/latest/org/netpreserve/jwarc/WarcTargetRecord.html) (abstract)

Methods available on all WARC records except Warcinfo:

```java
     (Optional<String>) record.identifiedPayloadType(); // media type of payload identified by an independent check
               (String) record.target();                // captured URI as an unparsed string
                  (URI) record.targetURI();             // captured URI
(Optional<WarcPayload>) record.payload();               // payload
     (Optional<Digest>) record.payloadDigest();         // value of hash function applied to bytes of the payload
        (Optional<URI>) record.warcinfoID();            // ID of warcinfo record when stored separately
```

#### [WarcContinuation](https://www.javadoc.io/page/org.netpreserve/jwarc/latest/org/netpreserve/jwarc/WarcContinuation.html)

```java
             (String) continuation.segmentOriginId();    // record ID of first segment
   (Optional<String>) continuation.segmentTotalLength(); // (last only) total length of all segments
```

#### [WarcConversion](https://www.javadoc.io/page/org.netpreserve/jwarc/latest/org/netpreserve/jwarc/WarcConversion.html)

```java
      (Optional<URI>) conversion.refersTo();    // ID of record this one was converted from
```

#### [WarcCaptureRecord](https://www.javadoc.io/page/org.netpreserve/jwarc/latest/org/netpreserve/jwarc/WarcCaptureRecord.html) (abstract)

Methods available on metadata, request, resource and response records:

```java
          (List<URI>) capture.concurrentTo();   // other record IDs from the same capture event
 (Optional<InetAddr>) capture.ipAddress();      // IP address of the server
```

#### [WarcMetadata](https://www.javadoc.io/page/org.netpreserve/jwarc/latest/org/netpreserve/jwarc/WarcMetadata.html)

```java
            (Headers) metadata.fields();        // parses the body as application/warc-fields
```

#### [WarcRequest](https://www.javadoc.io/page/org.netpreserve/jwarc/latest/org/netpreserve/jwarc/WarcRequest.html)

```java
        (HttpRequest) request.http();           // parses the body as a HTTP request
        (BodyChannel) request.http().body();    // HTTP request body
            (Headers) request.http().headers(); // HTTP request headers
```

#### [WarcResource](https://www.javadoc.io/page/org.netpreserve/jwarc/latest/org/netpreserve/jwarc/WarcResource.html)

No methods are specific to resource records. See WarcRecord, WarcTargetRecord, WarcCaptureRecord above.

#### [WarcResponse](https://www.javadoc.io/page/org.netpreserve/jwarc/latest/org/netpreserve/jwarc/WarcResponse.html)

```java
       (HttpResponse) response.http();           // parses the body as a HTTP response
        (BodyChannel) response.http().body();    // HTTP response body
            (Headers) response.http().headers(); // HTTP response headers
```

#### [WarcRevisit](https://www.javadoc.io/page/org.netpreserve/jwarc/latest/org/netpreserve/jwarc/WarcResponse.html)

```java
       (HttpResponse) revisit.http();              // parses the body as a HTTP response
            (Headers) revisit.http().headers();    // HTTP response headers (note: revisits never have a payload!)
                (URI) revisit.profile()            // revisit profile (not modified or identical payload)
                (URI) revisit.refersTo();          // id of record this is a duplicate of
                (URI) revisit.refersToTargetURI(); // targetURI of the referred to record 
            (Instant) revisit.refersToDate();      // date of the referred to record  
```

Note: revisit records never have a payload so 

## Pros and Cons

### Advantages of jwarc

**Performance**
- ✅ **Fastest Java WARC library**: Up to 5-13x faster than alternatives for uncompressed WARCs
- ✅ **NIO-based architecture**: Minimal memory copying, efficient buffer management
- ✅ **Multi-threading support**: Parallel processing for validation and deduplication
- ✅ **Ragel FSM parser**: Highly optimized state machine for parsing speed

**Modern Design**
- ✅ **Type-safe API**: Strongly-typed classes for each WARC record type
- ✅ **Java 8+ features**: Streams, Optional, Instant, and modern idioms
- ✅ **Immutable records**: Thread-safe, predictable behavior
- ✅ **Builder pattern**: Clean, fluent API for record construction

**Feature Rich**
- ✅ **Multiple compression formats**: GZIP, Zstandard (zstd), and Brotli support
- ✅ **Protocol support**: HTTP/1.x, HTTPS, Gemini protocol
- ✅ **Comprehensive validation**: Built-in WARC 1.1 header validation
- ✅ **Filter expressions**: DSL for powerful record filtering
- ✅ **Command-line tools**: 15+ tools for WARC manipulation
- ✅ **ARC format support**: Automatic detection and parsing

**Developer Experience**
- ✅ **Well documented**: Extensive JavaDoc, guides, and examples
- ✅ **Active maintenance**: Regular updates and bug fixes
- ✅ **Apache 2.0 license**: Permissive open-source license
- ✅ **Maven Central**: Easy integration into projects

### Limitations

**Format Support**
- ❌ **No ARC writing**: Cannot create ARC format files (WARC is preferred)
- ⚠️ **Brotli read-only**: Brotli compression is decompression-only
- ⚠️ **Limited HTTP/2**: No HTTP/2 protocol support yet

**Compatibility**
- ⚠️ **Java 8 minimum**: Requires JDK 8 or later (modern for most, but not all environments)
- ⚠️ **Strict parsing by default**: May reject some non-compliant WARCs (lenient mode available)

**Advanced Features**
- ⚠️ **No in-place editing**: Cannot modify existing WARC records in place
- ⚠️ **Limited async I/O**: Not fully asynchronous (future enhancement)
- ⚠️ **No built-in indexing**: Must generate indices separately

### When to Choose jwarc

**Best suited for:**
- High-performance WARC processing applications
- Modern Java projects (8+)
- Applications requiring strict WARC 1.1 compliance
- Projects needing modern compression (zstd)
- Command-line WARC manipulation and analysis
- Web archiving systems requiring validation

**Consider alternatives when:**
- You need ARC format writing capability
- Working with legacy Java environments (< Java 8)
- You require HTTP/2 protocol support
- In-place WARC editing is essential

### Migration from Other Libraries

If you're migrating from **JWAT** or **webarchive-commons**, jwarc offers:
- Significant performance improvements (1.4x - 13x faster)
- Type-safe API reducing errors
- Better documentation and examples
- Active maintenance and modern features

See the [Comparison](#comparison) section below for detailed feature comparison.

## Comparison

| Criteria            | jwarc      | [JWAT]          | [webarchive-commons] |
|---------------------|------------|-----------------|----------------------|
| License             | Apache 2   | Apache 2        | Apache 2             |
| Parser based on     | Ragel FSM  | Hand-rolled FSM | Apache HTTP          |
| Push parsing        | Low level  | ✘               | ✘                    |
| Folded headers †    | ✔          | ✔               | ✔                    | 
| [Encoded words] †   | ✘          | ✘ (disabled)    | ✘                    |
| Validation          | Comprehensive | ✔            | ✘                    |
| Strict parsing  ‡   | ✔          | ✘               | ✘                    |
| Lenient parsing     | HTTP only  | ✔               | ✔                    |
| Multi-value headers | ✔          | ✔               | ✘                    |
| I/O Framework       | NIO        | IO              | IO                   |
| Record type classes | ✔          | ✘               | ✘                    |
| Typed accessors     | ✔          | ✔               | Some                 |
| GZIP detection      | ✔          | ✔               | Filename only        |
| [zstd compression]  | Read/Write | ✘               | ✘                    |
| brotli compression  | Read-only  | ✘               | ✘                    |
| Gemini protocol     | ✔          | ✘               | ✘                    |
| WARC writer         | ✔          | ✔               | ✔                    |
| ARC reader          | Auto       | Separate API    | Factory              |
| ARC writer          | ✘          | ✔               | ✔                    |
| Header validation   | ✔          | Limited         | ✘                    |
| Multi-threading     | ✔          | Limited         | ✘                    |
| Speed * (.warc)     | 1x         | ~5x slower      | ~13x slower          |
| Speed * (.warc.gz)  | 1x         | ~1.4x slower    | ~2.8x slower         |

(†) WARC features copied from HTTP that have since been deprecated in HTTP. I'm not aware of any software that writes
WARCs using these features and usage of them should probably be avoided. JWAT behaves differently from jwarc and
webarchive-commons as it does not trim whitespace on folded lines.

(‡) JWAT and webarchive-commons both accept arbitrary UTF-8 characters in field names. jwarc strictly enforces the
grammar rules from the WARC specification, although it does not currently enforce the rules for the values of specific
individual fields.

(*) Relative time to scan records after JIT steady state. Only indicative. Need to redo this with a better
benchmark. JWAT was configured with a 8192 byte buffer as with default options it is 27x slower. For comparison
merely decompressing the .warc.gz file with GZIPInputStream is about 0.95x.

See also: [Unaffiliated benchmark against other languages](https://code402.com/hello-warc-common-crawl-code-samples)

[More recent benchmarks against Java libraries](https://github.com/iipc/jwarc/pull/19)

[JWAT]: https://sbforge.org/display/JWAT/JWAT
[webarchive-commons]: https://github.com/iipc/webarchive-commons
[Encoded words]: https://www.ietf.org/rfc/rfc2047.txt
[zstd compression]: https://iipc.github.io/warc-specifications/specifications/warc-zstd/

### Other WARC libraries

* [go-warc](https://github.com/wolfgangmeyers/go-warc) (Go)
* [node-warc](https://www.npmjs.com/package/node-warc) (Node.js)
* [warc](https://github.com/datatogether/warc) (Go)
* [warc](https://github.com/internetarchive/warc) (Python)
* [warc-clojure](https://github.com/shriphani/warc-clojure) (Clojure) - JWAT wrapper
* [warc-hadoop](https://github.com/ept/warc-hadoop) (Java)
* [warcat](https://github.com/chfoo/warcat) (Python)
* [warcio](https://github.com/webrecorder/warcio) (Python)
* [warctools](https://github.com/internetarchive/warctools) (Python)
* [webarchive](https://github.com/richardlehane/webarchive) (Go)
