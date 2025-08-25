# Design Document

## Overview

The web link crawler is designed as a Python 3.13-based command-line tool using uv for dependency management and execution. The system performs breadth-first traversal of web pages within a specified domain, uses a queue-based approach to manage URLs to be crawled, maintains a set of visited URLs to prevent duplicates, and implements configurable limits and error handling for robust operation.

## Architecture

The system follows a modular architecture with clear separation of concerns:

```
WebLinkCrawler
├── URLManager (manages visited URLs and queue)
├── HTTPClient (handles web requests with retry logic)
├── LinkExtractor (parses HTML and extracts links)
├── DomainValidator (validates same-domain constraints)
├── OutputFormatter (formats and displays results)
└── ConfigManager (handles command-line arguments and settings)
```

## Components and Interfaces

### 1. WebLinkCrawler (Main Controller)
**Purpose:** Orchestrates the crawling process and coordinates between components.

**Key Methods:**
- `crawl(start_url: str) -> CrawlResult`
- `process_url(url: str, depth: int) -> List[str]`
- `run() -> None`

**Responsibilities:**
- Initialize components with configuration
- Manage the main crawling loop
- Coordinate between URL management and processing
- Handle graceful shutdown and result compilation

### 2. URLManager
**Purpose:** Manages the queue of URLs to crawl and tracks visited URLs.

**Key Methods:**
- `add_url(url: str, depth: int) -> bool`
- `get_next_url() -> Tuple[str, int]`
- `is_visited(url: str) -> bool`
- `mark_visited(url: str) -> None`
- `has_pending_urls() -> bool`

**Data Structures:**
- `visited_urls: Set[str]` - Normalized URLs that have been processed
- `url_queue: Queue[Tuple[str, int]]` - URLs to process with their depth
- `url_depth_map: Dict[str, int]` - Mapping of URLs to their discovery depth

### 3. HTTPClient
**Purpose:** Handles HTTP requests with proper error handling and retry logic.

**Key Methods:**
- `fetch_page(url: str) -> Optional[str]`
- `is_valid_response(response: requests.Response) -> bool`

**Features:**
- Configurable timeout settings
- Exponential backoff retry mechanism
- User-agent header configuration
- Rate limiting with configurable delays
- Proper handling of HTTP status codes

### 4. LinkExtractor
**Purpose:** Parses HTML content and extracts all href links.

**Key Methods:**
- `extract_links(html_content: str, base_url: str) -> List[str]`
- `normalize_url(url: str, base_url: str) -> str`
- `is_valid_link(url: str) -> bool`

**Features:**
- BeautifulSoup-based HTML parsing
- Relative URL resolution
- URL normalization (remove fragments, normalize paths)
- Filter out non-HTTP(S) schemes (mailto:, javascript:, etc.)

### 5. DomainValidator
**Purpose:** Validates whether URLs belong to the same domain and are within the allowed path scope of the starting URL.

**Key Methods:**
- `is_same_domain(url: str, base_domain: str) -> bool`
- `extract_domain(url: str) -> str`
- `is_within_path_scope(url: str, base_path: str) -> bool`
- `should_crawl(url: str) -> bool`

**Features:**
- Subdomain handling configuration
- Protocol normalization
- Port number handling
- Path hierarchy validation (only crawl URLs within or deeper than starting URL path)
- Path normalization and comparison

### 6. OutputFormatter
**Purpose:** Formats and displays crawling results in various formats.

**Key Methods:**
- `format_results(results: CrawlResult, format_type: str) -> str`
- `print_progress(current_url: str, depth: int, total_found: int) -> None`
- `print_summary(results: CrawlResult) -> None`

**Supported Formats:**
- Text (hierarchical tree structure)
- JSON (structured data)
- CSV (tabular format)

### 7. ConfigManager
**Purpose:** Handles command-line arguments and configuration settings.

**Configuration Options:**
- `max_depth: int` (default: 3)
- `delay: float` (default: 1.0 seconds)
- `timeout: int` (default: 10 seconds)
- `output_format: str` (default: "text")
- `include_external: bool` (default: False)
- `max_urls: int` (default: 1000)
- `verbose: bool` (default: False)

## Data Models

### CrawlResult
```python
@dataclass
class CrawlResult:
    start_url: str
    internal_urls: List[UrlInfo]
    external_urls: List[UrlInfo]
    failed_urls: List[FailedUrl]
    total_processed: int
    total_found: int
    crawl_duration: float
```

### UrlInfo
```python
@dataclass
class UrlInfo:
    url: str
    depth: int
    parent_url: Optional[str]
    status_code: Optional[int]
    discovered_at: datetime
```

### FailedUrl
```python
@dataclass
class FailedUrl:
    url: str
    error_message: str
    attempted_at: datetime
    parent_url: Optional[str]
```

### CrawlerConfig
```python
@dataclass
class CrawlerConfig:
    max_depth: int = 3
    delay: float = 1.0
    timeout: int = 10
    output_format: str = "text"
    include_external: bool = False
    max_urls: int = 1000
    verbose: bool = False
    user_agent: str = "WebLinkCrawler/1.0"
    base_url: str = ""  # Starting URL for path scope validation
    base_path: str = ""  # Extracted path from starting URL
```

## Error Handling

### HTTP Errors
- **Connection Errors:** Log and continue with next URL
- **Timeout Errors:** Retry with exponential backoff (max 3 attempts)
- **4xx Errors:** Log as failed URL, continue crawling
- **5xx Errors:** Retry with backoff, then log as failed

### Parsing Errors
- **Malformed HTML:** Use BeautifulSoup's lenient parsing
- **Invalid URLs:** Log warning and skip
- **Encoding Issues:** Attempt multiple encoding detection methods

### Resource Limits
- **Memory Limits:** Implement URL queue size limits
- **Time Limits:** Optional maximum crawl duration
- **URL Limits:** Configurable maximum number of URLs to process

## Testing Strategy

### Unit Tests
- **URLManager:** Test queue operations, duplicate detection, depth tracking
- **LinkExtractor:** Test HTML parsing, URL normalization, relative URL resolution
- **DomainValidator:** Test domain matching logic, subdomain handling
- **HTTPClient:** Test retry logic, timeout handling, response validation
- **OutputFormatter:** Test different output formats, result formatting

### Integration Tests
- **End-to-End Crawling:** Test complete crawl workflow with mock HTTP responses
- **Configuration Handling:** Test command-line argument parsing and validation
- **Error Scenarios:** Test behavior with various error conditions

### Test Data
- **Mock HTML Pages:** Create test HTML with various link structures
- **Domain Test Cases:** Test internal/external link classification
- **Error Response Mocking:** Simulate various HTTP error conditions

## Performance Considerations

### Memory Management
- Use generators where possible to reduce memory footprint
- Implement URL queue size limits to prevent memory exhaustion
- Clear processed data periodically for long-running crawls

### Network Efficiency
- Implement connection pooling for HTTP requests
- Use session objects to reuse connections
- Respect robots.txt (optional feature)
- Implement proper rate limiting

### Scalability
- Design for potential future async/await implementation
- Modular architecture allows for easy component replacement
- Configuration-driven behavior for different use cases

## Development Environment

### Python Version and Package Management
- **Python Version:** 3.13
- **Package Manager:** uv for fast dependency resolution and virtual environment management
- **Project Structure:** Standard Python package with pyproject.toml configuration
- **Dependencies:** Managed through uv with lock file for reproducible builds

### uv Configuration
- Use `uv init` to initialize the project
- Dependencies managed in `pyproject.toml`
- Virtual environment automatically managed by uv
- Fast installation and resolution of dependencies