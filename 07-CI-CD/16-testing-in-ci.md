# Testing in CI/CD

## Testing Pyramid

```
┌─────────────────────────────────────────────────────────────────┐
│                    Testing Pyramid                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                          ▲                                       │
│                         /│\                                      │
│                        / │ \                                     │
│                       /  │  \     E2E Tests (Few)               │
│                      /   │   \    - Slow, expensive             │
│                     /────┼────\   - Test full system            │
│                    /     │     \                                 │
│                   /      │      \                                │
│                  / Integration   \  Integration Tests           │
│                 /    Tests        \ - Medium speed               │
│                /─────────┼─────────\ - Test components together │
│               /          │          \                            │
│              /           │           \                           │
│             /     Unit Tests          \  Unit Tests (Many)      │
│            /─────────────┼─────────────\ - Fast, cheap          │
│                          │               - Test single units    │
│                                                                  │
│   Fast ◄────────────────────────────────────────────► Slow     │
│   Cheap ◄───────────────────────────────────────────► Expensive │
│   Many ◄────────────────────────────────────────────► Few       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Types of Tests

| Type | Scope | Speed | When to Run |
|------|-------|-------|-------------|
| Unit | Single function/class | Fast | Every commit |
| Integration | Multiple components | Medium | Every commit/PR |
| E2E | Full application | Slow | Before deploy |
| Performance | Load/stress | Varies | Scheduled/release |
| Security | Vulnerabilities | Medium | Every PR |

## Unit Testing

### JavaScript (Jest)

```javascript
// math.js
export function add(a, b) {
  return a + b;
}

// math.test.js
import { add } from './math';

describe('add function', () => {
  test('adds two positive numbers', () => {
    expect(add(1, 2)).toBe(3);
  });

  test('adds negative numbers', () => {
    expect(add(-1, -2)).toBe(-3);
  });

  test('handles zero', () => {
    expect(add(5, 0)).toBe(5);
  });
});
```

### CI Configuration (Jest)

```yaml
# GitHub Actions
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm test -- --coverage --ci
      - uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
```

### Python (pytest)

```python
# calculator.py
def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

# test_calculator.py
import pytest
from calculator import divide

def test_divide_positive():
    assert divide(10, 2) == 5

def test_divide_negative():
    assert divide(-10, 2) == -5

def test_divide_by_zero():
    with pytest.raises(ValueError):
        divide(10, 0)

@pytest.mark.parametrize("a,b,expected", [
    (10, 2, 5),
    (9, 3, 3),
    (15, 5, 3),
])
def test_divide_parametrized(a, b, expected):
    assert divide(a, b) == expected
```

### CI Configuration (pytest)

```yaml
# GitHub Actions
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install -e ".[dev]"
      - run: pytest --cov=src --cov-report=xml
      - uses: codecov/codecov-action@v3
```

### Java (JUnit 5)

```java
// Calculator.java
public class Calculator {
    public int divide(int a, int b) {
        if (b == 0) {
            throw new IllegalArgumentException("Cannot divide by zero");
        }
        return a / b;
    }
}

// CalculatorTest.java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {
    private Calculator calculator;

    @BeforeEach
    void setUp() {
        calculator = new Calculator();
    }

    @Test
    void shouldDividePositiveNumbers() {
        assertEquals(5, calculator.divide(10, 2));
    }

    @Test
    void shouldThrowOnDivideByZero() {
        assertThrows(IllegalArgumentException.class,
            () -> calculator.divide(10, 0));
    }

    @ParameterizedTest
    @CsvSource({"10,2,5", "9,3,3", "15,5,3"})
    void shouldDivideCorrectly(int a, int b, int expected) {
        assertEquals(expected, calculator.divide(a, b));
    }
}
```

### CI Configuration (Maven/JUnit)

```yaml
# GitHub Actions
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'maven'
      - run: mvn test
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: target/surefire-reports/
```

## Integration Testing

### API Integration Test

```javascript
// api.test.js
import request from 'supertest';
import app from './app';

describe('API Integration Tests', () => {
  beforeAll(async () => {
    await setupDatabase();
  });

  afterAll(async () => {
    await cleanupDatabase();
  });

  describe('POST /users', () => {
    test('creates a new user', async () => {
      const response = await request(app)
        .post('/users')
        .send({ name: 'John', email: 'john@example.com' })
        .expect(201);

      expect(response.body).toHaveProperty('id');
      expect(response.body.name).toBe('John');
    });

    test('returns 400 for invalid data', async () => {
      await request(app)
        .post('/users')
        .send({ name: '' })
        .expect(400);
    });
  });
});
```

### Database Integration Test

```python
# test_user_repository.py
import pytest
from app.models import User
from app.repositories import UserRepository

@pytest.fixture
def db_session():
    """Create test database session."""
    session = create_test_session()
    yield session
    session.rollback()
    session.close()

class TestUserRepository:
    def test_create_user(self, db_session):
        repo = UserRepository(db_session)
        user = repo.create(name="John", email="john@example.com")

        assert user.id is not None
        assert user.name == "John"

    def test_find_by_email(self, db_session):
        repo = UserRepository(db_session)
        repo.create(name="John", email="john@example.com")

        user = repo.find_by_email("john@example.com")

        assert user is not None
        assert user.name == "John"
```

### CI with Test Database

```yaml
# GitHub Actions
jobs:
  integration-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install -e ".[dev]"
      - run: pytest tests/integration/
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/testdb
```

## End-to-End Testing

### Playwright (JavaScript)

```javascript
// e2e/login.spec.js
import { test, expect } from '@playwright/test';

test.describe('Login Flow', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
  });

  test('successful login', async ({ page }) => {
    await page.fill('[name="email"]', 'user@example.com');
    await page.fill('[name="password"]', 'password123');
    await page.click('button[type="submit"]');

    await expect(page).toHaveURL('/dashboard');
    await expect(page.locator('h1')).toHaveText('Welcome');
  });

  test('shows error on invalid credentials', async ({ page }) => {
    await page.fill('[name="email"]', 'wrong@example.com');
    await page.fill('[name="password"]', 'wrongpass');
    await page.click('button[type="submit"]');

    await expect(page.locator('.error')).toBeVisible();
  });
});
```

### Playwright CI Configuration

```yaml
# GitHub Actions
jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npm run build
      - run: npm run start &
      - run: npx playwright test
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
```

### Cypress

```javascript
// cypress/e2e/login.cy.js
describe('Login', () => {
  beforeEach(() => {
    cy.visit('/login');
  });

  it('logs in successfully', () => {
    cy.get('[name="email"]').type('user@example.com');
    cy.get('[name="password"]').type('password123');
    cy.get('button[type="submit"]').click();

    cy.url().should('include', '/dashboard');
    cy.contains('h1', 'Welcome');
  });

  it('shows validation errors', () => {
    cy.get('button[type="submit"]').click();
    cy.contains('Email is required');
  });
});
```

## Test Reporting

### JUnit XML Format

```yaml
# Most CI systems understand JUnit XML
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test -- --reporters=jest-junit
      - uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Jest Tests
          path: junit.xml
          reporter: jest-junit
```

### Code Coverage

```yaml
# Upload to Codecov
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test -- --coverage
      - uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
          fail_ci_if_error: true
          verbose: true

# Or Coveralls
      - uses: coverallsapp/github-action@v2
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Test Summary in PR

```yaml
# GitHub Actions
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test -- --json --outputFile=results.json
      - uses: actions/github-script@v7
        if: always()
        with:
          script: |
            const fs = require('fs');
            const results = JSON.parse(fs.readFileSync('results.json'));
            const summary = `
            ## Test Results
            - Tests: ${results.numTotalTests}
            - Passed: ${results.numPassedTests}
            - Failed: ${results.numFailedTests}
            `;
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: summary
            });
```

## Performance Testing

### k6 Load Testing

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 20 },  // Ramp up
    { duration: '1m', target: 20 },   // Stay at 20
    { duration: '10s', target: 0 },   // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% under 500ms
    http_req_failed: ['rate<0.01'],    // <1% error rate
  },
};

export default function () {
  const res = http.get('http://localhost:3000/api/users');

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });

  sleep(1);
}
```

### k6 in CI

```yaml
jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker-compose up -d
      - uses: grafana/k6-action@v0.3.0
        with:
          filename: load-test.js
      - run: docker-compose down
```

## Security Testing

### OWASP ZAP

```yaml
jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker-compose up -d
      - uses: zaproxy/action-full-scan@v0.7.0
        with:
          target: 'http://localhost:3000'
          rules_file_name: '.zap/rules.tsv'
      - uses: actions/upload-artifact@v4
        with:
          name: zap-report
          path: report_html.html
```

### Dependency Scanning

```yaml
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # npm audit
      - run: npm audit --audit-level=high

      # Snyk
      - uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
```

## Test Best Practices in CI

### Fast Feedback

```yaml
# Run fastest tests first
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint

  unit-test:
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - run: npm test -- --testPathPattern=unit

  integration-test:
    needs: unit-test
    runs-on: ubuntu-latest
    steps:
      - run: npm test -- --testPathPattern=integration

  e2e-test:
    needs: integration-test
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:e2e
```

### Parallel Test Execution

```yaml
# Split tests across runners
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - run: npm test -- --shard=${{ matrix.shard }}/${{ strategy.job-total }}
```

### Flaky Test Handling

```yaml
# Retry flaky tests
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test -- --retries=2

      # Or with action
      - uses: nick-invision/retry@v2
        with:
          max_attempts: 3
          command: npm test
```

## Quick Reference

### Test Commands

| Framework | Run Tests | Coverage | Watch |
|-----------|-----------|----------|-------|
| Jest | `npm test` | `npm test -- --coverage` | `npm test -- --watch` |
| pytest | `pytest` | `pytest --cov` | `pytest-watch` |
| JUnit | `mvn test` | Jacoco plugin | - |
| Playwright | `npx playwright test` | - | `--ui` |

### Coverage Thresholds

```json
// Jest config
{
  "coverageThreshold": {
    "global": {
      "branches": 80,
      "functions": 80,
      "lines": 80,
      "statements": 80
    }
  }
}
```

### Common CI Patterns

| Pattern | Description |
|---------|-------------|
| Fail fast | Stop on first failure |
| Parallel tests | Run tests concurrently |
| Test sharding | Split tests across jobs |
| Retry flaky | Retry failed tests |
| Cache deps | Speed up test runs |

---

**Previous:** [15-build-automation.md](15-build-automation.md) | **Next:** [17-code-quality.md](17-code-quality.md)
