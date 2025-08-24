# Implementation Plan

- [ ] 1. Set up project structure and core data models
  - Initialize Python 3.13 project with uv using `uv init`
  - Create pyproject.toml with project metadata and dependencies
  - Define data classes for CrawlResult, UrlInfo, FailedUrl, and CrawlerConfig
  - Create directory structure and __init__.py files for the web crawler package
  - _Requirements: 1.1, 4.3_

- [ ] 2. Implement URL management and validation components
  - [ ] 2.1 Create DomainValidator class with domain checking logic
    - Implement is_same_domain method to check if URLs belong to same domain
    - Add extract_domain method to parse domain from URLs
    - Write unit tests for domain validation logic
    - _Requirements: 2.1, 2.2, 2.3_

  - [ ] 2.2 Implement URLManager class for queue and visited URL tracking
    - Create URL queue management with depth tracking
    - Implement visited URLs set to prevent duplicates
    - Add methods for adding, retrieving, and checking URLs
    - Write unit tests for URL management operations
    - _Requirements: 3.1, 3.2, 3.3_

- [ ] 3. Create HTTP client with error handling and retry logic
  - [ ] 3.1 Implement HTTPClient class with basic request functionality
    - Create fetch_page method with timeout and user-agent headers
    - Add response validation and status code handling
    - Write unit tests for basic HTTP functionality
    - _Requirements: 5.1, 5.3_

  - [ ] 3.2 Add retry logic and rate limiting to HTTPClient
    - Implement exponential backoff retry mechanism
    - Add configurable delays between requests
    - Handle network timeouts and connection errors gracefully
    - Write unit tests for retry and rate limiting behavior
    - _Requirements: 5.3, 5.4_

- [ ] 4. Implement HTML parsing and link extraction
  - Create LinkExtractor class with BeautifulSoup-based HTML parsing
  - Implement extract_links method to find all href attributes
  - Add URL normalization and relative URL resolution
  - Filter out non-HTTP schemes (mailto, javascript, etc.)
  - Write unit tests for link extraction and URL normalization
  - _Requirements: 1.2, 5.2_

- [ ] 5. Create configuration management system
  - Implement ConfigManager class for command-line argument parsing
  - Define default configuration values and validation
  - Add support for max_depth, delay, timeout, and output_format options
  - Write unit tests for configuration parsing and validation
  - _Requirements: 6.1, 6.2, 6.3, 6.4_

- [ ] 6. Implement output formatting and display
  - [ ] 6.1 Create OutputFormatter class with text output support
    - Implement hierarchical tree structure display for URLs
    - Add progress reporting during crawling process
    - Create summary statistics display
    - Write unit tests for text formatting
    - _Requirements: 4.1, 4.2, 4.3_

  - [ ] 6.2 Add JSON and CSV output format support
    - Implement JSON serialization for CrawlResult data
    - Add CSV export functionality with proper column headers
    - Write unit tests for different output formats
    - _Requirements: 6.4_

- [ ] 7. Create main crawler controller and orchestration
  - [ ] 7.1 Implement WebLinkCrawler main class
    - Create crawl method that orchestrates the crawling process
    - Implement main crawling loop with queue processing
    - Add depth limit checking and URL processing coordination
    - Write unit tests for crawler orchestration logic
    - _Requirements: 1.1, 1.3, 1.4, 3.3_

  - [ ] 7.2 Add error handling and graceful shutdown
    - Implement comprehensive error handling for all failure scenarios
    - Add graceful shutdown capability and result compilation
    - Handle edge cases like malformed HTML and invalid URLs
    - Write unit tests for error handling scenarios
    - _Requirements: 4.4, 5.1, 5.2_

- [ ] 8. Create command-line interface and entry point
  - Implement main CLI script with argument parsing
  - Add help text and usage examples
  - Configure entry point in pyproject.toml for uv execution
  - Create script that can be run with `uv run web-link-crawler`
  - Write integration tests for complete CLI workflow
  - _Requirements: 6.1_

- [ ] 9. Add comprehensive testing and validation
  - [ ] 9.1 Create integration tests with mock HTTP responses
    - Set up mock web server for testing crawling behavior
    - Test complete crawl workflow with various HTML structures
    - Validate depth limiting and domain boundary enforcement
    - _Requirements: 1.4, 2.4, 3.4_

  - [ ] 9.2 Add performance and edge case testing
    - Test memory usage with large URL sets
    - Validate behavior with circular references and infinite loops
    - Test error recovery and continuation after failures
    - _Requirements: 3.4, 5.1, 5.2, 5.3_

- [ ] 10. Create documentation and usage examples
  - Write comprehensive README with uv installation and usage instructions
  - Document how to run the crawler with `uv run` commands
  - Add code documentation and docstrings for all classes and methods
  - Create example scripts demonstrating different use cases with uv
  - _Requirements: 6.1, 6.4_