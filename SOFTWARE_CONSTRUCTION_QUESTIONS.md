# 🔍 Software Construction & Development - Critical Questions

## Based on Fresh Grocery Project Analysis

---

## 1️⃣ ARCHITECTURE & DESIGN PATTERNS

### Q1.1: Monolithic vs Microservices
**Question**: The entire application is built as a monolith with one backend serving all features (auth, products, cart, orders, staff). 

- **Why not microservices** for independent scaling of high-traffic services (like products/cart)?
- **Risk**: If one feature crashes, does the entire API go down?
- **Question**: What's the expected concurrent user load? At what point should this be refactored to microservices?
- **Consideration**: Effort vs benefit analysis - is monolith justified for current/expected scale?

---

### Q1.2: Frontend Framework Choice
**Question**: Frontend uses vanilla JavaScript with no framework (React, Vue, Angular).

- **Why this decision?** (Pros: no build step, smaller bundle; Cons: poor state management, harder maintenance as app grows)
- **How will you manage state as the frontend scales?** Currently using localStorage and direct DOM manipulation.
- **Code reusability**: Lots of repeated code in `main.js` (product fetching, pagination). Should this be refactored into reusable modules/components?
- **Long-term maintainability**: How will new developers onboard without a structured framework?

---

### Q1.3: Separation of Concerns
**Question**: Looking at `frontend/assets/js/main.js` (~400 lines):

- Multiple concerns mixed: API service, Cart management, Product display, Authentication
- Should there be separate modules like:
  - `apiService.js` (already exists, but intertwined)
  - `cartManager.js` (separate from main.js?)
  - `productManager.js` (product fetching/display)
  - `authManager.js` (authentication logic)?
- **Benefits**: Easy to test, less coupling, clearer dependencies
- **Current state**: Is this maintainable as the app grows?

---

## 2️⃣ CODE QUALITY & MAINTAINABILITY

### Q2.1: Lack of Unit Tests
**Question**: No test files found in the project (no `.test.js`, `.spec.js`, or test directory).

- **How do you verify** features work without automated tests?
- **Regression risk**: When updating cart or order logic, how do you ensure nothing breaks?
- **Code confidence**: How can you refactor with confidence?
- **Questions**:
  - Should there be tests for: `cartManager.addToCart()`? `updateQuantity()`? `ProductController.getProducts()`?
  - What testing framework would suit this project? (Jest, Mocha, Jasmine?)
  - Coverage target: 80%? 90%?

---

### Q2.2: Error Handling Consistency
**Question**: Error handling differs across files:

- Some endpoints return `{ status: 'error', message: '...' }`
- Some return `{ error: '...' }` or just `error.message`
- Frontend sometimes ignores backend errors
- **Questions**:
  - Should all errors follow ONE format (status, message, errorCode)?
  - Should there be an error code system? (E001 - "Invalid email format", E002 - "Stock unavailable")
  - How do you handle API errors on the frontend consistently?
  - What about network timeouts? Retry logic?

---

### Q2.3: Magic Strings & Numbers
**Question**: Constants are scattered throughout code:

```javascript
// In various files:
TAX_RATE = 0.15          // Why 15% tax?
DELIVERY_CHARGE = 150    // Why Rs. 150?
JWT_EXPIRY = '7d'        // Why 7 days?
String(productId).length >= 12  // Why check length?
```

- **Should these be centralized** in a `constants.js` file?
- **Who decides these values?** Business? Product owner?
- **How do you change them** without finding all occurrences?
- **Questions**:
  - Are there config files for business rules?
  - How do admins change these without code changes?

---

### Q2.4: Null/Undefined Handling
**Question**: Inconsistent null safety:

```javascript
// In cart.js - sometimes safe:
if (!cartManager.cart || cartManager.cart.length === 0)

// But in main.js - sometimes risky:
const featured = sampleProducts.slice(0, 4);  // What if undefined?
item.product_id?.category  // Optional chaining used
item.product_name || 'Unknown'  // Fallback used

// And in backend - sometimes risky:
const category = String(p.category).toLowerCase();  // What if null?
```

- **Should there be a defensive programming standard?**
- **Suggestion**: Use optional chaining (`?.`) and nullish coalescing (`??`) consistently
- **Question**: Should data validation happen at model/API level?

---

## 3️⃣ ERROR HANDLING & VALIDATION

### Q3.1: Frontend Validation
**Question**: Data validation happens in multiple places:

```javascript
// Frontend (client-side):
if (newQuantity > 999) { show error }

// Backend (server-side):
if (qty > 999) { return 401 }

// Database (model level):
quantity: { min: 1 }
```

- **Redundancy or necessary?** (Yes, but confusing)
- **Single source of truth**: Should validation rules be in one place?
- **Questions**:
  - If frontend and backend disagree (different max values), which wins?
  - How do you maintain consistency across layers?
  - What about async validation (email already exists)?

---

### Q3.2: API Input Validation
**Question**: Looking at backend endpoints:

```javascript
// Sometimes detail validation:
if (!product_id || !quantity) return 400

// Sometimes minimal:
const { product_id, quantity } = req.body;
// No validation - could crash if quantity is string, array, etc.
```

- **Should there be a validation middleware** like `joi` or `express-validator`?
- **Benefit**: Centralized, reusable, testable validation logic
- **Current risk**: SQL injection? XSS? Type coercion bugs?

---

### Q3.3: Error Recovery Flow
**Question**: When operations fail (network error, stock unavailable):

**Frontend example**: 
```javascript
// Cart operation fails
await cartManager.updateQuantity(productId, newQuantity)
// Then what? UI state inconsistent? Old value still displayed?
```

**Backend example**:
```javascript
// Order creation fails at step 3 of 5
// Are previous steps rolled back? Transaction support?
```

- **Questions**:
  - Are there database transactions for multi-step operations?
  - How do you recover from partial failures?
  - What's the rollback strategy?

---

## 4️⃣ SECURITY CONSIDERATIONS

### Q4.1: Password Security
**Question**: Passwords are hashed with bcryptjs. Good!

```javascript
// model hook:
schema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcryptjs.hash(this.password, 10);
  next();
});
```

- **Questions**:
  - What about password requirements? (min length, complexity?)
  - Should there be rate limiting on login? (brute force protection)
  - Password reset flow - is it secure? (email verification? one-time links?)
  - Are passwords **never** logged or exposed in errors?

---

### Q4.2: CORS Configuration
**Question**: CORS is enabled:

```javascript
app.use(cors());  // Allows ANY domain to call your API!
```

- **This is risky!** Only frontend should access the API.
- **Should be**: 
  ```javascript
  app.use(cors({
    origin: 'http://localhost:8000',  // or production domain
    credentials: true
  }));
  ```
- **Questions**:
  - In production, which domains should be allowed?
  - Are credentials (cookies/auth) only sent to trusted domains?
  - Have you tested CORS attacks?

---

### Q4.3: SQL Injection / NoSQL Injection
**Question**: Using Mongoose with parameterized queries (good!):

```javascript
const cart = await Cart.findOne({ user_id: userId });  // Safe - parameterized
```

- **But frontend has risky code**:
  ```javascript
  const url = `${API_BASE_URL}/products/search/${query}`;  // Could inject?
  ```
- **Questions**:
  - Is the search query validated on backend?
  - What if someone passes `"'; drop table products; --"`?
  - Should there be input sanitization?

---

### Q4.4: XSS (Cross-Site Scripting)
**Question**: User-generated content (product names, reviews) displayed without sanitization:

```javascript
// Frontend renders:
<div class="product-name">${product.name}</div>
// If product.name = "<img onerror='alert(1)'>", it could execute!
```

- **Alert**: Template literals (`${}`) bypass DOM API safety!
- **Questions**:
  - Should use `.textContent` instead of innerHTML?
  - Should input be sanitized on backend (remove tags)?
  - Should there be a sanitization library (DOMPurify)?

---

### Q4.5: CSRF (Cross-Site Request Forgery)
**Question**: No CSRF protection visible.

- **Risk**: Attacker could trick user into making unwanted requests
- **JWT helps**, but only if SameSite cookie flags are set
- **Questions**:
  - Are tokens stored in HttpOnly cookies or localStorage?
  - If localStorage, is CSRF protection needed?
  - What about state-changing operations (DELETE, PUT)?

---

## 5️⃣ TESTING & QUALITY ASSURANCE

### Q5.1: Test Coverage
**Question**: No tests found. How do you verify:

- ✓ Cart operations (add/remove/update) work correctly?
- ✓ Stock validation prevents overbooking?
- ✓ Order calculations are accurate?
- ✓ Authentication middleware blocks unauthorized users?
- ✓ Role-based access works (admin vs customer)?

**Suggested tests**:
```javascript
// Jest example
describe('CartManager', () => {
  test('add product increases quantity if exists', () => { ... });
  test('quantity cannot exceed 999', () => { ... });
  test('total calculated correctly', () => { ... });
});

describe('Cart API', () => {
  test('get cart requires authentication', () => { ... });
  test('cannot add more than stock available', () => { ... });
});
```

---

### Q5.2: Manual Testing Burden
**Question**: Without tests, manual testing required for:

- Every cart operation
- Every role permission
- Every error scenario
- Every browser compatibility
- Every mobile device size

- **Questions**:
  - How long does QA take?
  - How many bugs escape to production?
  - How confident are you in refactoring?

---

### Q5.3: Frontend Testing
**Question**: How do you test frontend without a framework?

- No component testing
- No state management testing
- No integration testing

**Suggestion**: Consider:
- **End-to-end tests**: Selenium, Playwright, Cypress
- **Visual regression tests**: Percy, BackstopJS
- **Performance tests**: Lighthouse, WebPageTest

---

## 6️⃣ PERFORMANCE & SCALABILITY

### Q6.1: Database Query Optimization
**Question**: Are there N+1 query problems?

```javascript
// Example - Does this happen?
const carts = await Cart.find();
for (let cart of carts) {
  const user = await User.findById(cart.user_id);  // LOOP QUERY!
}
// This is N+1: 1 cart query + N user queries
```

**Better**:
```javascript
const carts = await Cart.find().populate('user_id');  // Single query
```

- **Questions**:
  - Are all relationships properly `.populate()`d?
  - Any missing database indexes?
  - Are queries paginated for large datasets?

---

### Q6.2: API Response Size
**Question**: How large are API responses?

```javascript
// Does getProducts return everything?
const products = await Product.find();  // Could be 1000+ items!
```

- **Should include**:
  - Pagination (10-50 items per page)
  - Field projection (select only needed fields)
  - Lazy loading for images

---

### Q6.3: Frontend Performance
**Question**: 

- **DOM Manipulation**: Direct manipulation vs virtual DOM diffing?
- **Re-renders**: Does `displayCart()` update entire list or just changed items?
- **Asset loading**: Are images lazy-loaded?
- **Bundle size**: Lots of inline JavaScript - how large is main.js?
- **Caching**: Are responses cached locally?

---

### Q6.4: Rate Limiting
**Question**: Express-rate-limit is imported but is it configured?

```javascript
const rateLimit = require('express-rate-limit');
// But is it used on endpoints?

// Should protect against:
app.use('/api/', rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100  // 100 requests per window
}));
```

---

## 7️⃣ DATABASE DESIGN

### Q7.1: Denormalization
**Question**: In cart items, product data is denormalized:

```javascript
// Instead of just product_id reference:
{
  product_id: ObjectId,
  product_name: "Apples",      // Denormalized
  price: 150                    // Denormalized
}

// Questions:
// - What if product name or price changes?
// - Should cart show old price or new price?
// - Is this intentional? (Preserve order pricing)
```

- **Trade-off**: Faster reads but update consistency issues
- **Questions**:
  - Is this design decision documented?
  - How do you handle price changes?

---

### Q7.2: Schema Validation
**Question**: Mongoose schemas have validation:

```javascript
// Good:
quantity: { type: Number, required: true, min: 1 }

// But any missing on text fields?
name: { type: String, required: true, minlength: 2, maxlength: 100 }
```

- **Questions**:
  - Are email fields validated with regex?
  - Are phone fields validated?
  - Are there custom validators (email unique check, etc.)?

---

### Q7.3: Indexes
**Question**: Performance optimization - are indexes used?

```javascript
// Should have indexes on:
cartSchema.index({ user_id: 1 });          // Query by user
orderSchema.index({ user_id: 1, created_at: -1 });  // Recent user orders
productSchema.index({ category: 1, price: 1 });     // Filter by category+price
```

- **Questions**:
  - What queries are slow? (Use MongoDB profiler)
  - Are there compound indexes?
  - Are unused indexes removed?

---

## 8️⃣ API DESIGN

### Q8.1: RESTful Standards
**Question**: API follows REST somewhat, but could be better:

```javascript
// Good:
GET /api/products              // List
POST /api/cart/add             // But should be POST /api/cart + body

// Inconsistent:
POST /cart/remove              // But could be DELETE /cart/{id}
PUT /cart/update               // Good

// Verb-based (not REST):
POST /products/search/apple    // Should be GET /products?search=apple
```

- **Should be**:
  ```javascript
  GET    /api/products           // List all
  POST   /api/products           // Create
  GET    /api/products/:id       // Get one
  PUT    /api/products/:id       // Update
  DELETE /api/products/:id       // Delete

  GET    /api/cart               // Get cart
  POST   /api/cart               // Add item (or POST /api/cart/items)
  PUT    /api/cart/items/:id     // Update quantity
  DELETE /api/cart/items/:id     // Remove item
  ```

- **Questions**:
  - Is current design intentional for client convenience?
  - API versioning: What if breaking changes needed? (/v2/api?)

---

### Q8.2: Pagination Implementation
**Question**: Pagination used, but is it standard?

```javascript
// Response format:
{
  status: 'success',
  data: [...],
  pagination: { page, limit, total, pages }
}

// Questions:
// - Should use Link headers? (HTTP standard)
// - Should support cursor-based pagination for large datasets?
// - Default limit: too high? (could be DOS vector)
```

---

### Q8.3: API Versioning Strategy
**Question**: Currently no versioning (`/api/` vs `/api/v1/`).

- **What if breaking changes needed?**
  - Add new field (backward compatible)
  - Remove field (breaking - old clients break)
  - Change response format (breaking)

- **Future-proofing**: Should the API be versioned now?

---

## 9️⃣ FRONTEND ARCHITECTURE

### Q9.1: State Management
**Question**: State stored in multiple places:

```javascript
// 1. localStorage
localStorage.getItem('token')
localStorage.getItem('cart')

// 2. Variables
let sampleProducts = []
let wishlist = []

// 3. DOM
document.getElementById('cartCount').textContent

// 4. Backend
POST /api/cart/add → updates database
```

- **Questions**:
  - Source of truth: localStorage or backend?
  - What if desync between local and backend?
  - How do you handle concurrent changes?
  - Mobile app reuse possible?

---

### Q9.2: Code Organization
**Question**: Frontend files:

```
frontend/
├── assets/
│   ├── css/
│   │   ├── admin.css          (specific)
│   │   └── style.css          (general)
│   └── js/
│       ├── main.js            (~400 lines, does everything!)
│       ├── cart.js            (specific)
│       ├── api.js
│       ├── cartController.js  (unused?)
│       ├── products.js        (?)
│       └── ...
└── pages/
    ├── index.html
    ├── cart.html
    ├── products.html
    └── ...
```

- **Questions**:
  - What does each file do?
  - Why are there unused files (cartController.js)?
  - Should there be a shared utils folder?
  - Documentation for developers?

---

### Q9.3: Browser Compatibility
**Question**: Code uses modern JavaScript:

```javascript
// ES6+ features:
const, arrow functions, destructuring, template literals, async/await
// Sets, Maps, optional chaining, nullish coalescing
```

- **Questions**:
  - Need to support IE11? (Probably not - but what's the minimum?)
  - Are you transpiling (Babel)?
  - Polyfills needed?

---

## 🔟 DEPLOYMENT & DEVOPS

### Q10.1: Environment Configuration
**Question**: `.env` file structure:

```env
PORT=5000
NODE_ENV=development
MONGODB_URI=mongodb://localhost:27017/...
JWT_SECRET=your-secret-key
```

- **Issues**:
  - `.env` committed to git? (Security risk!)
  - `JWT_SECRET` is weak and visible
  - Different `.env` for dev/staging/production?
  - Docker support?

---

### Q10.2: Deployment Strategy
**Question**: How do you deploy?

- Questions:
  - CI/CD pipeline? (GitHub Actions, GitLab CI?)
  - Automated testing before deploy?
  - Database migrations automated?
  - Rollback strategy if deploy fails?
  - Monitoring/alerts in production?

---

### Q10.3: Frontend Deployment
**Question**: Frontend is static files, but:

- **Questions**:
  - Where are static files served? (Nginx, S3, Vercel?)
  - Cache busting for updates?
  - CDN for assets?
  - HTTP/2, gzip compression enabled?
  - Handled CORS properly in production?

---

### Q10.4: Scalability Planning
**Question**: As user base grows:

- **Currently**: Single backend server, single database
- **When 10,000 users?** 100,000? 1,000,000?
- **Questions**:
  - Database sharding strategy?
  - Backend load balancing?
  - Cache layer (Redis)?
  - Message queues for async operations?
  - Image optimization and CDN?
  - Do you have a roadmap?

---

## 1️⃣1️⃣ CODE DOCUMENTATION

### Q11.1: Missing Documentation
**Question**: Code lacks comments/documentation:

```javascript
// Example from cart.js:
function displayCart() {  // What does it do?
  ...
}

async function updateItemQuantity(productId, newQuantity) {
  // What are valid inputs?
  // What exceptions might it throw?
  // What's the expected behavior?
}
```

**Should have**:
```javascript
/**
 * Display cart items on the page
 * @description Renders all cart items with quantity controls
 * @returns {void}
 * @throws {Error} If cart container not found
 * @example
 *   displayCart();  // Renders cart from cartManager.cart
 */
async function updateItemQuantity(productId, newQuantity) {
  ...
}
```

---

### Q11.2: Architecture Documentation
**Question**: How do new developers understand:

- **Project structure**: What's in each folder?
- **Data flow**: Frontend → Backend → Database
- **Authentication flow**: How does JWT work?
- **Adding new features**: Where do I add new API endpoint?
- **Configuration**: What env variables exist?
- **Deployment**: How to deploy to production?

---

### Q11.3: API Documentation
**Question**: API endpoints documented?

- No Swagger/OpenAPI specification
- No Postman collection
- No README for API consumers

**Should have**:
```markdown
## GET /api/products

**Description**: List all products with pagination

**Parameters**:
- page (optional, default: 1)
- limit (optional, default: 10)
- category (optional string)
- sortBy (optional: price, rating)

**Response**:
```
{
  "status": "success",
  "data": [...],
  "pagination": {...}
}
```

**Errors**:
- 400: Invalid query parameters
- 500: Server error
```

---

## 1️⃣2️⃣ MONITORING & LOGGING

### Q12.1: Logging Strategy
**Question**: How do you debug issues in production?

```javascript
// Current:
console.log('message')
console.error('error')

// No logging library visible:
// - No Winston, Bunyan, Pino
// - No log levels (debug, info, warn, error)
// - No log file persistence
// - No centralized logging (ELK, Datadog, etc.)
```

- **Questions**:
  - How do you find errors in production?
  - How long are logs kept?
  - Can you search logs by user/error/date?
  - Performance impact of logging?

---

### Q12.2: Error Tracking
**Question**: Bugs in production - how do you know?

- No error tracking service visible (Sentry, Rollbar, New Relic)
- Users see errors but you don't know?

---

## 1️⃣3️⃣ DATA PRIVACY & COMPLIANCE

### Q13.1: GDPR & Privacy
**Question**: User data handling:

- **Questions**:
  - How do you delete user data? (right to be forgotten)
  - What data is collected and why?
  - User consent for data collection?
  - Password storage secure? (salted hash)
  - PII (Personally Identifiable Information) exposure risks?

---

### Q13.2: Data Encryption
**Question**: Is sensitive data encrypted?

```javascript
// Are these encrypted at rest?
- User passwords (yes, hashed)
- Phone numbers (?)
- Addresses (?)
- Payment info (?)
- JWT secrets (?)
```

---

## 1️⃣4️⃣ BUSINESS LOGIC CONCERNS

### Q14.1: Order Pricing Logic
**Question**: Order total calculation:

```javascript
subtotal = sum(price * quantity)
tax = subtotal * 0.15
delivery = 150
total = subtotal + tax + delivery
```

- **Questions**:
  - What if product price changes after added to cart?
  - Order placed at price X, but product now costs Y?
  - Should you capture price at add-time? (Currently doing this - good!)
  - What about discounts/coupons?
  - What about regional delivery charges?

---

### Q14.2: Stock Management
**Question**: Inventory management:

```javascript
// Current: pessimistic locking (check then reserve)
if (stock >= qty) {
  // Add to cart
}
```

- **Race condition risk**:
  ```
  User1: Check stock=100, add 50
  User2: Check stock=100, add 50  (Happens simultaneously)
  Result: 100 items sold, only added by 2 users → Stock negative!
  ```

- **Questions**:
  - Is this a problem? (Unlikely but possible)
  - Solution: Database transactions? Optimistic locking? Queues?
  - Overselling policy: Allow or prevent?

---

### Q14.3: Payment Processing
**Question**: Payment integration:

```javascript
// Backend has:
payment_method: ['cash_on_delivery', 'credit_card', 'online_bank']
```

- **But no actual payment processing visible**
- **Questions**:
  - How do you handle credit card payments securely?
  - PCI-DSS compliance needed?
  - Which payment gateway? (Stripe, PayPal, etc.)
  - Webhook handling for payment confirmation?

---

## 1️⃣5️⃣ TEAM & PROCESS QUESTIONS

### Q15.1: Development Process
**Question**:

- Git workflow (gitflow, trunk-based)?
- Code review process?
- Commit message standards?
- Branch naming conventions?
- Merge strategy (squash, rebase, merge)?

---

### Q15.2: Knowledge Sharing
**Question**:

- How are architectural decisions documented?
- Where is technical debt tracked?
- How do you onboard new developers?
- Code review checklist?
- Design review process?

---

### Q15.3: Timeline & MVP
**Question**:

- What's MVP vs future scope?
- Is this production-ready?
- Known limitations?
- Technical debt acknowledged?

---

## 📊 SUMMARY: Top Priority Questions

| Priority | Category | Question |
|----------|----------|----------|
| 🔴 **CRITICAL** | Testing | Why no automated tests? How to ensure quality? |
| 🔴 **CRITICAL** | Security | Is CORS properly configured? Risks exposed? |
| 🔴 **CRITICAL** | Performance | Any N+1 query problems? Pagination working? |
| 🟠 **HIGH** | Architecture | Is framework-less frontend scalable long-term? |
| 🟠 **HIGH** | Code Quality | Should code be refactored into modules? |
| 🟠 **HIGH** | Documentation | How do new developers understand system? |
| 🟡 **MEDIUM** | Deployment | What's the CI/CD pipeline? Deployment process? |
| 🟡 **MEDIUM** | Monitoring | How do you debug production issues? |
| 🟢 **LOW** | Data Privacy | GDPR compliance? Data retention policy? |

---

## 🎯 NEXT STEPS FOR IMPROVEMENT

1. **Add Test Suite** - Unit tests for business logic, API tests
2. **Security Audit** - CORS, XSS, injection, auth flows
3. **Performance Profiling** - Database queries, API response sizes
4. **Documentation** - API docs, architecture docs, developer guide
5. **Logging/Monitoring** - Error tracking, performance monitoring
6. **Refactor Frontend** - Consider lightweight framework (Svelte, Alpine.js)
7. **Database Optimization** - Review indexes, query patterns
8. **CI/CD Setup** - Automated testing and deployment pipeline

---

**Document Version**: 1.0
**Date**: April 23, 2026
**Purpose**: Critical review for software quality and best practices
**Status**: Discussion starter for team
