# Changelog

## Unreleased

### New Features

- **Gemini Protocol Support**: Added support for parsing Gemini protocol requests and responses
  - New classes: `GeminiRequest`, `GeminiResponse`, `GeminiParser`
  - Gemini status codes mapped to HTTP equivalents for compatibility
- **Brotli Compression**: Added support for Brotli decompression via `org.brotli:dec` (optional dependency)
- **Header Validation**: New `HeaderValidator` class for validating WARC headers against standard rules
- **Enhanced Tools**: Additional command-line tools for WARC manipulation
  - `ls` - List records in WARC files
  - `stats` - Print statistics about WARC and CDX files
  - `saveback` - Save wayback-style replayed pages as WARC records
- CDX improvements:
  - `CdxRecord`: added `surt()`, `format()`, `values()` and `toString()` methods
  - Better CDX format handling and serialization

### Improvements

- Enhanced error messages with source filename and record offset context
- Multi-threaded support for validation and deduplication operations
- Improved performance for large WARC file processing
- Better handling of edge cases in HTTP and WARC parsing

### Documentation

- **Comprehensive documentation updates**: All documentation files recreated and enhanced
- **New BACKLOG.md**: Detailed roadmap of future features and enhancements
- **Enhanced README.md**: Added Quick Developer Setup, Pros/Cons section, and Backlog reference
- **Improved CONTRIBUTING.md**: Added 5-minute quick start guide for contributors
- **Developer-friendly**: Better "check out and get up and running" instructions

### Dependencies

- Updated `com.github.luben:zstd-jni` from 1.5.7-4 to 1.5.7-6
- Updated `org.sonatype.central:central-publishing-maven-plugin` from 0.8.0 to 0.9.0

## 0.32.0

### Added

- HeaderValidator with WARC/1.1 standard ruleset
- ExtractTool: can now extract sequential concurrent records (`--concurrent` option)
- DedupeTool
  - In-memory cache for cross-URL digest-based deduplication (`--cache-size` option)
  - Now prints deduplication statistics (`--dry-run` and `--quiet` options)
  - Multi-threaded deduplication (`--threads` option)
- ValidateTool
  - Multi-threaded validation (`--threads` option) 
- ParsingException message is now annotated with the source filename and record offset when available

### Fixed

- RFC5952 canonical form is now used for IPv6 addresses in WARC-IP-Address
- HttpParser in lenient mode now:
  - accepts responses missing version number
  - ignores header lines missing :
  - ignores folded status lines
- WarcParser: treats `alexa/dat` ARC records as not HTTP type
