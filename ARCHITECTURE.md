# jwarc Architecture

This document provides an overview of jwarc's architecture, design decisions, and implementation details.

## Overview

jwarc is a high-performance Java library for reading and writing WARC (Web ARChive) files. It's designed around several key principles:

1. **Performance**: Use of NIO, minimal copying, and shared buffers
2. **Type Safety**: Strongly-typed API with specific classes for each record type
3. **Extensibility**: Support for custom record types and header fields
4. **Standards Compliance**: Strict parsing with optional lenient mode

## Core Components

### Message Hierarchy

The library uses an inheritance hierarchy to model messages:

```
Message (abstract)
├── HttpMessage (abstract)
│   ├── HttpRequest
│   └── HttpResponse
└── WarcRecord (abstract)
    ├── Warcinfo
    └── WarcTargetRecord (abstract)
        ├── WarcContinuation
        ├── WarcConversion
        └── WarcCaptureRecord (abstract)
            ├── WarcMetadata
            ├── WarcRequest
            ├── WarcResource
            ├── WarcResponse
            └── WarcRevisit
```

**Message**: Base class for both HTTP and WARC messages, providing common functionality for headers and body access.

**WarcRecord**: Base class for all WARC record types with common fields like date, record-id, content-type, etc.

**WarcTargetRecord**: Records that have a target URI (all except warcinfo).

**WarcCaptureRecord**: Records created during web capture (request, response, resource, metadata, revisit).

### Parsing

#### Ragel-Based Parsers

jwarc uses [Ragel](http://www.colm.net/open-source/ragel/) to generate high-performance finite state machine parsers:

- **WarcParser.rl** → WarcParser.java: Parses WARC headers
- **HttpParser.rl** → HttpParser.java: Parses HTTP requests/responses
- **GeminiParser.rl** → GeminiParser.java: Parses Gemini protocol messages
- **MediaType.rl** → MediaType.java: Parses media type strings
- **ChunkedBody.rl** → ChunkedBody.java: Parses HTTP chunked transfer encoding

The Ragel grammars define the exact syntax rules from the specifications. This approach provides:
- **Speed**: Compiled FSMs are very fast
- **Correctness**: Grammar explicitly matches the specification
- **Maintainability**: Grammar is easier to understand than hand-rolled code

#### Lenient Parsing

While strict by default, jwarc supports lenient parsing modes:

- **WARC**: Optional lenient mode for handling malformed records
- **HTTP**: Lenient by default, handles common deviations from spec
- **ARC**: Lenient by default, treats ARC as a WARC dialect

### I/O Architecture

#### NIO-Based Design

jwarc uses Java NIO (New I/O) throughout:

```
ReadableByteChannel
    ↓
ByteBuffer (reusable)
    ↓
Parser (FSM)
    ↓
Message Objects
```

Benefits:
- **Zero-copy**: Data can be read directly into native memory
- **Efficiency**: Minimize object allocation and GC pressure
- **Flexibility**: Support for files, sockets, pipes, memory-mapped files

#### Body Handling

Message bodies are abstracted through the `MessageBody` interface:

- **LengthedBody**: Body with known length (most common case)
- **ChunkedBody**: HTTP chunked transfer encoding
- **DecodedBody**: Transparently decodes content-encoding (gzip, deflate, brotli)

Bodies are read on-demand, not eagerly loaded into memory.

#### Compression Support

Automatic decompression for:
- **GZIP**: Built-in Java support
- **Zstandard (zstd)**: Via `com.github.luben:zstd-jni`
- **Brotli**: Via optional `org.brotli:dec` dependency

Compression is detected automatically from:
1. WARC-Block-Digest field
2. Magic bytes in the stream
3. File extension (.warc.gz)

### Filtering

The `WarcFilter` class implements a domain-specific language for filtering records:

```
WarcFilter
    ↓
WarcFilterCompiler
    ↓
Expression Tree
    ↓
Predicate<WarcRecord>
```

Filter expressions support:
- Comparison operators: `==`, `!=`, `<`, `>`, `<=`, `>=`
- Pattern matching: `=~` (regex)
- Boolean operators: `&&`, `||`, `!`
- Header field access: `warc-type`, `http:status`, etc.
- Special accessors: `:status` (HTTP status code)

### Validation

The `HeaderValidator` class provides configurable validation:

- **Field-level rules**: Pattern matching, repeatability, allowed record types
- **Record-level rules**: Mandatory fields per record type
- **Extensibility**: Rules can be added/modified programmatically

Built-in WARC 1.1 ruleset enforces:
- Required fields per record type
- Valid values (URIs, dates, digests, etc.)
- Proper header syntax

## Protocol Support

### HTTP/HTTPS

Full HTTP/1.0 and HTTP/1.1 support:
- Request/response parsing
- Chunked transfer encoding
- Multiple content encodings
- Custom and extension headers
- Lenient handling of common deviations

### Gemini

Lightweight support for the Gemini protocol:
- Parse Gemini requests (just a URL)
- Parse Gemini responses (status + meta + body)
- Map Gemini status codes to HTTP equivalents
- Support text/gemini content type

### ARC Format

Transparent ARC support:
- ARC records parsed as WARC dialect
- Automatic field mapping (ARC → WARC headers)
- Lenient parsing for older ARC files
- No ARC writing support (WARC is preferred)

## Command-Line Tools

Tools are organized under `org.netpreserve.jwarc.tools`:

```
WarcTool (main entry point)
├── CdxTool
├── DedupeTool
├── ExtractTool
├── FetchTool
├── FilterTool
├── ListTool
├── RecordTool
├── RecorderTool
├── SavebackTool
├── ScreenshotTool
├── ServeTool
├── StatsTool
└── ValidateTool
```

Each tool:
- Accepts command-line arguments
- Provides `--help` option
- Processes WARC files efficiently
- Handles errors gracefully

## Design Patterns

### Builder Pattern

Record creation uses the builder pattern:

```java
WarcResponse response = new WarcResponse.Builder()
    .date(Instant.now())
    .target(uri)
    .contentType("application/http")
    .body(bodyBytes)
    .build();
```

Benefits:
- Immutable records
- Clear, readable construction
- Optional fields handled elegantly
- Validation at build time

### Flyweight Pattern

Headers use a flyweight-like pattern:
- Field names are case-insensitive but preserve original case
- Multiple values for same field stored efficiently
- Lazy parsing of structured values

### Strategy Pattern

Compression/decompression uses strategy pattern:
- `WarcCompression`: Enum of compression strategies
- Each strategy knows how to wrap channels
- New compression types easily added

## Performance Considerations

### Memory Efficiency

- **Streaming**: Records processed one at a time
- **Buffer reuse**: ByteBuffers reused across records
- **Lazy parsing**: Headers/bodies parsed on demand
- **No unnecessary copies**: Data stays in buffers when possible

### Speed Optimizations

- **FSM parsers**: Ragel-generated parsers are extremely fast
- **NIO**: Non-blocking I/O where beneficial
- **Multi-threading**: Tools support parallel processing
- **Minimal allocation**: Reuse objects where possible

### Benchmarks

Relative performance (from README comparison):
- Uncompressed WARC: ~5-13x faster than alternatives
- Compressed WARC: ~1.4-2.8x faster than alternatives

## Extensibility

### Custom Record Types

Register custom record types:

```java
reader.registerType("myrecord", MyRecord::new);
```

### Custom Header Fields

Add typed accessors for extension headers:

```java
public class MyRecord extends WarcRecord {
    public Optional<String> myField() {
        return headers().first("My-Custom-Field");
    }
}
```

### Custom Validation Rules

Extend header validation:

```java
HeaderValidator validator = HeaderValidator.standard();
validator.field("My-Field")
    .pattern(Pattern.compile("[A-Z]+"))
    .requireOn("myrecord");
```

## Threading Model

jwarc is designed to be thread-safe in normal usage:

- **Readers**: Not thread-safe, use one per thread
- **Writers**: Not thread-safe, use one per thread
- **Records**: Immutable once built, safe to share
- **Tools**: Support parallel processing with `--threads` option

## Dependencies

### Required

- Java 8+ (Java 25 recommended for new development)
- `com.github.luben:zstd-jni` - Zstandard compression

### Optional

- `org.brotli:dec` - Brotli decompression

### Build/Test

- JUnit 4 - Unit testing framework
- Maven - Build tool

## Future Considerations

Potential areas for enhancement:

1. **WARC 1.1 Features**: Full segment support
2. **ARC Writing**: Support writing ARC format
3. **HTTP/2**: Add HTTP/2 protocol support
4. **Performance**: Further optimization of hot paths
5. **Async I/O**: Fully async processing pipeline
6. **WARC Editing**: In-place record modification

## References

- [WARC Specification](https://iipc.github.io/warc-specifications/)
- [HTTP/1.1 RFC 7230-7235](https://tools.ietf.org/html/rfc7230)
- [Gemini Protocol](https://gemini.circumlunar.space/)
- [Ragel State Machine Compiler](http://www.colm.net/open-source/ragel/)
