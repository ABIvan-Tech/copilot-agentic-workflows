# Testing & QA Practices

## Overview
Comprehensive guide for writing effective tests, following TDD practices, and ensuring code quality through automated testing.

## When to Use This Skill
- Writing unit, integration, or E2E tests
- Implementing test-driven development (TDD)
- Setting up testing infrastructure
- Debugging failing tests
- Improving test coverage
- Implementing mocking and stubbing strategies

## Testing Pyramid

### Unit Tests (Base - 70%)
**Purpose:** Test individual functions/methods in isolation
**Speed:** Very fast (milliseconds)
**Characteristics:**
- No external dependencies
- Test one thing at a time
- Should be deterministic
- Easy to write and maintain

### Integration Tests (Middle - 20%)
**Purpose:** Test how components work together
**Speed:** Moderate (seconds)
**Characteristics:**
- Test interactions between modules
- May use real database/APIs in test mode
- Test data flow between components

### E2E Tests (Top - 10%)
**Purpose:** Test complete user workflows
**Speed:** Slow (seconds to minutes)
**Characteristics:**
- Test from user's perspective
- Use real browser/environment
- Test critical user journeys only

## Unit Testing Best Practices

### 1. Test Structure (AAA Pattern)
```javascript
describe('UserService', () => {
  it('should create a new user with hashed password', async () => {
    // Arrange - Set up test data and dependencies
    const userData = { email: 'test@example.com', password: 'password123' };
    const mockHasher = jest.fn().mockResolvedValue('hashed_password');
    
    // Act - Execute the code being tested
    const user = await userService.create(userData, mockHasher);
    
    // Assert - Verify the results
    expect(user.email).toBe('test@example.com');
    expect(user.password).toBe('hashed_password');
    expect(mockHasher).toHaveBeenCalledWith('password123');
  });
});
```

### 2. Test Naming Convention
Use descriptive names that explain what, when, and expected result:
```javascript
// ✅ Good
it('should return 404 when user is not found')
it('should throw ValidationError when email is invalid')
it('should calculate total price including tax')

// ❌ Bad
it('works')
it('test user')
it('should pass')
```

### 3. One Assertion Per Test (When Possible)
```javascript
// ✅ Good - Focused tests
it('should set user email correctly', () => {
  const user = new User('test@example.com');
  expect(user.email).toBe('test@example.com');
});

it('should set user id automatically', () => {
  const user = new User('test@example.com');
  expect(user.id).toBeDefined();
});

// ⚠️ Acceptable - Related assertions
it('should create user with valid properties', () => {
  const user = new User('test@example.com', 'John');
  expect(user.email).toBe('test@example.com');
  expect(user.name).toBe('John');
  expect(user.createdAt).toBeInstanceOf(Date);
});
```

### 4. Test Edge Cases and Error Conditions
```javascript
describe('divide', () => {
  it('should divide two positive numbers', () => {
    expect(divide(10, 2)).toBe(5);
  });
  
  it('should handle negative numbers', () => {
    expect(divide(-10, 2)).toBe(-5);
  });
  
  it('should throw error when dividing by zero', () => {
    expect(() => divide(10, 0)).toThrow('Division by zero');
  });
  
  it('should handle decimal results', () => {
    expect(divide(10, 3)).toBeCloseTo(3.333, 3);
  });
});
```

## Mocking & Stubbing

### When to Mock
- External APIs/services
- Database calls
- File system operations
- Time-dependent functions
- Third-party libraries

### Mock Types

#### 1. Function Mocks (Spies)
```javascript
// Jest
const mockFn = jest.fn();
mockFn.mockReturnValue(42);
mockFn.mockResolvedValue({ id: 1 }); // For promises
mockFn.mockRejectedValue(new Error('Failed')); // For errors

expect(mockFn).toHaveBeenCalledWith('arg1', 'arg2');
expect(mockFn).toHaveBeenCalledTimes(1);

// Vitest (similar API)
const mockFn = vi.fn();
```

#### 2. Module Mocks
```javascript
// Mock entire module
jest.mock('./userService', () => ({
  getUser: jest.fn().mockResolvedValue({ id: 1, name: 'John' }),
  createUser: jest.fn()
}));

// Partial mock (keep some real implementations)
jest.mock('./userService', () => ({
  ...jest.requireActual('./userService'),
  getUser: jest.fn() // Only mock this function
}));
```

#### 3. Dependency Injection for Testability
```javascript
// ✅ Good - Easy to test
class UserService {
  constructor(database, emailService) {
    this.db = database;
    this.emailService = emailService;
  }
  
  async createUser(userData) {
    const user = await this.db.insert(userData);
    await this.emailService.sendWelcome(user.email);
    return user;
  }
}

// Test with mocks
const mockDb = { insert: jest.fn().mockResolvedValue({ id: 1 }) };
const mockEmail = { sendWelcome: jest.fn() };
const service = new UserService(mockDb, mockEmail);

// ❌ Bad - Hard to test
class UserService {
  async createUser(userData) {
    const user = await database.insert(userData); // Global dependency
    await sendEmail(user.email); // Tightly coupled
    return user;
  }
}
```

## Testing Asynchronous Code

### Promises
```javascript
// Using async/await
it('should fetch user data', async () => {
  const user = await fetchUser(123);
  expect(user.id).toBe(123);
});

// Using .resolves/.rejects
it('should fetch user data', () => {
  return expect(fetchUser(123)).resolves.toEqual({ id: 123 });
});

it('should handle errors', () => {
  return expect(fetchUser(-1)).rejects.toThrow('Invalid ID');
});
```

### Callbacks
```javascript
it('should call callback with result', (done) => {
  fetchUser(123, (err, user) => {
    expect(err).toBeNull();
    expect(user.id).toBe(123);
    done(); // Signal test completion
  });
});
```

## Test-Driven Development (TDD)

### Red-Green-Refactor Cycle

**1. Red - Write a failing test**
```javascript
it('should calculate total price with tax', () => {
  const cart = new ShoppingCart();
  cart.addItem({ price: 100, quantity: 2 });
  expect(cart.getTotalWithTax(0.1)).toBe(220); // 200 + 10% tax
});
// Test fails - getTotalWithTax doesn't exist yet
```

**2. Green - Write minimal code to pass**
```javascript
class ShoppingCart {
  constructor() {
    this.items = [];
  }
  
  addItem(item) {
    this.items.push(item);
  }
  
  getTotalWithTax(taxRate) {
    const subtotal = this.items.reduce((sum, item) => 
      sum + (item.price * item.quantity), 0);
    return subtotal * (1 + taxRate);
  }
}
// Test passes
```

**3. Refactor - Improve code quality**
```javascript
class ShoppingCart {
  constructor() {
    this.items = [];
  }
  
  addItem(item) {
    this.items.push(item);
  }
  
  getSubtotal() {
    return this.items.reduce((sum, item) => 
      sum + (item.price * item.quantity), 0);
  }
  
  getTotalWithTax(taxRate) {
    return this.getSubtotal() * (1 + taxRate);
  }
}
// Tests still pass, code is cleaner
```

## Integration Testing

### Database Integration Tests
```javascript
describe('UserRepository Integration', () => {
  let db;
  
  beforeAll(async () => {
    // Set up test database
    db = await setupTestDatabase();
  });
  
  afterAll(async () => {
    await db.close();
  });
  
  beforeEach(async () => {
    // Clear database before each test
    await db.query('DELETE FROM users');
  });
  
  it('should save and retrieve user', async () => {
    const repo = new UserRepository(db);
    const userData = { email: 'test@example.com', name: 'John' };
    
    const savedUser = await repo.save(userData);
    expect(savedUser.id).toBeDefined();
    
    const retrievedUser = await repo.findById(savedUser.id);
    expect(retrievedUser.email).toBe('test@example.com');
  });
});
```

### API Integration Tests
```javascript
describe('POST /api/users', () => {
  it('should create a new user', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ email: 'test@example.com', password: 'password123' })
      .expect(201);
    
    expect(response.body).toMatchObject({
      id: expect.any(Number),
      email: 'test@example.com'
    });
    expect(response.body.password).toBeUndefined(); // Shouldn't return password
  });
  
  it('should return 400 for invalid email', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ email: 'invalid-email', password: 'password123' })
      .expect(400);
    
    expect(response.body.error).toContain('Invalid email');
  });
});
```

## E2E Testing (Playwright/Cypress)

### Playwright Example
```javascript
import { test, expect } from '@playwright/test';

test('user can sign up and log in', async ({ page }) => {
  // Navigate to signup page
  await page.goto('/signup');
  
  // Fill out form
  await page.fill('input[name="email"]', 'test@example.com');
  await page.fill('input[name="password"]', 'password123');
  await page.click('button[type="submit"]');
  
  // Verify redirect to dashboard
  await expect(page).toHaveURL('/dashboard');
  await expect(page.locator('h1')).toContainText('Welcome');
});

test('displays error for invalid credentials', async ({ page }) => {
  await page.goto('/login');
  
  await page.fill('input[name="email"]', 'wrong@example.com');
  await page.fill('input[name="password"]', 'wrongpassword');
  await page.click('button[type="submit"]');
  
  await expect(page.locator('.error-message')).toContainText('Invalid credentials');
});
```

### Best Practices for E2E Tests
1. **Test user journeys, not implementation**
2. **Use data-testid attributes** instead of CSS selectors
3. **Keep tests independent** - each test should work in isolation
4. **Clean up test data** after each test
5. **Test critical paths only** - E2E tests are expensive

## Code Coverage

### What to Aim For
- **Statements**: 80%+ coverage
- **Branches**: 75%+ coverage
- **Functions**: 80%+ coverage
- **Lines**: 80%+ coverage

### Coverage Commands
```bash
# Jest
npm test -- --coverage

# Vitest
npm test -- --coverage

# Generate HTML report
npm test -- --coverage --coverageReporters=html
```

### Don't Chase 100% Coverage
- Focus on critical business logic
- Some code is hard/unnecessary to test (simple getters/setters)
- Test behavior, not implementation

## Common Testing Patterns

### 1. Test Fixtures (Reusable Test Data)
```javascript
// fixtures/users.js
export const validUser = {
  email: 'test@example.com',
  name: 'John Doe',
  password: 'password123'
};

export const adminUser = {
  ...validUser,
  email: 'admin@example.com',
  role: 'admin'
};

// In tests
import { validUser, adminUser } from './fixtures/users';
```

### 2. Factory Functions
```javascript
function createTestUser(overrides = {}) {
  return {
    id: Math.random(),
    email: 'test@example.com',
    name: 'John Doe',
    createdAt: new Date(),
    ...overrides
  };
}

// Usage
const user1 = createTestUser();
const user2 = createTestUser({ email: 'different@example.com' });
```

### 3. Custom Matchers
```javascript
expect.extend({
  toBeValidEmail(received) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    const pass = emailRegex.test(received);
    return {
      pass,
      message: () => `expected ${received} to be a valid email`
    };
  }
});

// Usage
expect('test@example.com').toBeValidEmail();
```

## Testing Best Practices Checklist

- [ ] Tests are independent (can run in any order)
- [ ] Tests are fast (unit tests < 100ms each)
- [ ] Test names clearly describe what is being tested
- [ ] Each test focuses on one behavior
- [ ] Edge cases and error conditions are tested
- [ ] No hardcoded values (use constants/fixtures)
- [ ] Mocks are used appropriately (not over-mocked)
- [ ] Tests are maintainable (don't test implementation details)
- [ ] Critical code paths have high coverage
- [ ] E2E tests cover main user workflows

## Framework-Specific Resources

### JavaScript/TypeScript
- **Jest**: Unit/Integration testing
- **Vitest**: Fast Vite-native testing
- **Playwright**: E2E testing (recommended)
- **Cypress**: E2E testing (UI-focused)
- **Testing Library**: React/Vue component testing

### Python
- **pytest**: Unit/Integration testing
- **unittest**: Built-in testing
- **pytest-mock**: Mocking utilities
- **Selenium**: E2E testing

### Java
- **JUnit 5**: Unit testing
- **Mockito**: Mocking framework
- **TestContainers**: Integration testing with Docker
- **Selenium/Playwright**: E2E testing

## Common Pitfalls to Avoid

1. **Testing implementation details** - Test behavior, not internal structure
2. **Over-mocking** - Balance between isolation and realistic tests
3. **Flaky tests** - Tests that randomly fail (usually timing issues)
4. **Slow tests** - Keep unit tests fast
5. **Ignoring test maintenance** - Treat test code like production code
6. **Testing the framework** - Don't test library code, test your code
7. **No assertions** - Every test should have at least one assertion
