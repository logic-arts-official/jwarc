# Contributing to jwarc

Thank you for your interest in contributing to jwarc! This document provides guidelines and instructions for contributing to the project.

## Getting Started

### Prerequisites

- **JDK 8 or later** (Java 25 recommended for new projects)
- **Apache Maven** 3.6+
- **Git**

### Building the Project

1. Clone the repository:
   ```bash
   git clone https://github.com/iipc/jwarc.git
   cd jwarc
   ```

2. Build the project:
   ```bash
   mvn clean package
   ```

3. Run tests:
   ```bash
   mvn test
   ```

### Project Structure

```
jwarc/
├── src/                           # Source code
│   └── org/netpreserve/jwarc/
│       ├── *.java                 # Core WARC library classes
│       ├── cdx/                   # CDX format support
│       ├── net/                   # Network services
│       └── tools/                 # Command-line tools
├── test/                          # Test code
├── resources/                     # Resource files
├── test-resources/               # Test resource files
├── README.md                      # Main documentation
├── CHANGELOG.md                   # Version history
└── pom.xml                        # Maven configuration
```

## Development Guidelines

### Code Style

- Follow standard Java naming conventions
- Use 4 spaces for indentation (no tabs)
- Keep line length reasonable (preferably under 120 characters)
- Write clear, self-documenting code with meaningful variable names
- Add comments for complex logic

### Adding New Features

1. **Create a new branch** for your feature:
   ```bash
   git checkout -b feature/your-feature-name
   ```

2. **Write tests** for your feature in the `test/` directory

3. **Update documentation** including:
   - JavaDoc comments for public APIs
   - README.md if adding user-facing features
   - CHANGELOG.md under "Unreleased" section

4. **Ensure all tests pass**:
   ```bash
   mvn test
   ```

5. **Build the project**:
   ```bash
   mvn clean package
   ```

### Ragel Parsers

Some parsers in this project are generated from Ragel grammar files (*.rl). If you modify a `.rl` file:

1. Install [Ragel](http://www.colm.net/open-source/ragel/)
2. Regenerate the Java file:
   ```bash
   ragel -J src/org/netpreserve/jwarc/YourParser.rl
   ```

### Adding Command-Line Tools

New command-line tools should:

1. Extend or be registered with `WarcTool.java`
2. Follow the naming convention `*Tool.java` in `src/org/netpreserve/jwarc/tools/`
3. Include a `--help` option showing usage
4. Be documented in README.md under "Command-line tools" section
5. Include unit tests in `test/org/netpreserve/jwarc/tools/`

## Pull Request Process

1. **Update documentation** for any user-facing changes
2. **Add tests** for new functionality
3. **Ensure the build passes**: `mvn clean test`
4. **Update CHANGELOG.md** with a note about your changes
5. **Submit a pull request** with a clear description of:
   - What problem does it solve?
   - What changes were made?
   - Are there any breaking changes?

### Pull Request Checklist

- [ ] Code builds successfully (`mvn clean package`)
- [ ] All tests pass (`mvn test`)
- [ ] New tests added for new functionality
- [ ] Documentation updated (README, JavaDoc, etc.)
- [ ] CHANGELOG.md updated
- [ ] Code follows project style guidelines

## Testing

### Running Tests

```bash
# Run all tests
mvn test

# Run a specific test class
mvn test -Dtest=WarcReaderTest

# Run with verbose output
mvn test -X
```

### Writing Tests

- Use JUnit 4 framework
- Place tests in `test/` directory mirroring the `src/` structure
- Test files should follow the pattern `*Test.java`
- Use descriptive test method names that explain what is being tested

Example:
```java
@Test
public void testParsingValidWarcRecord() throws Exception {
    // Test implementation
}
```

## Reporting Issues

When reporting issues, please include:

- **jwarc version** (from `java -jar jwarc.jar version`)
- **Java version** (from `java -version`)
- **Operating system**
- **Steps to reproduce** the issue
- **Expected behavior** vs **actual behavior**
- **Stack traces** or error messages (if applicable)
- **Sample WARC files** (if relevant and permissible)

## License

By contributing to jwarc, you agree that your contributions will be licensed under the Apache License 2.0.

## Questions?

If you have questions about contributing, feel free to:
- Open an issue for discussion
- Check existing issues and pull requests for similar topics
- Review the README.md for general project information

Thank you for contributing to jwarc!
