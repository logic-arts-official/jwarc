# jwarc API Quick Reference

A concise guide to commonly used jwarc APIs. For complete documentation see the [JavaDoc](https://www.javadoc.io/doc/org.netpreserve/jwarc).

## Reading WARC Files

### WarcReader

```java
// Open a WARC file
WarcReader reader = new WarcReader(FileChannel.open(Paths.get("file.warc")));
WarcReader reader = new WarcReader(Paths.get("file.warc"));

// Iterate over records
for (WarcRecord record : reader) {
    // Process record
}

// Use as stream
reader.records()
    .filter(r -> r instanceof WarcResponse)
    .forEach(r -> process(r));

// Close when done
reader.close();

// Enable lenient parsing
reader.setLenient(true);

// Register custom record type
reader.registerType("myrecord", MyRecord::new);
```

### WarcRecord - Common Methods

```java
record.type()                 // "request", "response", "resource", etc.
record.id()                   // URI - globally unique identifier
record.date()                 // Instant - capture time
record.contentType()          // MediaType - body content type
record.body()                 // MessageBody - record body
record.headers()              // MessageHeaders - all WARC headers

// Optional fields
record.blockDigest()          // Optional<Digest> - hash of body
record.segmentNumber()        // Optional<Long> - segment position
record.truncated()            // TruncationReason - why truncated
```

### WarcTargetRecord - Records with URIs

```java
// Available on: request, response, resource, metadata, revisit, conversion, continuation
record.target()               // String - captured URI (unparsed)
record.targetURI()            // URI - captured URI (parsed)
record.payload()              // Optional<WarcPayload> - payload
record.payloadDigest()        // Optional<Digest> - hash of payload
record.identifiedPayloadType() // Optional<String> - independently identified type
record.warcinfoID()           // Optional<URI> - associated warcinfo
```

### WarcResponse / WarcRequest

```java
WarcResponse response = (WarcResponse) record;

// Parse HTTP from body
HttpResponse http = response.http();
http.status()                 // int - HTTP status code (200, 404, etc.)
http.reason()                 // String - reason phrase ("OK", "Not Found")
http.headers()                // MessageHeaders - HTTP headers
http.body()                   // MessageBody - HTTP body

// WARC fields
response.concurrentTo()       // List<URI> - concurrent record IDs
response.ipAddress()          // Optional<InetAddress> - server IP
```

### Warcinfo

```java
Warcinfo info = (Warcinfo) record;
Headers fields = info.fields(); // Parse as application/warc-fields
String software = fields.first("software").orElse("unknown");
```

### WarcRevisit

```java
WarcRevisit revisit = (WarcRevisit) record;
revisit.profile()             // URI - revisit profile
revisit.refersTo()            // URI - ID of duplicate record
revisit.refersToTargetURI()   // URI - target of duplicate
revisit.refersToDate()        // Instant - date of duplicate
```

## Writing WARC Files

### WarcWriter

```java
// Open for writing
WarcWriter writer = new WarcWriter(FileChannel.open(path, WRITE, CREATE));
WarcWriter writer = new WarcWriter(System.out); // Write to stdout

// With compression
WarcWriter writer = new WarcWriter(channel, WarcCompression.GZIP);

// Write records
writer.write(record);

// Fetch a URL
writer.fetch(URI.create("http://example.org/"));

// Get current position
long pos = writer.position();

// Close
writer.close();
```

### Building Records

```java
// Warcinfo
Warcinfo info = new Warcinfo.Builder()
    .fields("software", "my-crawler/1.0",
            "format", "WARC/1.1")
    .build();

// Request
WarcRequest request = new WarcRequest.Builder()
    .target("http://example.org/")
    .date(Instant.now())
    .body(httpRequestBytes)
    .contentType("application/http")
    .build();

// Response
WarcResponse response = new WarcResponse.Builder()
    .target("http://example.org/")
    .date(Instant.now())
    .body(httpResponseBytes)
    .contentType("application/http")
    .concurrentTo(request.id())
    .ipAddress(InetAddress.getByName("93.184.216.34"))
    .build();

// Resource (standalone, no HTTP wrapper)
WarcResource resource = new WarcResource.Builder()
    .target("http://example.org/image.jpg")
    .date(Instant.now())
    .body(imageBytes)
    .contentType("image/jpeg")
    .build();

// Metadata
WarcMetadata metadata = new WarcMetadata.Builder()
    .target("http://example.org/")
    .date(Instant.now())
    .fields("fetchTimeMs", "245",
            "outlinks", "15")
    .build();
```

## HTTP Parsing

### HttpRequest

```java
WarcRequest warcRequest = (WarcRequest) record;
HttpRequest http = warcRequest.http();

http.method()                 // String - "GET", "POST", etc.
http.target()                 // String - request target
http.headers()                // MessageHeaders - HTTP headers
http.body()                   // MessageBody - request body
```

### HttpResponse

```java
WarcResponse warcResponse = (WarcResponse) record;
HttpResponse http = warcResponse.http();

http.status()                 // int - status code
http.reason()                 // String - reason phrase
http.headers()                // MessageHeaders
http.body()                   // MessageBody - decoded payload
```

## Working with Headers

### MessageHeaders

```java
Headers headers = record.headers();

// Get first value
Optional<String> value = headers.first("Content-Type");

// Get sole value (error if multiple)
Optional<String> location = headers.sole("Location");

// Get all values
List<String> cookies = headers.all("Set-Cookie");

// Check if contains value
boolean hasGzip = headers.contains("Content-Encoding", "gzip");

// As map
Map<String, List<String>> map = headers.map();
```

## Working with Bodies

### MessageBody

```java
MessageBody body = record.body();

body.size()                   // long - body length in bytes
body.position()               // long - current position
body.read(buffer)             // int - read into ByteBuffer
body.stream()                 // InputStream - as input stream

// Consume entire body
byte[] bytes = IOUtils.readNBytes(body.stream(), (int) body.size());
```

## Filtering

### WarcFilter

```java
// Compile a filter
WarcFilter filter = WarcFilter.compile(":status >= 400");

// Use as Predicate
reader.records()
    .filter(filter)
    .forEach(this::process);

// Filter expressions
":status == 200"              // HTTP status is 200
"warc-type == 'response'"     // WARC type
"http:content-type =~ 'text/html'" // HTTP header regex match
"content-length > 1000"       // Header numeric comparison
"warc-date > '2023-01-01'"    // Date comparison
"!(warc-type == 'metadata')"  // Boolean not

// Combine filters
"warc-type == 'response' && :status == 200"
"warc-type == 'response' || warc-type == 'resource'"
```

## Validation

### HeaderValidator

```java
// Use standard WARC 1.1 rules
HeaderValidator validator = HeaderValidator.standard();

// Validate a record
List<String> errors = validator.validate(record);
if (!errors.isEmpty()) {
    for (String error : errors) {
        System.err.println(error);
    }
}

// Forbid extension fields
validator.forbidUnknownFields();

// Add custom rules
validator.field("My-Custom-Field")
    .pattern(Pattern.compile("[A-Z]+"))
    .requireOn("response");
```

## Media Types

### MediaType

```java
MediaType type = MediaType.parse("text/html; charset=utf-8");

type.base()                   // MediaType - "text/html"
type.type()                   // String - "text"
type.subtype()                // String - "html"
type.parameters()             // Map<String,String> - {"charset": "utf-8"}

// Predefined types
MediaType.HTTP
MediaType.HTML
MediaType.WARC_FIELDS
MediaType.OCTET_STREAM
```

## Compression

### Detecting Compression

```java
WarcReader reader = new WarcReader(path);
WarcCompression compression = reader.compression();

if (compression == WarcCompression.GZIP) {
    // Processing gzipped WARC
}
```

### Writing Compressed

```java
// GZIP compression
WarcWriter writer = new WarcWriter(channel, WarcCompression.GZIP);

// Zstandard compression  
WarcWriter writer = new WarcWriter(channel, WarcCompression.ZSTD);
```

## Gemini Protocol

### GeminiResponse

```java
GeminiResponse gemini = GeminiResponse.parse(channel, buffer);

gemini.status()               // int - Gemini status (20, 30, 40, etc.)
gemini.statusHttpEquivalent() // int - HTTP equivalent (200, 307, 404, etc.)
gemini.meta()                 // String - meta info/MIME type
gemini.contentType()          // MediaType - parsed content type
gemini.body()                 // MessageBody - response body
```

## URI Utilities

### URIs

```java
import org.netpreserve.jwarc.URIs;

// Normalize
URI normalized = URIs.normalize(uri);

// Percent decode
String decoded = URIs.percentDecode("hello%20world");

// Parse with error recovery
URI parsed = URIs.parseLeniently("http://example.org");
```

## CDX Format

### Writing CDX

```java
import org.netpreserve.jwarc.cdx.*;

CdxWriter writer = new CdxWriter(System.out, " ");

for (WarcRecord record : reader) {
    if (record instanceof WarcCaptureRecord) {
        WarcCaptureRecord capture = (WarcCaptureRecord) record;
        writer.write(capture, position);
    }
    position += record.totalLength();
}
```

## Common Patterns

### Process All HTML Responses

```java
try (WarcReader reader = new WarcReader(path)) {
    for (WarcRecord record : reader) {
        if (record instanceof WarcResponse) {
            WarcResponse response = (WarcResponse) record;
            if (response.contentType().base().equals(MediaType.HTTP)) {
                HttpResponse http = response.http();
                if (http.status() == 200 && 
                    http.contentType().type().equals("text") &&
                    http.contentType().subtype().equals("html")) {
                    processHtml(http.body().stream());
                }
            }
        }
    }
}
```

### Create WARC from Crawl

```java
try (WarcWriter writer = new WarcWriter(path, WarcCompression.GZIP)) {
    // Write warcinfo
    writer.write(new Warcinfo.Builder()
        .fields("software", "my-crawler/1.0")
        .build());
    
    // Crawl and write
    for (String url : urls) {
        HttpURLConnection conn = (HttpURLConnection) 
            new URL(url).openConnection();
        
        // Build and write request
        WarcRequest request = buildRequest(conn);
        writer.write(request);
        
        // Build and write response
        WarcResponse response = buildResponse(conn, request);
        writer.write(response);
    }
}
```

### Extract Payloads

```java
try (WarcReader reader = new WarcReader(path)) {
    for (WarcRecord record : reader) {
        if (record instanceof WarcResponse) {
            WarcResponse response = (WarcResponse) record;
            Optional<WarcPayload> payload = response.payload();
            if (payload.isPresent()) {
                // Save payload
                Files.copy(payload.get().body().stream(),
                    Paths.get("payloads", record.id().toString()));
            }
        }
    }
}
```

## Error Handling

### ParsingException

```java
try {
    WarcReader reader = new WarcReader(path);
    for (WarcRecord record : reader) {
        // Process
    }
} catch (ParsingException e) {
    System.err.println("Parse error at offset " + e.getPosition());
    System.err.println(e.getMessage());
}
```

### Lenient Mode

```java
WarcReader reader = new WarcReader(path);
reader.setLenient(true); // Be forgiving of errors

for (WarcRecord record : reader) {
    try {
        // Process record
    } catch (Exception e) {
        // Handle per-record errors
        continue;
    }
}
```

## Performance Tips

1. **Reuse buffers**: When processing many files, reuse ByteBuffer instances
2. **Stream processing**: Don't load entire files into memory
3. **Filter early**: Apply WarcFilter before expensive operations
4. **Parallel processing**: Use Java streams with `.parallel()` for CPU-bound work
5. **Compression**: GZIP is standard, ZSTD offers better compression ratio
6. **Direct buffers**: Use direct ByteBuffers for large files (see NIO docs)

## Further Reading

- [Full JavaDoc](https://www.javadoc.io/doc/org.netpreserve/jwarc)
- [WARC Specification](https://iipc.github.io/warc-specifications/)
- [Project README](README.md)
- [Architecture Guide](ARCHITECTURE.md)
- [Contributing Guide](CONTRIBUTING.md)
