# Testing Strategy for URL Shortener

This document describes the complete testing strategy for the production-grade URL Shortener service.

## Table of Contents

- [Testing Philosophy](#testing-philosophy)
- [Test Types](#test-types)
- [Running Tests](#running-tests)
- [Test Coverage](#test-coverage)
- [Failure Scenarios](#failure-scenarios)
- [Performance Testing](#performance-testing)
- [Continuous Integration](#continuous-integration)

## Testing Philosophy

### Principles

1. **Correctness First**: Every test must verify correct behavior
2. **Production Realism**: Tests should reflect production scenarios
3. **Fast Feedback**: Unit tests should run in seconds
4. **Isolation**: Tests should not depend on external services (except integration tests)
5. **Maintainability**: Tests should be readable and maintainable

### Test Pyramid

```
        /\
       /  \      E2E Tests (Few)
      /----\
     /      \    Integration Tests (Some)
    /--------\
   /          \  Unit Tests (Many)
  /------------\
```

- **Unit Tests**: Fast, isolated, test individual components
- **Integration Tests**: Test component interactions with real dependencies
- **E2E Tests**: Test complete user flows

## Test Types

### 1. Unit Tests

**Location**: `src/test/java/com/shortener/`

**Purpose**: Test individual components in isolation

**Key Test Classes**:

- `Base62EncoderTest`: Encoding/decoding logic
- `ShortCodeGeneratorTest`: Short code generation
- `UrlMappingTest`: Entity business logic
- `UrlShorteningServiceTest`: URL shortening service
- `UrlRedirectServiceTest`: Redirect service (CRITICAL)
- `RedisRateLimiterTest`: Rate limiting logic

**Characteristics**:
- Use mocks for dependencies
- Run in milliseconds
- No external dependencies
- High code coverage

### 2. Service Layer Tests

**Purpose**: Test service layer with mocked repositories

**Key Scenarios**:

1. **UrlShorteningService**:
   - ✅ Successful URL shortening
   - ✅ URL normalization (adds https://)
   - ✅ Custom alias handling
   - ✅ Custom alias conflicts
   - ✅ Database failures
   - ✅ Redis cache failures (should not fail operation)
   - ✅ Transaction rollback
   - ✅ Invalid URL input
   - ✅ Expiration date handling

2. **UrlRedirectService** (CRITICAL):
   - ✅ Cache hit → no DB call
   - ✅ Cache miss → DB fallback → cache populate
   - ✅ Expired URL → 410 Gone
   - ✅ Inactive URL → 410 Gone
   - ✅ Invalid format → 400 Bad Request
   - ✅ Redis down + DB up → succeeds
   - ✅ Redis up + DB down + cache hit → succeeds
   - ✅ Redis up + DB down + cache miss → 503
   - ✅ Redis increment failure → redirect still succeeds
   - ✅ Analytics failure → redirect still succeeds
   - ✅ No DB writes on redirect path

### 3. Integration Tests

**Location**: `src/test/java/com/shortener/integration/`

**Purpose**: Test with real PostgreSQL and Redis

**Technology**: Testcontainers

**Key Scenarios**:

1. Full shorten → redirect flow
2. Redis key creation and expiration
3. Click counter increments
4. Rate limiting end-to-end
5. Health endpoint verification

**Setup**:
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class UrlShortenerIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = ...;
    
    @Container
    static RedisContainer redis = ...;
}
```

### 4. Rate Limiting Tests

**Purpose**: Verify rate limiting behavior

**Key Scenarios**:

1. ✅ Allow requests under limit
2. ✅ Block requests over limit (429)
3. ✅ Sliding window behavior
4. ✅ Different limits for endpoints
5. ✅ Per-IP isolation
6. ✅ Redis down → fail-open
7. ✅ Retry-After header
8. ✅ Atomic behavior under concurrency

### 5. Failure & Chaos Testing

**Purpose**: Verify system behavior under failures

**Scenarios**:

1. **Redis Shutdown**:
   - System should fall back to database
   - Redirects should still work (slower)
   - Rate limiting should allow all requests

2. **Database Shutdown**:
   - Cached redirects should still work
   - New shorten requests should fail (503)
   - Cache misses should fail (503)

3. **High Concurrency**:
   - 1000+ concurrent redirects
   - Verify no data corruption
   - Verify performance degradation

4. **Memory Pressure**:
   - Large number of Redis keys
   - Verify TTL expiration works
   - Verify memory cleanup

### 6. Performance Tests

**Purpose**: Measure latency, throughput, and error rates

**Tools**:
- Apache Bench (`ab`)
- `wrk`
- Custom scripts

**Metrics**:
- Latency (p50, p95, p99)
- Throughput (requests/second)
- Error rate
- Cache hit rate

## Running Tests

### Unit Tests

```bash
# Run all unit tests
mvn test

# Run specific test class
mvn test -Dtest=Base62EncoderTest

# Run with coverage
mvn test jacoco:report
```

### Integration Tests

```bash
# Run integration tests (requires Docker)
mvn test -Dtest=*IntegrationTest

# Or use Testcontainers
mvn verify
```

### All Tests

```bash
# Run all tests
mvn clean test

# Skip integration tests
mvn test -DskipITs
```

## Test Coverage

### Target Coverage

- **Overall**: > 80%
- **Critical Paths**: > 95%
  - `UrlRedirectService`: 100%
  - `Base62Encoder`: 100%
  - `ShortCodeGenerator`: 100%

### Coverage Report

```bash
# Generate coverage report
mvn test jacoco:report

# View report
open target/site/jacoco/index.html
```

## Failure Scenarios

### Expected Behaviors

| Scenario | Expected Behavior | Test |
|----------|------------------|------|
| Redis down | Fall back to DB, allow redirects | ✅ |
| DB down + cache hit | Serve from cache | ✅ |
| DB down + cache miss | Return 503 | ✅ |
| Rate limit exceeded | Return 429 with Retry-After | ✅ |
| Invalid short code | Return 400 | ✅ |
| Expired URL | Return 410 Gone | ✅ |
| Inactive URL | Return 410 Gone | ✅ |
| Analytics failure | Log warning, continue | ✅ |
| Counter increment failure | Log warning, continue | ✅ |

### Failure Injection

**Using Testcontainers**:
```java
// Stop Redis container
redis.stop();

// Verify fallback behavior
// ...

// Restart Redis
redis.start();
```

**Using Chaos Engineering Tools**:
- Chaos Monkey
- Litmus
- Gremlin

## Performance Testing

### Load Test Scripts

**Location**: `scripts/`

1. **load-test.sh**: Apache Bench load testing
   ```bash
   ./scripts/load-test.sh
   ```

2. **wrk-load-test.lua**: wrk load testing
   ```bash
   wrk -t12 -c400 -d30s -s scripts/wrk-load-test.lua http://localhost:8080
   ```

3. **performance-test.sh**: Latency and throughput
   ```bash
   ./scripts/performance-test.sh
   ```

### Performance Targets

- **Redirect (cache hit)**: < 1ms p99
- **Redirect (cache miss)**: < 50ms p99
- **Shorten**: < 100ms p99
- **Throughput**: > 10,000 req/s (redirects)
- **Error rate**: < 0.1%

### Benchmarking

```bash
# Redirect endpoint
ab -n 10000 -c 100 http://localhost:8080/{shortCode}

# Shorten endpoint
ab -n 1000 -c 10 -p request.json -T application/json \
  http://localhost:8080/api/v1/urls/shorten
```

## Continuous Integration

### CI Pipeline

1. **Unit Tests**: Run on every commit
2. **Integration Tests**: Run on PR
3. **Performance Tests**: Run nightly
4. **Coverage Report**: Generate and publish

### GitHub Actions Example

```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-java@v2
      - run: mvn test
      - run: mvn jacoco:report
```

## Test Data Management

### Test Fixtures

- Use builders for test data
- Keep test data minimal
- Use factories for complex objects

### Database State

- Use `@Transactional` for test isolation
- Clean up after tests
- Use Testcontainers for integration tests

## Best Practices

1. **Test Naming**: Use descriptive names
   ```java
   @Test
   @DisplayName("Cache hit should return URL immediately without DB call")
   void resolveShortCodeCacheHit() { ... }
   ```

2. **Arrange-Act-Assert**: Follow AAA pattern
   ```java
   // Arrange
   String shortCode = "4c92";
   when(...).thenReturn(...);
   
   // Act
   String result = service.resolveShortCode(shortCode);
   
   // Assert
   assertEquals(expected, result);
   ```

3. **Mock External Dependencies**: Don't test external services
4. **Test Edge Cases**: Boundary values, null, empty
5. **Verify Interactions**: Check method calls with mocks
6. **Isolate Tests**: Tests should not depend on each other

## Troubleshooting

### Common Issues

1. **Tests Fail Intermittently**:
   - Check for race conditions
   - Verify test isolation
   - Check timing assumptions

2. **Integration Tests Fail**:
   - Verify Docker is running
   - Check Testcontainers setup
   - Verify port availability

3. **Performance Tests Fail**:
   - Check system resources
   - Verify no other processes using resources
   - Increase timeout values

## Conclusion

This testing strategy ensures:
- ✅ Correctness of all features
- ✅ Resilience to failures
- ✅ Performance under load
- ✅ Maintainability of codebase

For questions or issues, refer to the main [README.md](README.md) or open an issue.




