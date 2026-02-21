# Code Quality & Clean Code

## Overview
This skill provides comprehensive guidance on writing clean, maintainable, and scalable code. It covers SOLID principles, design patterns, code smells, refactoring techniques, and clean architecture practices. Following these principles leads to code that is easier to understand, test, maintain, and extend. The examples focus on JavaScript/TypeScript in modern web development contexts (React, Node.js) but the principles apply universally across languages and frameworks.

## When to Use This Skill
- Writing new features or components
- Refactoring existing code
- Conducting or participating in code reviews
- Architecting new systems or modules
- Improving code maintainability and testability
- Reducing technical debt
- Debugging complex issues caused by poor code structure
- Training junior developers on best practices
- Making design decisions about patterns and architecture

---

## 1. SOLID Principles

### Single Responsibility Principle (SRP)
*A class/module should have one, and only one, reason to change.*

**❌ Bad - Multiple responsibilities:**
```typescript
// UserManager does too many things
class UserManager {
  async createUser(userData: UserData) {
    // Validation
    if (!userData.email || !userData.email.includes('@')) {
      throw new Error('Invalid email');
    }
    
    // Database operations
    const user = await db.users.create(userData);
    
    // Email sending
    await fetch('https://email-service.com/send', {
      method: 'POST',
      body: JSON.stringify({
        to: user.email,
        subject: 'Welcome!',
        body: `Welcome ${user.name}!`
      })
    });
    
    // Logging
    console.log(`User created: ${user.id}`);
    
    // Analytics
    await analytics.track('user_created', { userId: user.id });
    
    return user;
  }
}
```

**✅ Good - Separated responsibilities:**
```typescript
// Each class has a single responsibility
class UserValidator {
  validate(userData: UserData): ValidationResult {
    const errors: string[] = [];
    if (!userData.email || !userData.email.includes('@')) {
      errors.push('Invalid email');
    }
    if (!userData.name || userData.name.length < 2) {
      errors.push('Name must be at least 2 characters');
    }
    return { isValid: errors.length === 0, errors };
  }
}

class UserRepository {
  async create(userData: UserData): Promise<User> {
    return await db.users.create(userData);
  }
  
  async findById(id: string): Promise<User | null> {
    return await db.users.findOne({ id });
  }
}

class EmailService {
  async sendWelcomeEmail(user: User): Promise<void> {
    await this.emailClient.send({
      to: user.email,
      subject: 'Welcome!',
      body: `Welcome ${user.name}!`
    });
  }
}

class UserAnalytics {
  track(event: string, properties: Record<string, any>): void {
    this.analyticsClient.track(event, properties);
  }
}

class UserService {
  constructor(
    private validator: UserValidator,
    private repository: UserRepository,
    private emailService: EmailService,
    private analytics: UserAnalytics
  ) {}
  
  async createUser(userData: UserData): Promise<User> {
    const validation = this.validator.validate(userData);
    if (!validation.isValid) {
      throw new ValidationError(validation.errors);
    }
    
    const user = await this.repository.create(userData);
    await this.emailService.sendWelcomeEmail(user);
    this.analytics.track('user_created', { userId: user.id });
    
    return user;
  }
}
```

### Open/Closed Principle (OCP)
*Software entities should be open for extension, but closed for modification.*

**❌ Bad - Must modify class to add new behavior:**
```typescript
class PaymentProcessor {
  processPayment(amount: number, type: string) {
    if (type === 'credit_card') {
      // Credit card processing logic
      return this.processCreditCard(amount);
    } else if (type === 'paypal') {
      // PayPal processing logic
      return this.processPayPal(amount);
    } else if (type === 'stripe') {
      // Stripe processing logic
      return this.processStripe(amount);
    }
    // Need to modify this class every time we add a new payment method!
  }
}
```

**✅ Good - Open for extension via abstraction:**
```typescript
interface PaymentMethod {
  process(amount: number): Promise<PaymentResult>;
  validate(amount: number): boolean;
}

class CreditCardPayment implements PaymentMethod {
  async process(amount: number): Promise<PaymentResult> {
    // Credit card specific logic
    return { success: true, transactionId: '...' };
  }
  
  validate(amount: number): boolean {
    return amount > 0 && amount < 10000;
  }
}

class PayPalPayment implements PaymentMethod {
  async process(amount: number): Promise<PaymentResult> {
    // PayPal specific logic
    return { success: true, transactionId: '...' };
  }
  
  validate(amount: number): boolean {
    return amount > 0;
  }
}

class CryptoPayment implements PaymentMethod {
  async process(amount: number): Promise<PaymentResult> {
    // Cryptocurrency specific logic
    return { success: true, transactionId: '...' };
  }
  
  validate(amount: number): boolean {
    return amount > 0.01;
  }
}

class PaymentProcessor {
  constructor(private paymentMethod: PaymentMethod) {}
  
  async processPayment(amount: number): Promise<PaymentResult> {
    if (!this.paymentMethod.validate(amount)) {
      throw new Error('Invalid payment amount');
    }
    return await this.paymentMethod.process(amount);
  }
  
  // Can easily swap payment methods without modifying this class
  setPaymentMethod(method: PaymentMethod): void {
    this.paymentMethod = method;
  }
}

// Usage - easily extensible with new payment methods
const processor = new PaymentProcessor(new CreditCardPayment());
await processor.processPayment(100);

// Switch to different method
processor.setPaymentMethod(new CryptoPayment());
await processor.processPayment(0.5);
```

### Liskov Substitution Principle (LSP)
*Objects of a superclass should be replaceable with objects of its subclasses without breaking the application.*

**❌ Bad - Subclass violates superclass contract:**
```typescript
class Bird {
  fly(): void {
    console.log('Flying...');
  }
}

class Penguin extends Bird {
  fly(): void {
    // Penguins can't fly! This violates LSP
    throw new Error('Penguins cannot fly!');
  }
}

function makeBirdFly(bird: Bird) {
  bird.fly(); // This will crash if bird is a Penguin!
}

const penguin = new Penguin();
makeBirdFly(penguin); // ❌ Throws error - LSP violation
```

**✅ Good - Proper abstraction respecting capabilities:**
```typescript
interface Animal {
  move(): void;
  makeSound(): void;
}

interface Flyable {
  fly(): void;
}

class Bird implements Animal {
  move(): void {
    console.log('Moving...');
  }
  
  makeSound(): void {
    console.log('Chirp!');
  }
}

class Sparrow extends Bird implements Flyable {
  fly(): void {
    console.log('Sparrow flying...');
  }
}

class Penguin extends Bird {
  move(): void {
    console.log('Penguin waddling...');
  }
  
  swim(): void {
    console.log('Penguin swimming...');
  }
}

function makeAnimalMove(animal: Animal) {
  animal.move(); // Works for all animals
}

function makeFlyableObjectFly(flyable: Flyable) {
  flyable.fly(); // Only accepts things that can fly
}

const sparrow = new Sparrow();
const penguin = new Penguin();

makeAnimalMove(sparrow); // ✅ Works
makeAnimalMove(penguin); // ✅ Works
makeFlyableObjectFly(sparrow); // ✅ Works
// makeFlyableObjectFly(penguin); // ✅ TypeScript prevents this at compile time
```

### Interface Segregation Principle (ISP)
*Clients should not be forced to depend on interfaces they don't use.*

**❌ Bad - Fat interface with unneeded methods:**
```typescript
interface Worker {
  work(): void;
  eat(): void;
  sleep(): void;
  getPaid(): void;
}

class HumanWorker implements Worker {
  work(): void { console.log('Working...'); }
  eat(): void { console.log('Eating...'); }
  sleep(): void { console.log('Sleeping...'); }
  getPaid(): void { console.log('Getting paid...'); }
}

class RobotWorker implements Worker {
  work(): void { console.log('Working 24/7...'); }
  
  // Robots don't eat or sleep - forced to implement useless methods
  eat(): void { 
    throw new Error('Robots don\'t eat!');
  }
  
  sleep(): void { 
    throw new Error('Robots don\'t sleep!');
  }
  
  getPaid(): void {
    throw new Error('Robots don\'t get paid!');
  }
}
```

**✅ Good - Segregated interfaces:**
```typescript
interface Workable {
  work(): void;
}

interface Eatable {
  eat(): void;
}

interface Sleepable {
  sleep(): void;
}

interface Payable {
  getPaid(): void;
}

class HumanWorker implements Workable, Eatable, Sleepable, Payable {
  work(): void { console.log('Working...'); }
  eat(): void { console.log('Eating lunch...'); }
  sleep(): void { console.log('Sleeping...'); }
  getPaid(): void { console.log('Getting paid...'); }
}

class RobotWorker implements Workable {
  work(): void { 
    console.log('Working 24/7 without breaks...'); 
  }
  
  charge(): void {
    console.log('Charging battery...');
  }
}

class Contractor implements Workable, Payable {
  work(): void { console.log('Working on contract...'); }
  getPaid(): void { console.log('Receiving payment...'); }
  // Doesn't need to implement eat() or sleep()
}

// Functions only depend on what they need
function scheduleWork(worker: Workable) {
  worker.work();
}

function processPayroll(employee: Payable) {
  employee.getPaid();
}

const human = new HumanWorker();
const robot = new RobotWorker();
const contractor = new Contractor();

scheduleWork(human);     // ✅ Works
scheduleWork(robot);     // ✅ Works
scheduleWork(contractor); // ✅ Works
processPayroll(human);   // ✅ Works
processPayroll(contractor); // ✅ Works
// processPayroll(robot); // ✅ TypeScript prevents this
```

### Dependency Inversion Principle (DIP)
*Depend upon abstractions, not concretions. High-level modules should not depend on low-level modules.*

**❌ Bad - Tight coupling to concrete implementations:**
```typescript
class MySQLDatabase {
  connect() { /* Connect to MySQL */ }
  query(sql: string) { /* Execute MySQL query */ }
}

class UserService {
  private db: MySQLDatabase;
  
  constructor() {
    this.db = new MySQLDatabase(); // Tightly coupled!
  }
  
  async getUser(id: string) {
    return this.db.query(`SELECT * FROM users WHERE id = ${id}`);
  }
}

// Problem: Cannot switch databases or test without MySQL
// Cannot inject mocks for testing
```

**✅ Good - Depend on abstractions:**
```typescript
interface Database {
  connect(): Promise<void>;
  query<T>(query: string, params?: any[]): Promise<T>;
  disconnect(): Promise<void>;
}

class MySQLDatabase implements Database {
  async connect(): Promise<void> {
    // MySQL connection logic
  }
  
  async query<T>(query: string, params?: any[]): Promise<T> {
    // MySQL query execution
    return {} as T;
  }
  
  async disconnect(): Promise<void> {
    // Cleanup
  }
}

class PostgreSQLDatabase implements Database {
  async connect(): Promise<void> {
    // PostgreSQL connection logic
  }
  
  async query<T>(query: string, params?: any[]): Promise<T> {
    // PostgreSQL query execution
    return {} as T;
  }
  
  async disconnect(): Promise<void> {
    // Cleanup
  }
}

class MockDatabase implements Database {
  private data: Map<string, any> = new Map();
  
  async connect(): Promise<void> {}
  
  async query<T>(query: string, params?: any[]): Promise<T> {
    // Return mock data for testing
    return this.data.get(query) as T;
  }
  
  async disconnect(): Promise<void> {}
  
  setMockData(query: string, data: any): void {
    this.data.set(query, data);
  }
}

class UserService {
  constructor(private db: Database) {}
  
  async getUser(id: string): Promise<User> {
    return this.db.query<User>(
      'SELECT * FROM users WHERE id = ?',
      [id]
    );
  }
}

// Usage - easily swap implementations
const mysqlDb = new MySQLDatabase();
const prodService = new UserService(mysqlDb);

// Testing - inject mock
const mockDb = new MockDatabase();
mockDb.setMockData('SELECT * FROM users WHERE id = ?', { id: '1', name: 'Test' });
const testService = new UserService(mockDb);

// Switch to PostgreSQL without changing UserService
const postgresDb = new PostgreSQLDatabase();
const newProdService = new UserService(postgresDb);
```

---

## 2. Design Patterns

### Creational Patterns

#### Factory Pattern
*Creates objects without specifying the exact class to create.*

**❌ Bad - Direct instantiation everywhere:**
```typescript
class ProductList extends React.Component {
  renderProduct(type: string, data: any) {
    // Scattered object creation logic
    if (type === 'book') {
      return new Book(data.title, data.author, data.pages);
    } else if (type === 'electronics') {
      return new Electronics(data.name, data.warranty, data.voltage);
    } else if (type === 'clothing') {
      return new Clothing(data.name, data.size, data.color);
    }
    // Hard to maintain and extend
  }
}
```

**✅ Good - Factory pattern:**
```typescript
interface Product {
  id: string;
  name: string;
  price: number;
  getDescription(): string;
}

class Book implements Product {
  constructor(
    public id: string,
    public name: string,
    public price: number,
    public author: string,
    public pages: number
  ) {}
  
  getDescription(): string {
    return `${this.name} by ${this.author} - ${this.pages} pages`;
  }
}

class Electronics implements Product {
  constructor(
    public id: string,
    public name: string,
    public price: number,
    public warranty: string,
    public voltage: string
  ) {}
  
  getDescription(): string {
    return `${this.name} - ${this.warranty} warranty, ${this.voltage}`;
  }
}

class ProductFactory {
  static createProduct(type: string, data: any): Product {
    switch (type) {
      case 'book':
        return new Book(data.id, data.name, data.price, data.author, data.pages);
      case 'electronics':
        return new Electronics(data.id, data.name, data.price, data.warranty, data.voltage);
      default:
        throw new Error(`Unknown product type: ${type}`);
    }
  }
}

// Usage in React component
function ProductList({ items }: { items: any[] }) {
  const products = items.map(item => 
    ProductFactory.createProduct(item.type, item)
  );
  
  return (
    <div>
      {products.map(product => (
        <div key={product.id}>
          <h3>{product.name}</h3>
          <p>{product.getDescription()}</p>
          <p>${product.price}</p>
        </div>
      ))}
    </div>
  );
}
```

#### Builder Pattern
*Constructs complex objects step by step.*

**❌ Bad - Telescoping constructor:**
```typescript
class HttpRequest {
  constructor(
    url: string,
    method?: string,
    headers?: Record<string, string>,
    body?: any,
    timeout?: number,
    retries?: number,
    cache?: boolean,
    credentials?: string
  ) {
    // Too many parameters, confusing order
  }
}

// Usage is error-prone and unreadable
const request = new HttpRequest(
  'https://api.example.com',
  'POST',
  { 'Content-Type': 'application/json' },
  { data: 'value' },
  5000,
  3,
  false,
  'include'
);
```

**✅ Good - Builder pattern:**
```typescript
class HttpRequest {
  constructor(
    public url: string,
    public method: string = 'GET',
    public headers: Record<string, string> = {},
    public body?: any,
    public timeout: number = 30000,
    public retries: number = 0,
    public cache: boolean = true,
    public credentials?: string
  ) {}
}

class HttpRequestBuilder {
  private url: string = '';
  private method: string = 'GET';
  private headers: Record<string, string> = {};
  private body?: any;
  private timeout: number = 30000;
  private retries: number = 0;
  private cache: boolean = true;
  private credentials?: string;
  
  setUrl(url: string): HttpRequestBuilder {
    this.url = url;
    return this;
  }
  
  setMethod(method: string): HttpRequestBuilder {
    this.method = method;
    return this;
  }
  
  setHeaders(headers: Record<string, string>): HttpRequestBuilder {
    this.headers = { ...this.headers, ...headers };
    return this;
  }
  
  setHeader(key: string, value: string): HttpRequestBuilder {
    this.headers[key] = value;
    return this;
  }
  
  setBody(body: any): HttpRequestBuilder {
    this.body = body;
    return this;
  }
  
  setTimeout(timeout: number): HttpRequestBuilder {
    this.timeout = timeout;
    return this;
  }
  
  setRetries(retries: number): HttpRequestBuilder {
    this.retries = retries;
    return this;
  }
  
  setCache(cache: boolean): HttpRequestBuilder {
    this.cache = cache;
    return this;
  }
  
  setCredentials(credentials: string): HttpRequestBuilder {
    this.credentials = credentials;
    return this;
  }
  
  build(): HttpRequest {
    if (!this.url) {
      throw new Error('URL is required');
    }
    return new HttpRequest(
      this.url,
      this.method,
      this.headers,
      this.body,
      this.timeout,
      this.retries,
      this.cache,
      this.credentials
    );
  }
}

// Usage - readable and flexible
const request = new HttpRequestBuilder()
  .setUrl('https://api.example.com/users')
  .setMethod('POST')
  .setHeader('Content-Type', 'application/json')
  .setHeader('Authorization', 'Bearer token123')
  .setBody({ name: 'John', email: 'john@example.com' })
  .setTimeout(5000)
  .setRetries(3)
  .setCache(false)
  .build();
```

#### Singleton Pattern
*Ensures a class has only one instance.*

**❌ Bad - When NOT to use Singleton (most cases!):**
```typescript
// Anti-pattern: Global state that makes testing hard
class ConfigManager {
  private static instance: ConfigManager;
  private config: any = {};
  
  private constructor() {}
  
  static getInstance(): ConfigManager {
    if (!ConfigManager.instance) {
      ConfigManager.instance = new ConfigManager();
    }
    return ConfigManager.instance;
  }
  
  set(key: string, value: any) {
    this.config[key] = value;
  }
  
  get(key: string) {
    return this.config[key];
  }
}

// Problems:
// 1. Hard to test - global mutable state
// 2. Hidden dependencies
// 3. Tight coupling
// 4. Difficult to mock
```

**✅ Good - When Singleton is appropriate (rare!):**
```typescript
// Legitimate use case: Logger with single output stream
class Logger {
  private static instance: Logger;
  private logStream: WritableStream;
  
  private constructor() {
    this.logStream = this.initializeLogStream();
  }
  
  static getInstance(): Logger {
    if (!Logger.instance) {
      Logger.instance = new Logger();
    }
    return Logger.instance;
  }
  
  private initializeLogStream(): WritableStream {
    // Initialize single log stream
    return process.stdout;
  }
  
  log(level: string, message: string, meta?: any): void {
    const timestamp = new Date().toISOString();
    const logEntry = JSON.stringify({
      timestamp,
      level,
      message,
      ...meta
    });
    this.logStream.write(logEntry + '\n');
  }
  
  info(message: string, meta?: any): void {
    this.log('INFO', message, meta);
  }
  
  error(message: string, meta?: any): void {
    this.log('ERROR', message, meta);
  }
}

// Better alternative: Dependency injection instead of singleton
class ConfigService {
  constructor(private config: Record<string, any>) {}
  
  get(key: string): any {
    return this.config[key];
  }
}

// Inject the same instance everywhere
const config = new ConfigService({ apiUrl: 'https://api.example.com' });
const userService = new UserService(config);
const orderService = new OrderService(config);
```

### Structural Patterns

#### Adapter Pattern
*Allows incompatible interfaces to work together.*

**❌ Bad - Tight coupling to specific implementation:**
```typescript
// Old payment system
class LegacyPaymentGateway {
  makePayment(accountNumber: string, amount: number): boolean {
    console.log(`Processing $${amount} from account ${accountNumber}`);
    return true;
  }
}

class CheckoutService {
  processOrder(order: Order) {
    const gateway = new LegacyPaymentGateway();
    // Tightly coupled to legacy system
    const success = gateway.makePayment(order.accountNumber, order.total);
    // Can't easily switch to new payment system
  }
}
```

**✅ Good - Adapter pattern:**
```typescript
interface PaymentGateway {
  processPayment(paymentDetails: PaymentDetails): Promise<PaymentResult>;
}

interface PaymentDetails {
  amount: number;
  currency: string;
  source: string;
}

interface PaymentResult {
  success: boolean;
  transactionId: string;
}

// Legacy system
class LegacyPaymentGateway {
  makePayment(accountNumber: string, amount: number): boolean {
    console.log(`Processing $${amount} from account ${accountNumber}`);
    return true;
  }
}

// Adapter for legacy system
class LegacyPaymentAdapter implements PaymentGateway {
  constructor(private legacyGateway: LegacyPaymentGateway) {}
  
  async processPayment(details: PaymentDetails): Promise<PaymentResult> {
    // Adapt new interface to legacy system
    const success = this.legacyGateway.makePayment(
      details.source,
      details.amount
    );
    
    return {
      success,
      transactionId: `LEGACY-${Date.now()}`
    };
  }
}

// New modern payment system
class StripePaymentGateway implements PaymentGateway {
  async processPayment(details: PaymentDetails): Promise<PaymentResult> {
    // Direct implementation for new system
    const response = await fetch('https://api.stripe.com/v1/charges', {
      method: 'POST',
      body: JSON.stringify(details)
    });
    const data = await response.json();
    return {
      success: data.success,
      transactionId: data.id
    };
  }
}

class CheckoutService {
  constructor(private paymentGateway: PaymentGateway) {}
  
  async processOrder(order: Order): Promise<void> {
    // Works with any payment gateway through common interface
    const result = await this.paymentGateway.processPayment({
      amount: order.total,
      currency: 'USD',
      source: order.paymentSource
    });
    
    if (result.success) {
      console.log(`Payment successful: ${result.transactionId}`);
    }
  }
}

// Easy to switch between implementations
const legacyService = new CheckoutService(
  new LegacyPaymentAdapter(new LegacyPaymentGateway())
);

const modernService = new CheckoutService(
  new StripePaymentGateway()
);
```

#### Decorator Pattern
*Adds new functionality to objects dynamically.*

**❌ Bad - Inheritance explosion:**
```typescript
class Coffee {
  cost(): number { return 2.0; }
  description(): string { return 'Simple coffee'; }
}

class CoffeeWithMilk extends Coffee {
  cost(): number { return super.cost() + 0.5; }
  description(): string { return super.description() + ', milk'; }
}

class CoffeeWithMilkAndSugar extends CoffeeWithMilk {
  cost(): number { return super.cost() + 0.25; }
  description(): string { return super.description() + ', sugar'; }
}

// Need a new class for every combination!
class CoffeeWithSugar extends Coffee { /* ... */ }
class CoffeeWithMilkAndSugarAndVanilla extends CoffeeWithMilkAndSugar { /* ... */ }
// This becomes unmanageable quickly
```

**✅ Good - Decorator pattern:**
```typescript
interface Beverage {
  cost(): number;
  description(): string;
}

class Coffee implements Beverage {
  cost(): number {
    return 2.0;
  }
  
  description(): string {
    return 'Simple coffee';
  }
}

abstract class BeverageDecorator implements Beverage {
  constructor(protected beverage: Beverage) {}
  
  abstract cost(): number;
  abstract description(): string;
}

class MilkDecorator extends BeverageDecorator {
  cost(): number {
    return this.beverage.cost() + 0.5;
  }
  
  description(): string {
    return this.beverage.description() + ', milk';
  }
}

class SugarDecorator extends BeverageDecorator {
  cost(): number {
    return this.beverage.cost() + 0.25;
  }
  
  description(): string {
    return this.beverage.description() + ', sugar';
  }
}

class VanillaDecorator extends BeverageDecorator {
  cost(): number {
    return this.beverage.cost() + 0.75;
  }
  
  description(): string {
    return this.beverage.description() + ', vanilla';
  }
}

// Usage - compose decorators dynamically
let beverage: Beverage = new Coffee();
console.log(`${beverage.description()} = $${beverage.cost()}`);
// Output: "Simple coffee = $2.0"

beverage = new MilkDecorator(beverage);
console.log(`${beverage.description()} = $${beverage.cost()}`);
// Output: "Simple coffee, milk = $2.5"

beverage = new SugarDecorator(beverage);
beverage = new VanillaDecorator(beverage);
console.log(`${beverage.description()} = $${beverage.cost()}`);
// Output: "Simple coffee, milk, sugar, vanilla = $3.5"

// Easy to add any combination without new classes
```

#### Facade Pattern
*Provides a simplified interface to a complex subsystem.*

**❌ Bad - Complex API exposed to clients:**
```typescript
// Client must understand and coordinate multiple subsystems
class OrderController {
  async createOrder(req: Request, res: Response) {
    // Client deals with all complexity
    const inventory = new InventorySystem();
    const payment = new PaymentSystem();
    const shipping = new ShippingSystem();
    const notification = new NotificationSystem();
    
    // Complex coordination logic
    const available = await inventory.checkAvailability(req.body.items);
    if (!available) {
      return res.status(400).json({ error: 'Items not available' });
    }
    
    await inventory.reserveItems(req.body.items);
    
    try {
      const paymentResult = await payment.processPayment(req.body.payment);
      if (!paymentResult.success) {
        await inventory.releaseItems(req.body.items);
        return res.status(400).json({ error: 'Payment failed' });
      }
      
      const shipment = await shipping.createShipment(req.body.address, req.body.items);
      await notification.sendOrderConfirmation(req.body.email, shipment.trackingNumber);
      
      res.json({ success: true, shipment });
    } catch (error) {
      await inventory.releaseItems(req.body.items);
      throw error;
    }
  }
}
```

**✅ Good - Facade simplifies the interface:**
```typescript
class InventorySystem {
  async checkAvailability(items: Item[]): Promise<boolean> { return true; }
  async reserveItems(items: Item[]): Promise<void> {}
  async releaseItems(items: Item[]): Promise<void> {}
}

class PaymentSystem {
  async processPayment(details: any): Promise<PaymentResult> { 
    return { success: true, transactionId: '123' };
  }
}

class ShippingSystem {
  async createShipment(address: Address, items: Item[]): Promise<Shipment> {
    return { trackingNumber: 'TRACK123', estimatedDelivery: new Date() };
  }
}

class NotificationSystem {
  async sendOrderConfirmation(email: string, trackingNumber: string): Promise<void> {}
}

// Facade that simplifies the complex subsystems
class OrderFacade {
  private inventory: InventorySystem;
  private payment: PaymentSystem;
  private shipping: ShippingSystem;
  private notification: NotificationSystem;
  
  constructor() {
    this.inventory = new InventorySystem();
    this.payment = new PaymentSystem();
    this.shipping = new ShippingSystem();
    this.notification = new NotificationSystem();
  }
  
  async createOrder(orderDetails: OrderDetails): Promise<OrderResult> {
    // Check availability
    const available = await this.inventory.checkAvailability(orderDetails.items);
    if (!available) {
      throw new Error('Items not available');
    }
    
    // Reserve items
    await this.inventory.reserveItems(orderDetails.items);
    
    try {
      // Process payment
      const paymentResult = await this.payment.processPayment(orderDetails.payment);
      if (!paymentResult.success) {
        await this.inventory.releaseItems(orderDetails.items);
        throw new Error('Payment failed');
      }
      
      // Create shipment
      const shipment = await this.shipping.createShipment(
        orderDetails.address,
        orderDetails.items
      );
      
      // Send notification
      await this.notification.sendOrderConfirmation(
        orderDetails.email,
        shipment.trackingNumber
      );
      
      return {
        success: true,
        orderId: `ORD-${Date.now()}`,
        trackingNumber: shipment.trackingNumber,
        transactionId: paymentResult.transactionId
      };
    } catch (error) {
      // Rollback on failure
      await this.inventory.releaseItems(orderDetails.items);
      throw error;
    }
  }
}

// Controller is now much simpler
class OrderController {
  constructor(private orderFacade: OrderFacade) {}
  
  async createOrder(req: Request, res: Response) {
    try {
      const result = await this.orderFacade.createOrder(req.body);
      res.json(result);
    } catch (error) {
      res.status(400).json({ error: error.message });
    }
  }
}
```

### Behavioral Patterns

#### Strategy Pattern
*Defines a family of algorithms and makes them interchangeable.*

**❌ Bad - Conditional logic everywhere:**
```typescript
class PriceCalculator {
  calculatePrice(basePrice: number, customerType: string): number {
    if (customerType === 'regular') {
      return basePrice;
    } else if (customerType === 'premium') {
      return basePrice * 0.9; // 10% discount
    } else if (customerType === 'vip') {
      return basePrice * 0.8; // 20% discount
    } else if (customerType === 'employee') {
      return basePrice * 0.5; // 50% discount
    }
    return basePrice;
  }
  
  // Adding new customer types requires modifying this class
}
```

**✅ Good - Strategy pattern:**
```typescript
interface PricingStrategy {
  calculatePrice(basePrice: number): number;
  getDiscountDescription(): string;
}

class RegularPricing implements PricingStrategy {
  calculatePrice(basePrice: number): number {
    return basePrice;
  }
  
  getDiscountDescription(): string {
    return 'No discount';
  }
}

class PremiumPricing implements PricingStrategy {
  calculatePrice(basePrice: number): number {
    return basePrice * 0.9; // 10% discount
  }
  
  getDiscountDescription(): string {
    return '10% member discount';
  }
}

class VIPPricing implements PricingStrategy {
  calculatePrice(basePrice: number): number {
    return basePrice * 0.8; // 20% discount
  }
  
  getDiscountDescription(): string {
    return '20% VIP discount';
  }
}

class SeasonalPricing implements PricingStrategy {
  constructor(private discountPercent: number) {}
  
  calculatePrice(basePrice: number): number {
    return basePrice * (1 - this.discountPercent / 100);
  }
  
  getDiscountDescription(): string {
    return `${this.discountPercent}% seasonal discount`;
  }
}

class PriceCalculator {
  constructor(private strategy: PricingStrategy) {}
  
  setStrategy(strategy: PricingStrategy): void {
    this.strategy = strategy;
  }
  
  calculatePrice(basePrice: number): number {
    return this.strategy.calculatePrice(basePrice);
  }
  
  getDiscountInfo(): string {
    return this.strategy.getDiscountDescription();
  }
}

// Usage - easily switch strategies
const calculator = new PriceCalculator(new RegularPricing());
console.log(calculator.calculatePrice(100)); // 100

calculator.setStrategy(new VIPPricing());
console.log(calculator.calculatePrice(100)); // 80

// Add holiday discount without modifying existing code
calculator.setStrategy(new SeasonalPricing(15));
console.log(calculator.calculatePrice(100)); // 85

// React example
function ProductPrice({ basePrice, customerType }: Props) {
  const strategyMap: Record<string, PricingStrategy> = {
    regular: new RegularPricing(),
    premium: new PremiumPricing(),
    vip: new VIPPricing()
  };
  
  const calculator = new PriceCalculator(strategyMap[customerType]);
  const finalPrice = calculator.calculatePrice(basePrice);
  
  return (
    <div>
      <span className="original-price">${basePrice}</span>
      <span className="final-price">${finalPrice}</span>
      <span className="discount-info">{calculator.getDiscountInfo()}</span>
    </div>
  );
}
```

#### Observer Pattern
*Defines a subscription mechanism to notify multiple objects about events.*

**❌ Bad - Tight coupling and manual notification:**
```typescript
class StockTracker {
  private price: number = 0;
  
  setPrice(price: number) {
    this.price = price;
    // Tightly coupled to specific notification methods
    this.sendEmailAlert(price);
    this.updateMobileApp(price);
    this.logPriceChange(price);
    // Adding new notifications requires modifying this class
  }
  
  private sendEmailAlert(price: number) { /* ... */ }
  private updateMobileApp(price: number) { /* ... */ }
  private logPriceChange(price: number) { /* ... */ }
}
```

**✅ Good - Observer pattern:**
```typescript
interface Observer {
  update(data: any): void;
}

interface Subject {
  attach(observer: Observer): void;
  detach(observer: Observer): void;
  notify(): void;
}

class StockTracker implements Subject {
  private observers: Observer[] = [];
  private price: number = 0;
  private symbol: string;
  
  constructor(symbol: string) {
    this.symbol = symbol;
  }
  
  attach(observer: Observer): void {
    const isExist = this.observers.includes(observer);
    if (!isExist) {
      this.observers.push(observer);
    }
  }
  
  detach(observer: Observer): void {
    const index = this.observers.indexOf(observer);
    if (index > -1) {
      this.observers.splice(index, 1);
    }
  }
  
  notify(): void {
    for (const observer of this.observers) {
      observer.update({
        symbol: this.symbol,
        price: this.price,
        timestamp: new Date()
      });
    }
  }
  
  setPrice(price: number): void {
    this.price = price;
    this.notify();
  }
  
  getPrice(): number {
    return this.price;
  }
}

class EmailAlertObserver implements Observer {
  constructor(private email: string, private threshold: number) {}
  
  update(data: any): void {
    if (data.price > this.threshold) {
      console.log(`Email to ${this.email}: ${data.symbol} exceeded $${this.threshold}`);
      // Send actual email
    }
  }
}

class MobileAppObserver implements Observer {
  constructor(private userId: string) {}
  
  update(data: any): void {
    console.log(`Push notification to user ${this.userId}: ${data.symbol} = $${data.price}`);
    // Send push notification
  }
}

class LoggerObserver implements Observer {
  update(data: any): void {
    console.log(`[${data.timestamp.toISOString()}] ${data.symbol}: $${data.price}`);
  }
}

class ChartObserver implements Observer {
  private dataPoints: any[] = [];
  
  update(data: any): void {
    this.dataPoints.push({ time: data.timestamp, price: data.price });
    console.log('Chart updated with new data point');
  }
  
  getChartData(): any[] {
    return this.dataPoints;
  }
}

// Usage
const appleStock = new StockTracker('AAPL');

const emailAlert = new EmailAlertObserver('investor@example.com', 150);
const mobileApp = new MobileAppObserver('user123');
const logger = new LoggerObserver();
const chart = new ChartObserver();

appleStock.attach(emailAlert);
appleStock.attach(mobileApp);
appleStock.attach(logger);
appleStock.attach(chart);

// All observers are automatically notified
appleStock.setPrice(148.50);
appleStock.setPrice(152.75); // Email alert triggered

// Can remove observers dynamically
appleStock.detach(emailAlert);
appleStock.setPrice(155.00); // Email not sent

// React example with hooks
function useStockObserver(tracker: StockTracker) {
  const [price, setPrice] = useState(0);
  
  useEffect(() => {
    const observer: Observer = {
      update: (data) => setPrice(data.price)
    };
    
    tracker.attach(observer);
    return () => tracker.detach(observer);
  }, [tracker]);
  
  return price;
}
```

#### Command Pattern
*Encapsulates a request as an object, allowing parameterization and queuing.*

**❌ Bad - Direct coupling to operations:**
```typescript
function TextEditor() {
  const [text, setText] = useState('');
  
  const handleBold = () => {
    setText(`<b>${text}</b>`);
    // No way to undo!
  };
  
  const handleItalic = () => {
    setText(`<i>${text}</i>`);
    // No way to undo!
  };
  
  // Cannot implement undo/redo
  // Cannot queue operations
  // Cannot save operation history
}
```

**✅ Good - Command pattern:**
```typescript
interface Command {
  execute(): void;
  undo(): void;
  getDescription(): string;
}

class TextDocument {
  private content: string = '';
  
  getContent(): string {
    return this.content;
  }
  
  setContent(content: string): void {
    this.content = content;
  }
  
  insert(text: string, position: number): void {
    this.content = 
      this.content.slice(0, position) + 
      text + 
      this.content.slice(position);
  }
  
  delete(start: number, end: number): void {
    this.content = 
      this.content.slice(0, start) + 
      this.content.slice(end);
  }
}

class InsertTextCommand implements Command {
  private previousContent: string;
  
  constructor(
    private document: TextDocument,
    private text: string,
    private position: number
  ) {
    this.previousContent = document.getContent();
  }
  
  execute(): void {
    this.document.insert(this.text, this.position);
  }
  
  undo(): void {
    this.document.setContent(this.previousContent);
  }
  
  getDescription(): string {
    return `Insert "${this.text}" at position ${this.position}`;
  }
}

class DeleteTextCommand implements Command {
  private deletedText: string;
  private previousContent: string;
  
  constructor(
    private document: TextDocument,
    private start: number,
    private end: number
  ) {
    this.previousContent = document.getContent();
    this.deletedText = document.getContent().slice(start, end);
  }
  
  execute(): void {
    this.document.delete(this.start, this.end);
  }
  
  undo(): void {
    this.document.setContent(this.previousContent);
  }
  
  getDescription(): string {
    return `Delete "${this.deletedText}"`;
  }
}

class BoldTextCommand implements Command {
  private previousContent: string;
  
  constructor(private document: TextDocument) {
    this.previousContent = document.getContent();
  }
  
  execute(): void {
    const content = this.document.getContent();
    this.document.setContent(`<b>${content}</b>`);
  }
  
  undo(): void {
    this.document.setContent(this.previousContent);
  }
  
  getDescription(): string {
    return 'Make text bold';
  }
}

class CommandHistory {
  private history: Command[] = [];
  private currentIndex: number = -1;
  
  execute(command: Command): void {
    // Remove any commands after current index (when undoing then doing new command)
    this.history = this.history.slice(0, this.currentIndex + 1);
    
    command.execute();
    this.history.push(command);
    this.currentIndex++;
  }
  
  undo(): boolean {
    if (this.currentIndex < 0) {
      return false;
    }
    
    const command = this.history[this.currentIndex];
    command.undo();
    this.currentIndex--;
    return true;
  }
  
  redo(): boolean {
    if (this.currentIndex >= this.history.length - 1) {
      return false;
    }
    
    this.currentIndex++;
    const command = this.history[this.currentIndex];
    command.execute();
    return true;
  }
  
  getHistory(): string[] {
    return this.history.map(cmd => cmd.getDescription());
  }
  
  canUndo(): boolean {
    return this.currentIndex >= 0;
  }
  
  canRedo(): boolean {
    return this.currentIndex < this.history.length - 1;
  }
}

// Usage
const document = new TextDocument();
const history = new CommandHistory();

// Execute commands
history.execute(new InsertTextCommand(document, 'Hello', 0));
console.log(document.getContent()); // "Hello"

history.execute(new InsertTextCommand(document, ' World', 5));
console.log(document.getContent()); // "Hello World"

history.execute(new BoldTextCommand(document));
console.log(document.getContent()); // "<b>Hello World</b>"

// Undo
history.undo();
console.log(document.getContent()); // "Hello World"

history.undo();
console.log(document.getContent()); // "Hello"

// Redo
history.redo();
console.log(document.getContent()); // "Hello World"

// React example
function TextEditor() {
  const [document] = useState(() => new TextDocument());
  const [history] = useState(() => new CommandHistory());
  const [content, setContent] = useState('');
  
  const executeCommand = (command: Command) => {
    history.execute(command);
    setContent(document.getContent());
  };
  
  const handleUndo = () => {
    if (history.undo()) {
      setContent(document.getContent());
    }
  };
  
  const handleRedo = () => {
    if (history.redo()) {
      setContent(document.getContent());
    }
  };
  
  return (
    <div>
      <div>
        <button onClick={handleUndo} disabled={!history.canUndo()}>
          Undo
        </button>
        <button onClick={handleRedo} disabled={!history.canRedo()}>
          Redo
        </button>
        <button onClick={() => executeCommand(new BoldTextCommand(document))}>
          Bold
        </button>
      </div>
      <textarea value={content} onChange={(e) => {
        executeCommand(new InsertTextCommand(document, e.target.value, 0));
      }} />
      <div>
        <h4>History:</h4>
        <ul>
          {history.getHistory().map((desc, i) => (
            <li key={i}>{desc}</li>
          ))}
        </ul>
      </div>
    </div>
  );
}
```

---

## 3. Code Smells & Refactoring

### Long Functions/Methods

**❌ Bad - 50+ line function doing too much:**
```typescript
async function processOrder(orderId: string) {
  // Fetch order
  const order = await db.orders.findById(orderId);
  if (!order) throw new Error('Order not found');
  
  // Validate order
  if (!order.items || order.items.length === 0) {
    throw new Error('Order has no items');
  }
  if (order.status !== 'pending') {
    throw new Error('Order already processed');
  }
  
  // Calculate totals
  let subtotal = 0;
  for (const item of order.items) {
    subtotal += item.price * item.quantity;
  }
  const tax = subtotal * 0.0825;
  const shipping = subtotal > 50 ? 0 : 5.99;
  const total = subtotal + tax + shipping;
  
  // Check inventory
  for (const item of order.items) {
    const stock = await db.inventory.findByProductId(item.productId);
    if (stock.quantity < item.quantity) {
      throw new Error(`Insufficient stock for ${item.name}`);
    }
  }
  
  // Reserve inventory
  for (const item of order.items) {
    await db.inventory.update(item.productId, {
      $inc: { quantity: -item.quantity }
    });
  }
  
  // Process payment
  const payment = await paymentGateway.charge({
    amount: total,
    source: order.paymentMethod
  });
  if (!payment.success) {
    // Rollback inventory
    for (const item of order.items) {
      await db.inventory.update(item.productId, {
        $inc: { quantity: item.quantity }
      });
    }
    throw new Error('Payment failed');
  }
  
  // Update order
  await db.orders.update(orderId, {
    status: 'confirmed',
    total,
    paymentId: payment.id
  });
  
  // Send confirmation email
  await emailService.send({
    to: order.email,
    subject: 'Order Confirmed',
    body: `Your order ${orderId} has been confirmed!`
  });
  
  return order;
}
```

**✅ Good - Extracted into focused functions:**
```typescript
async function processOrder(orderId: string): Promise<Order> {
  const order = await fetchAndValidateOrder(orderId);
  const totals = calculateOrderTotals(order);
  
  await validateInventory(order.items);
  await reserveInventory(order.items);
  
  try {
    const payment = await processPayment(totals.total, order.paymentMethod);
    await updateOrderStatus(orderId, 'confirmed', totals.total, payment.id);
    await sendOrderConfirmation(order.email, orderId);
    return order;
  } catch (error) {
    await rollbackInventory(order.items);
    throw error;
  }
}

async function fetchAndValidateOrder(orderId: string): Promise<Order> {
  const order = await db.orders.findById(orderId);
  if (!order) {
    throw new Error('Order not found');
  }
  if (!order.items || order.items.length === 0) {
    throw new Error('Order has no items');
  }
  if (order.status !== 'pending') {
    throw new Error('Order already processed');
  }
  return order;
}

interface OrderTotals {
  subtotal: number;
  tax: number;
  shipping: number;
  total: number;
}

function calculateOrderTotals(order: Order): OrderTotals {
  const subtotal = order.items.reduce((sum, item) => 
    sum + (item.price * item.quantity), 0
  );
  const tax = subtotal * 0.0825;
  const shipping = subtotal > 50 ? 0 : 5.99;
  const total = subtotal + tax + shipping;
  
  return { subtotal, tax, shipping, total };
}

async function validateInventory(items: OrderItem[]): Promise<void> {
  for (const item of items) {
    const stock = await db.inventory.findByProductId(item.productId);
    if (stock.quantity < item.quantity) {
      throw new Error(`Insufficient stock for ${item.name}`);
    }
  }
}

async function reserveInventory(items: OrderItem[]): Promise<void> {
  for (const item of items) {
    await db.inventory.update(item.productId, {
      $inc: { quantity: -item.quantity }
    });
  }
}

async function rollbackInventory(items: OrderItem[]): Promise<void> {
  for (const item of items) {
    await db.inventory.update(item.productId, {
      $inc: { quantity: item.quantity }
    });
  }
}

async function processPayment(amount: number, paymentMethod: string): Promise<Payment> {
  const payment = await paymentGateway.charge({
    amount,
    source: paymentMethod
  });
  if (!payment.success) {
    throw new Error('Payment failed');
  }
  return payment;
}

async function updateOrderStatus(
  orderId: string,
  status: string,
  total: number,
  paymentId: string
): Promise<void> {
  await db.orders.update(orderId, {
    status,
    total,
    paymentId
  });
}

async function sendOrderConfirmation(email: string, orderId: string): Promise<void> {
  await emailService.send({
    to: email,
    subject: 'Order Confirmed',
    body: `Your order ${orderId} has been confirmed!`
  });
}
```

### Deep Nesting

**❌ Bad - Deep nesting:**
```typescript
function processUserData(user: any) {
  if (user) {
    if (user.isActive) {
      if (user.subscription) {
        if (user.subscription.plan === 'premium') {
          if (user.subscription.expiresAt > Date.now()) {
            if (user.preferences) {
              if (user.preferences.notifications) {
                return sendPremiumNotification(user);
              }
            }
          }
        }
      }
    }
  }
  return null;
}
```

**✅ Good - Early returns and guard clauses:**
```typescript
function processUserData(user: any) {
  if (!user) return null;
  if (!user.isActive) return null;
  if (!user.subscription) return null;
  if (user.subscription.plan !== 'premium') return null;
  if (user.subscription.expiresAt <= Date.now()) return null;
  if (!user.preferences?.notifications) return null;
  
  return sendPremiumNotification(user);
}

// Even better - extract validation
function isPremiumUserWithNotifications(user: any): boolean {
  return !!(
    user?.isActive &&
    user.subscription?.plan === 'premium' &&
    user.subscription.expiresAt > Date.now() &&
    user.preferences?.notifications
  );
}

function processUserData(user: any) {
  if (!isPremiumUserWithNotifications(user)) {
    return null;
  }
  return sendPremiumNotification(user);
}
```

### Duplicate Code (DRY Violations)

**❌ Bad - Repeated logic:**
```typescript
function createPost(data: PostData) {
  if (!data.title || data.title.length < 5) {
    throw new Error('Title must be at least 5 characters');
  }
  if (!data.content || data.content.length < 20) {
    throw new Error('Content must be at least 20 characters');
  }
  // ... create post
}

function updatePost(id: string, data: PostData) {
  if (!data.title || data.title.length < 5) {
    throw new Error('Title must be at least 5 characters');
  }
  if (!data.content || data.content.length < 20) {
    throw new Error('Content must be at least 20 characters');
  }
  // ... update post
}

function createComment(data: CommentData) {
  if (!data.content || data.content.length < 20) {
    throw new Error('Content must be at least 20 characters');
  }
  // ... create comment
}
```

**✅ Good - Extract common logic:**
```typescript
function validateMinLength(value: string, minLength: number, fieldName: string): void {
  if (!value || value.length < minLength) {
    throw new Error(`${fieldName} must be at least ${minLength} characters`);
  }
}

function validatePostData(data: PostData): void {
  validateMinLength(data.title, 5, 'Title');
  validateMinLength(data.content, 20, 'Content');
}

function createPost(data: PostData) {
  validatePostData(data);
  // ... create post
}

function updatePost(id: string, data: PostData) {
  validatePostData(data);
  // ... update post
}

function createComment(data: CommentData) {
  validateMinLength(data.content, 20, 'Content');
  // ... create comment
}
```

### God Objects/Classes

**❌ Bad - Class doing everything:**
```typescript
class UserManager {
  // Authentication
  login(email: string, password: string) { }
  logout(userId: string) { }
  resetPassword(email: string) { }
  
  // User CRUD
  createUser(data: any) { }
  updateUser(id: string, data: any) { }
  deleteUser(id: string) { }
  
  // Permissions
  checkPermission(userId: string, action: string) { }
  grantPermission(userId: string, permission: string) { }
  
  // Profile
  updateProfile(userId: string, profile: any) { }
  uploadAvatar(userId: string, file: File) { }
  
  // Notifications
  sendNotification(userId: string, message: string) { }
  getNotifications(userId: string) { }
  
  // Analytics
  trackUserActivity(userId: string, action: string) { }
  getUserStats(userId: string) { }
  
  // ... and more!
}
```

**✅ Good - Split into focused classes:**
```typescript
class AuthenticationService {
  login(email: string, password: string): Promise<Session> { return {} as Session; }
  logout(userId: string): Promise<void> { }
  resetPassword(email: string): Promise<void> { }
  verifyToken(token: string): Promise<boolean> { return true; }
}

class UserRepository {
  create(data: UserData): Promise<User> { return {} as User; }
  findById(id: string): Promise<User | null> { return null; }
  update(id: string, data: Partial<UserData>): Promise<User> { return {} as User; }
  delete(id: string): Promise<void> { }
}

class PermissionService {
  check(userId: string, action: string): Promise<boolean> { return true; }
  grant(userId: string, permission: string): Promise<void> { }
  revoke(userId: string, permission: string): Promise<void> { }
}

class UserProfileService {
  update(userId: string, profile: ProfileData): Promise<Profile> { return {} as Profile; }
  uploadAvatar(userId: string, file: File): Promise<string> { return ''; }
}

class NotificationService {
  send(userId: string, message: string): Promise<void> { }
  getForUser(userId: string): Promise<Notification[]> { return []; }
  markAsRead(notificationId: string): Promise<void> { }
}

class UserAnalyticsService {
  track(userId: string, event: string, properties?: any): void { }
  getStats(userId: string): Promise<UserStats> { return {} as UserStats; }
}
```

### Magic Numbers

**❌ Bad - Unexplained numbers:**
```typescript
function calculateDiscount(price: number, customerType: string): number {
  if (customerType === 'vip') {
    return price * 0.8;
  }
  if (price > 100) {
    return price * 0.95;
  }
  return price;
}

function isValidPassword(password: string): boolean {
  return password.length >= 8 && /[A-Z]/.test(password) && /[0-9]/.test(password);
}

setTimeout(() => {
  checkStatus();
}, 300000);
```

**✅ Good - Named constants:**
```typescript
const DISCOUNT = {
  VIP: 0.20,           // 20% discount for VIP customers
  BULK_ORDER: 0.05,     // 5% discount for orders over threshold
} as const;

const ORDER_THRESHOLD = {
  BULK_ORDER_MINIMUM: 100,  // Minimum amount for bulk discount
  FREE_SHIPPING: 50,        // Free shipping threshold
} as const;

function calculateDiscount(price: number, customerType: string): number {
  if (customerType === 'vip') {
    return price * (1 - DISCOUNT.VIP);
  }
  if (price > ORDER_THRESHOLD.BULK_ORDER_MINIMUM) {
    return price * (1 - DISCOUNT.BULK_ORDER);
  }
  return price;
}

const PASSWORD_REQUIREMENTS = {
  MIN_LENGTH: 8,
  REQUIRES_UPPERCASE: true,
  REQUIRES_NUMBER: true,
} as const;

function isValidPassword(password: string): boolean {
  return (
    password.length >= PASSWORD_REQUIREMENTS.MIN_LENGTH &&
    /[A-Z]/.test(password) &&
    /[0-9]/.test(password)
  );
}

const STATUS_CHECK_INTERVAL_MS = 5 * 60 * 1000; // 5 minutes

setTimeout(() => {
  checkStatus();
}, STATUS_CHECK_INTERVAL_MS);
```

---

## 4. Clean Code Principles

### Naming Conventions

**❌ Bad names:**
```typescript
function f(x: any) { return x * 2; }
const d = new Date();
const temp = getUserData();
let flag = true;
const arr = [1, 2, 3];
const data = { n: 'John', a: 25 };
```

**✅ Good names:**
```typescript
function doubleValue(value: number): number { 
  return value * 2; 
}

const currentDate = new Date();
const userData = getUserData();
const isUserActive = true;
const userIds = [1, 2, 3];
const user = { name: 'John', age: 25 };

// Function names should be verbs
function calculateTotal() { }
function fetchUserData() { }
function isValidEmail() { }
function handleSubmit() { }

// Boolean variables should be questions
const isLoading = false;
const hasPermission = true;
const canEdit = false;
const shouldRetry = true;

// Classes should be nouns
class UserRepository { }
class PaymentProcessor { }
class EmailValidator { }
```

### Function Size & Composition

**❌ Bad - Large functions:**
```typescript
function registerUser(email: string, password: string, name: string) {
  // Validation (20 lines)
  // Database operations (15 lines)
  // Email sending (10 lines)
  // Analytics (5 lines)
  // ... 50+ lines total
}
```

**✅ Good - Small, focused functions:**
```typescript
async function registerUser(
  email: string,
  password: string,
  name: string
): Promise<User> {
  validateRegistrationData(email, password, name);
  const user = await createUserInDatabase(email, password, name);
  await sendWelcomeEmail(user);
  trackUserRegistration(user.id);
  return user;
}

function validateRegistrationData(
  email: string,
  password: string,
  name: string
): void {
  if (!isValidEmail(email)) {
    throw new Error('Invalid email format');
  }
  if (!isStrongPassword(password)) {
    throw new Error('Password too weak');
  }
  if (!name || name.length < 2) {
    throw new Error('Name too short');
  }
}

// Each function does one thing well
// Easy to read, test, and maintain
```

### Comment Best Practices

**❌ Bad comments:**
```typescript
// Loop through users
for (const user of users) {
  // Check if user is active
  if (user.isActive) {
    // Update user
    user.lastSeen = new Date();
  }
}

// Increment i
i++;

// This is the user controller
class UserController { }

// TODO: Fix this later (written 2 years ago)
// HACK: Don't touch this!!!
// Magic code, don't ask me how it works
```

**✅ Good comments:**
```typescript
// Comments explain WHY, not WHAT

// User must be active for at least 30 days before they can post reviews
// This prevents spam from newly created bot accounts
if (user.isActive && daysSinceRegistration(user) >= 30) {
  allowReviewPosting(user);
}

// Binary search requires sorted input. Performance: O(log n)
function binarySearch(arr: number[], target: number): number { }

/**
 * Calculates compound interest using the formula: A = P(1 + r/n)^(nt)
 * 
 * @param principal - Initial investment amount
 * @param rate - Annual interest rate (e.g., 0.05 for 5%)
 * @param years - Investment period in years
 * @param compoundFrequency - Number of times interest compounds per year
 * @returns Final amount after compound interest
 */
function calculateCompoundInterest(
  principal: number,
  rate: number,
  years: number,
  compoundFrequency: number = 12
): number {
  return principal * Math.pow(1 + rate / compoundFrequency, compoundFrequency * years);
}

// WORKAROUND: API returns null instead of empty array before v2.0
// Remove this check after migrating to API v2.0 (scheduled for Q2 2026)
const items = response.items ?? [];

// Self-documenting code needs fewer comments
function isEligibleForPremiumDiscount(user: User): boolean {
  const hasActiveSubscription = user.subscription?.status === 'active';
  const isLongTermCustomer = user.memberSince < oneYearAgo();
  return hasActiveSubscription && isLongTermCustomer;
}
```

### Error Handling

**❌ Bad error handling:**
```typescript
// Silently swallowing errors
try {
  await saveData(data);
} catch (e) {
  // Do nothing - data is lost!
}

// Catching everything
try {
  processPayment();
} catch (error) {
  console.log('Something went wrong');
}

// No error context
throw new Error('Failed');
```

**✅ Good error handling:**
```typescript
// Custom error types with context
class ValidationError extends Error {
  constructor(
    message: string,
    public field: string,
    public value: any
  ) {
    super(message);
    this.name = 'ValidationError';
  }
}

class PaymentError extends Error {
  constructor(
    message: string,
    public transactionId: string,
    public amount: number
  ) {
    super(message);
    this.name = 'PaymentError';
  }
}

// Specific error handling
async function processOrder(order: Order): Promise<OrderResult> {
  try {
    validateOrder(order);
    const payment = await processPayment(order);
    return { success: true, orderId: order.id };
  } catch (error) {
    if (error instanceof ValidationError) {
      logger.warn('Order validation failed', {
        field: error.field,
        value: error.value,
        orderId: order.id
      });
      return { success: false, error: 'Invalid order data' };
    }
    
    if (error instanceof PaymentError) {
      logger.error('Payment processing failed', {
        transactionId: error.transactionId,
        amount: error.amount,
        orderId: order.id
      });
      await refundOrder(order);
      return { success: false, error: 'Payment failed' };
    }
    
    // Unexpected errors are re-thrown
    logger.error('Unexpected error processing order', { error, orderId: order.id });
    throw error;
  }
}

// Result type pattern (alternative to throwing errors)
type Result<T, E = Error> = 
  | { success: true; value: T }
  | { success: false; error: E };

async function createUser(data: UserData): Promise<Result<User, string>> {
  if (!isValidEmail(data.email)) {
    return { success: false, error: 'Invalid email' };
  }
  
  try {
    const user = await db.users.create(data);
    return { success: true, value: user };
  } catch (error) {
    return { success: false, error: 'Database error' };
  }
}

// Usage
const result = await createUser(userData);
if (result.success) {
  console.log('User created:', result.value);
} else {
  console.error('Failed to create user:', result.error);
}
```

### DRY, KISS, YAGNI

**DRY (Don't Repeat Yourself):**
```typescript
// ❌ Bad - Repetition
const fullName1 = `${user.firstName} ${user.lastName}`;
const fullName2 = `${author.firstName} ${author.lastName}`;
const fullName3 = `${admin.firstName} ${admin.lastName}`;

// ✅ Good
function getFullName(person: { firstName: string; lastName: string }): string {
  return `${person.firstName} ${person.lastName}`;
}
```

**KISS (Keep It Simple, Stupid):**
```typescript
// ❌ Bad - Overly complex
function isEven(n: number): boolean {
  return n % 2 === 0 ? true : n % 2 === 1 ? false : true;
}

// ✅ Good
function isEven(n: number): boolean {
  return n % 2 === 0;
}
```

**YAGNI (You Aren't Gonna Need It):**
```typescript
// ❌ Bad - Building features you don't need yet
class User {
  constructor(
    public name: string,
    public email: string,
    // Don't add these until needed:
    public preferences?: UserPreferences,
    public settings?: UserSettings,
    public metadata?: any,
    public tags?: string[],
    public customFields?: Record<string, any>
  ) {}
}

// ✅ Good - Start simple, add when needed
class User {
  constructor(
    public name: string,
    public email: string
  ) {}
}
```

---

## 5. Clean Architecture

### Layered Architecture

```typescript
// ❌ Bad - No separation of concerns
export default function UserPage() {
  const [users, setUsers] = useState([]);
  
  useEffect(() => {
    // Direct database/API calls in UI component
    fetch('https://api.example.com/users')
      .then(res => res.json())
      .then(data => {
        // Business logic in UI
        const filtered = data.filter(u => u.age > 18);
        setUsers(filtered);
      });
  }, []);
  
  return <div>{users.map(u => <div>{u.name}</div>)}</div>;
}

// ✅ Good - Layered architecture
// Domain Layer - Pure business logic
interface User {
  id: string;
  name: string;
  age: number;
}

function isAdult(user: User): boolean {
  return user.age >= 18;
}

// Data Layer - API/Database interactions
class UserRepository {
  async findAll(): Promise<User[]> {
    const response = await fetch('https://api.example.com/users');
    return response.json();
  }
}

// Service Layer - Application logic
class UserService {
  constructor(private repository: UserRepository) {}
  
  async getAdultUsers(): Promise<User[]> {
    const users = await this.repository.findAll();
    return users.filter(isAdult);
  }
}

// Presentation Layer - UI only
export default function UserPage() {
  const [users, setUsers] = useState<User[]>([]);
  const userService = useMemo(() => 
    new UserService(new UserRepository()), []
  );
  
  useEffect(() => {
    userService.getAdultUsers().then(setUsers);
  }, [userService]);
  
  return (
    <div>
      {users.map(user => (
        <UserCard key={user.id} user={user} />
      ))}
    </div>
  );
}
```

### Dependency Flow Rules

```typescript
// ❌ Bad - Inner layers depend on outer layers
class UserService {
  async getUser(id: string) {
    // Service layer depends on Express (framework detail)
    const req = getCurrentRequest();
    return await fetch(`/api/users/${id}`);
  }
}

// ✅ Good - Dependencies point inward
// Core domain (innermost layer)
interface User {
  id: string;
  name: string;
}

// Application layer
interface UserRepository {
  findById(id: string): Promise<User | null>;
}

class UserService {
  constructor(private repository: UserRepository) {}
  
  async getUser(id: string): Promise<User | null> {
    return this.repository.findById(id);
  }
}

// Infrastructure layer (outermost)
class ApiUserRepository implements UserRepository {
  async findById(id: string): Promise<User | null> {
    const response = await fetch(`/api/users/${id}`);
    return response.json();
  }
}

// Dependency injection at the edges
const repository = new ApiUserRepository();
const userService = new UserService(repository);
```

---

## 6. Code Review Guidelines

### What to Look For

**✅ Code Review Checklist:**

1. **Correctness:**
   - Does it work as intended?
   - Are edge cases handled?
   - Are there logical errors?

2. **Testing:**
   - Are there adequate tests?
   - Do tests cover edge cases?
   - Are tests readable and maintainable?

3. **Design:**
   - Does it follow SOLID principles?
   - Is the code DRY?
   - Are there appropriate abstractions?

4. **Readability:**
   - Are names descriptive?
   - Is the code self-documenting?
   - Are comments helpful (not redundant)?

5. **Security:**
   - Input validation present?
   - No SQL injection vulnerabilities?
   - Secrets not hardcoded?
   - Authentication/authorization proper?

6. **Performance:**
   - No unnecessary loops or queries?
   - Appropriate data structures used?
   - No memory leaks?

7. **Error Handling:**
   - Errors handled appropriately?
   - Error messages helpful?
   - No silent failures?

---

## Checklist

### Before Committing Code

**Design & Architecture:**
- [ ] Each class/module has a single responsibility
- [ ] Code is open for extension, closed for modification
- [ ] Dependencies point toward abstractions, not concretions
- [ ] No inappropriate intimacy between modules
- [ ] Appropriate design patterns used (not over-engineered)

**Code Quality:**
- [ ] Functions are small (< 20 lines ideally)
- [ ] Functions do one thing and do it well
- [ ] No deep nesting (max 2-3 levels)
- [ ] No code duplication (DRY principle)
- [ ] No magic numbers (use named constants)
- [ ] No dead code or commented-out code

**Naming:**
- [ ] Variables, functions, classes have descriptive names
- [ ] Boolean variables are named as questions (isActive, hasPermission)
- [ ] Functions are named as verbs (fetchUser, calculateTotal)
- [ ] Classes are named as nouns (UserService, PaymentProcessor)

**Error Handling:**
- [ ] All errors are caught and handled appropriately
- [ ] Error messages are descriptive and actionable
- [ ] No silent failures
- [ ] Custom error types for different error categories

**Testing:**
- [ ] Unit tests cover critical logic
- [ ] Edge cases are tested
- [ ] Tests are readable and maintainable
- [ ] Mocks are used appropriately
- [ ] Test names describe what is being tested

**Security:**
- [ ] All user input is validated and sanitized
- [ ] No secrets in code (use environment variables)
- [ ] SQL injection prevented (parameterized queries)
- [ ] XSS vulnerabilities addressed
- [ ] Authentication and authorization implemented correctly

**Performance:**
- [ ] No N+1 query problems
- [ ] Appropriate indexes on database queries
- [ ] No unnecessary loops or iterations
- [ ] Efficient data structures used
- [ ] No memory leaks

**Documentation:**
- [ ] Complex logic is commented (explains WHY, not WHAT)
- [ ] Public APIs have JSDoc/TSDoc comments
- [ ] README updated if needed
- [ ] Breaking changes documented

**Code Review:**
- [ ] Code has been self-reviewed
- [ ] Tests pass locally
- [ ] Linting passes
- [ ] Type checking passes (TypeScript)
- [ ] No console.log or debugging code left in
- [ ] Commit messages are clear and descriptive
