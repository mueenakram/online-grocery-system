# 💡 Recommendations & Code Examples - Software Construction Best Practices

## Addressing the Key Questions

---

## 1️⃣ TESTING RECOMMENDATIONS

### Current State: ❌ No tests

### Recommended Test Structure

```
backend/
├── __tests__/
│   ├── unit/
│   │   ├── models/
│   │   │   └── Cart.test.js
│   │   ├── controllers/
│   │   │   ├── cartController.test.js
│   │   │   ├── authController.test.js
│   │   │   └── productController.test.js
│   │   └── utils/
│   │       └── validation.test.js
│   ├── integration/
│   │   ├── cart.integration.test.js
│   │   ├── auth.integration.test.js
│   │   └── order.integration.test.js
│   └── fixtures/
│       └── seedData.js
├── ...
└── package.json (with test scripts)

frontend/__tests__/
├── unit/
│   ├── cartManager.test.js
│   ├── apiService.test.js
│   └── utils.test.js
└── e2e/
    ├── cart.e2e.test.js
    └── checkout.e2e.test.js
```

### Sample Test - Backend (Jest)

```javascript
// backend/__tests__/unit/controllers/cartController.test.js
const request = require('supertest');
const app = require('../../server');
const Cart = require('../../models/CartMongo');
const User = require('../../models/UserMongo');
const Product = require('../../models/ProductMongo');

describe('CartController', () => {
  let token;
  let userId;
  let productId;
  
  beforeAll(async () => {
    // Setup: Create test user and product
    const user = await User.create({
      email: 'test@cart.com',
      password: 'test123',
      name: 'Test User'
    });
    userId = user._id;
    
    // Get JWT token
    const res = await request(app)
      .post('/api/auth/login')
      .send({ email: 'test@cart.com', password: 'test123' });
    token = res.body.token;
    
    // Create test product
    const product = await Product.create({
      name: 'Test Apple',
      price: 150,
      stock_quantity: 100
    });
    productId = product._id;
  });
  
  afterAll(async () => {
    // Cleanup
    await User.deleteMany({});
    await Product.deleteMany({});
    await Cart.deleteMany({});
  });

  // TEST 1: Add to cart
  describe('POST /api/cart/add', () => {
    test('should add product to empty cart', async () => {
      const res = await request(app)
        .post('/api/cart/add')
        .set('Authorization', `Bearer ${token}`)
        .send({
          product_id: productId,
          quantity: 2
        });
      
      expect(res.status).toBe(200);
      expect(res.body.status).toBe('success');
      expect(res.body.cart.items).toHaveLength(1);
      expect(res.body.cart.items[0].quantity).toBe(2);
    });

    test('should increment quantity if product already in cart', async () => {
      // Add first time
      await request(app)
        .post('/api/cart/add')
        .set('Authorization', `Bearer ${token}`)
        .send({ product_id: productId, quantity: 2 });
      
      // Add again
      const res = await request(app)
        .post('/api/cart/add')
        .set('Authorization', `Bearer ${token}`)
        .send({ product_id: productId, quantity: 3 });
      
      expect(res.body.cart.items[0].quantity).toBe(5);  // 2 + 3
    });

    test('should reject quantity > 999', async () => {
      const res = await request(app)
        .post('/api/cart/add')
        .set('Authorization', `Bearer ${token}`)
        .send({
          product_id: productId,
          quantity: 1000
        });
      
      expect(res.status).toBe(400);
      expect(res.body.message).toContain('Maximum quantity');
    });

    test('should reject if insufficient stock', async () => {
      const res = await request(app)
        .post('/api/cart/add')
        .set('Authorization', `Bearer ${token}`)
        .send({
          product_id: productId,
          quantity: 200  // More than 100 available
        });
      
      expect(res.status).toBe(400);
      expect(res.body.message).toContain('Insufficient stock');
    });

    test('should reject without token (401)', async () => {
      const res = await request(app)
        .post('/api/cart/add')
        .send({ product_id: productId, quantity: 1 });
      
      expect(res.status).toBe(401);
    });
  });

  // TEST 2: Update quantity
  describe('PUT /api/cart/update', () => {
    beforeEach(async () => {
      await Cart.deleteMany({});
      await request(app)
        .post('/api/cart/add')
        .set('Authorization', `Bearer ${token}`)
        .send({ product_id: productId, quantity: 5 });
    });

    test('should update quantity', async () => {
      const res = await request(app)
        .put('/api/cart/update')
        .set('Authorization', `Bearer ${token}`)
        .send({ product_id: productId, quantity: 3 });
      
      expect(res.status).toBe(200);
      expect(res.body.cart.items[0].quantity).toBe(3);
    });

    test('should remove item if quantity = 0', async () => {
      const res = await request(app)
        .put('/api/cart/update')
        .set('Authorization', `Bearer ${token}`)
        .send({ product_id: productId, quantity: 0 });
      
      expect(res.status).toBe(200);
      expect(res.body.cart.items).toHaveLength(0);
    });
  });

  // TEST 3: Remove from cart
  describe('POST /api/cart/remove', () => {
    beforeEach(async () => {
      await Cart.deleteMany({});
      await request(app)
        .post('/api/cart/add')
        .set('Authorization', `Bearer ${token}`)
        .send({ product_id: productId, quantity: 2 });
    });

    test('should remove item from cart', async () => {
      const res = await request(app)
        .post('/api/cart/remove')
        .set('Authorization', `Bearer ${token}`)
        .send({ product_id: productId });
      
      expect(res.status).toBe(200);
      expect(res.body.cart.items).toHaveLength(0);
    });

    test('should fail if item not in cart', async () => {
      const res = await request(app)
        .post('/api/cart/remove')
        .set('Authorization', `Bearer ${token}`)
        .send({ product_id: 'invalid-id' });
      
      expect(res.status).toBe(404);
    });
  });
});
```

### Sample Test - Frontend (Jest)

```javascript
// frontend/__tests__/unit/cartManager.test.js
const CartManager = require('../../assets/js/main.js').cartManager;

describe('CartManager', () => {
  beforeEach(() => {
    // Clear localStorage
    localStorage.clear();
  });

  describe('addToCart', () => {
    test('should add product to empty cart', () => {
      const product = {
        id: '1',
        name: 'Apple',
        price: 150,
        icon: '🍎'
      };
      
      cartManager.addToCart(product);
      
      expect(cartManager.cart).toHaveLength(1);
      expect(cartManager.cart[0].quantity).toBe(1);
    });

    test('should increment quantity if product exists', () => {
      const product = { id: '1', name: 'Apple', price: 150, icon: '🍎' };
      
      cartManager.addToCart(product);
      cartManager.addToCart(product);
      
      expect(cartManager.cart).toHaveLength(1);
      expect(cartManager.cart[0].quantity).toBe(2);
    });

    test('should persist to localStorage', () => {
      const product = { id: '1', name: 'Apple', price: 150, icon: '🍎' };
      cartManager.addToCart(product);
      
      const stored = JSON.parse(localStorage.getItem('cart'));
      expect(stored).toHaveLength(1);
      expect(stored[0].id).toBe('1');
    });
  });

  describe('updateQuantity', () => {
    beforeEach(() => {
      const product = { id: '1', name: 'Apple', price: 150, icon: '🍎' };
      cartManager.addToCart(product);
    });

    test('should update quantity', () => {
      cartManager.updateQuantity('1', 5);
      expect(cartManager.cart[0].quantity).toBe(5);
    });

    test('should limit quantity to 999', () => {
      cartManager.updateQuantity('1', 1000);
      expect(cartManager.cart[0].quantity).toBeLessThanOrEqual(999);
    });

    test('should enforce minimum quantity of 1', () => {
      cartManager.updateQuantity('1', 0);
      expect(cartManager.cart[0].quantity).toBeGreaterThanOrEqual(1);
    });
  });

  describe('removeFromCart', () => {
    beforeEach(() => {
      cartManager.addToCart({ id: '1', name: 'Apple', price: 150, icon: '🍎' });
      cartManager.addToCart({ id: '2', name: 'Banana', price: 80, icon: '🍌' });
    });

    test('should remove item from cart', () => {
      cartManager.removeFromCart('1');
      expect(cartManager.cart).toHaveLength(1);
      expect(cartManager.cart[0].id).toBe('2');
    });

    test('should handle removing non-existent item', () => {
      expect(() => cartManager.removeFromCart('999')).not.toThrow();
      expect(cartManager.cart).toHaveLength(2);
    });
  });

  describe('getCartTotal', () => {
    test('should calculate total correctly', () => {
      cartManager.addToCart({ id: '1', name: 'Apple', price: 150, icon: '🍎' });
      cartManager.addToCart({ id: '2', name: 'Banana', price: 80, icon: '🍌' });
      cartManager.updateQuantity('1', 2);  // 300
      cartManager.updateQuantity('2', 1);  // 80
      
      expect(cartManager.getCartTotal()).toBe(380);
    });
  });
});
```

### Package.json Configuration

```json
{
  "devDependencies": {
    "jest": "^29.0.0",
    "supertest": "^6.3.0",
    "@babel/preset-env": "^7.20.0",
    "mongodb-memory-server": "^8.12.0"
  },
  "scripts": {
    "test": "jest --coverage",
    "test:unit": "jest --testPathPattern=unit",
    "test:integration": "jest --testPathPattern=integration",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage --collectCoverageFrom='src/**/*.js'"
  },
  "jest": {
    "testEnvironment": "node",
    "collectCoverageFrom": [
      "controllers/**/*.js",
      "models/**/*.js",
      "!node_modules/**"
    ],
    "coverageThreshold": {
      "global": {
        "branches": 70,
        "functions": 70,
        "lines": 70,
        "statements": 70
      }
    }
  }
}
```

---

## 2️⃣ SECURITY RECOMMENDATIONS

### A. Fix CORS Configuration

**Current (❌ INSECURE)**:
```javascript
app.use(cors());  // Allows ANY origin!
```

**Recommended (✅ SECURE)**:
```javascript
const corsOptions = {
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:8000'],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  maxAge: 86400  // 24 hours
};

app.use(cors(corsOptions));
```

**Environment Variable**:
```env
ALLOWED_ORIGINS=http://localhost:8000,https://example.com
```

---

### B. Add Input Validation Middleware

**Bad (❌)**:
```javascript
const { product_id, quantity } = req.body;
// No validation, risky!
```

**Good (✅)**:
```javascript
const { body, validationResult } = require('express-validator');

const validateAddToCart = [
  body('product_id')
    .isMongoId()
    .withMessage('Invalid product ID'),
  body('quantity')
    .isInt({ min: 1, max: 999 })
    .withMessage('Quantity must be between 1 and 999'),
];

const handleValidationErrors = (req, res, next) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({ 
      status: 'error',
      errors: errors.array() 
    });
  }
  next();
};

router.post('/add', 
  validateAddToCart, 
  handleValidationErrors,
  authController.addToCart
);
```

---

### C. Prevent XSS in Frontend

**Bad (❌)**:
```javascript
// Template literals bypass safety!
cartItemsContainer.innerHTML = cartManager.cart.map(item => `
  <div class="product-name">${item.name}</div>
`).join('');
// If item.name = "<img onerror=alert(1)>", it executes!
```

**Good (✅)**:
```javascript
// Use textContent or DOM API
const container = document.createElement('div');
const nameElement = document.createElement('div');
nameElement.className = 'product-name';
nameElement.textContent = item.name;  // Safe!
container.appendChild(nameElement);
cartItemsContainer.appendChild(container);

// OR sanitize HTML
const DOMPurify = require('dompurify');
cartItemsContainer.innerHTML = DOMPurify.sanitize(
  cartManager.cart.map(item => `...`).join('')
);
```

---

### D. Add Security Headers

```javascript
const helmet = require('helmet');

app.use(helmet());  // Adds security headers

app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'", "'unsafe-inline'"],  // Ideally no unsafe-inline
    styleSrc: ["'self'", "'unsafe-inline'"],
    imgSrc: ["'self'", "https:"],
  },
}));
```

---

### E. Implement Rate Limiting

**Current (❌)**:
```javascript
const rateLimit = require('express-rate-limit');
// Imported but not used!
```

**Recommended (✅)**:
```javascript
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 100,  // 100 requests per window
  message: 'Too many requests, please try again later.',
  standardHeaders: true,  // Return rate limit info in headers
  legacyHeaders: false,  // Disable X-RateLimit-* headers
});

// Apply to sensitive endpoints
router.post('/auth/login', limiter, authController.login);
router.post('/auth/register', limiter, authController.register);

// Stricter limit for password reset
const strictLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,  // 1 hour
  max: 3,  // 3 attempts per hour
});
router.post('/auth/forgot-password', strictLimiter, authController.forgotPassword);
```

---

## 3️⃣ PERFORMANCE RECOMMENDATIONS

### A. Database Query Optimization

**Problem - N+1 Query (❌)**:
```javascript
// DON'T DO THIS:
const orders = await Order.find();
for (let order of orders) {
  const user = await User.findById(order.user_id);  // LOOP QUERY!
  order.user = user;
}
// Result: 1 query + N queries = Slow!
```

**Solution - Populate (✅)**:
```javascript
const orders = await Order.find()
  .populate('user_id')  // Single query with join
  .populate('items.product_id');  // Nested populate

// For complex queries:
const orders = await Order.find()
  .populate({
    path: 'user_id',
    select: 'name email phone'  // Only needed fields
  })
  .populate({
    path: 'items.product_id',
    select: 'name price'
  })
  .lean();  // Even faster - returns plain objects
```

---

### B. Add Database Indexes

**Current (❌)**: No visible indexes

**Recommended (✅)**:
```javascript
// models/UserMongo.js
userSchema.index({ email: 1 });  // Search by email
userSchema.index({ created_at: -1 });  // Sort by creation

// models/CartMongo.js
cartSchema.index({ user_id: 1 });  // Query cart by user

// models/OrderMongo.js
orderSchema.index({ user_id: 1, created_at: -1 });  // User's orders
orderSchema.index({ status: 1 });  // Filter by status

// models/ProductMongo.js
productSchema.index({ category: 1, price: 1 });  // Category + price filter
productSchema.index({ name: 'text' });  // Text search on name
```

---

### C. Implement Pagination

**Current (✅)**: Already has pagination, but verify it's used everywhere

**Example**:
```javascript
exports.getProducts = async (req, res) => {
  try {
    const page = Math.max(1, parseInt(req.query.page) || 1);
    const limit = Math.min(100, parseInt(req.query.limit) || 10);
    const skip = (page - 1) * limit;

    const products = await Product.find()
      .select('name price category rating')  // Only needed fields
      .sort({ created_at: -1 })
      .skip(skip)
      .limit(limit)
      .lean();

    const total = await Product.countDocuments();

    res.json({
      status: 'success',
      data: products,
      pagination: {
        page,
        limit,
        total,
        pages: Math.ceil(total / limit),
        hasNext: page < Math.ceil(total / limit)
      }
    });
  } catch (error) {
    res.status(500).json({ status: 'error', message: error.message });
  }
};
```

---

### D. Frontend Performance - Lazy Loading

```javascript
// Load images only when visible
document.addEventListener('DOMContentLoaded', () => {
  const imageObserver = new IntersectionObserver((entries, observer) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        const img = entry.target;
        img.src = img.dataset.src;  // Load image
        observer.unobserve(img);
      }
    });
  });

  document.querySelectorAll('img[data-src]').forEach(img => {
    imageObserver.observe(img);
  });
});

// HTML:
// <img data-src="path/to/image.jpg" alt="..." />
```

---

## 4️⃣ CODE QUALITY RECOMMENDATIONS

### A. Centralize Constants

**Current (❌)**: Scattered magic numbers/strings

**Recommended (✅)**:
```javascript
// config/constants.js
module.exports = {
  // Cart
  CART: {
    TAX_RATE: 0.15,
    DELIVERY_CHARGE: 150,
    MAX_QUANTITY: 999,
    MIN_QUANTITY: 1
  },
  
  // Authentication
  AUTH: {
    JWT_EXPIRY: '7d',
    PASSWORD_MIN_LENGTH: 6,
    SALT_ROUNDS: 10
  },
  
  // Pagination
  PAGINATION: {
    DEFAULT_PAGE: 1,
    DEFAULT_LIMIT: 10,
    MAX_LIMIT: 100
  },
  
  // Roles
  ROLES: {
    CUSTOMER: 'customer',
    ADMIN: 'admin',
    DELIVERY_RIDER: 'delivery_rider'
  },
  
  // Order status
  ORDER_STATUS: {
    PENDING: 'pending',
    CONFIRMED: 'confirmed',
    SHIPPED: 'shipped',
    DELIVERED: 'delivered',
    CANCELLED: 'cancelled'
  }
};

// Usage:
const { CART, ROLES } = require('../config/constants');
const tax = subtotal * CART.TAX_RATE;
if (user.role === ROLES.ADMIN) { ... }
```

---

### B. Create Shared Error Handler

**Current (❌)**: Error handling scattered in each controller

**Recommended (✅)**:
```javascript
// utils/AppError.js
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.timestamp = new Date().toISOString();
  }
}

module.exports = AppError;

// middleware/errorHandler.js
const errorHandler = (err, req, res, next) => {
  const statusCode = err.statusCode || 500;
  const isDevelopment = process.env.NODE_ENV === 'development';

  // Log error
  console.error({
    timestamp: new Date().toISOString(),
    method: req.method,
    url: req.url,
    error: err.message,
    stack: isDevelopment ? err.stack : undefined
  });

  res.status(statusCode).json({
    status: 'error',
    message: err.message,
    ...(isDevelopment && { stack: err.stack })
  });
};

app.use(errorHandler);

// Usage in controllers:
exports.getCart = async (req, res, next) => {
  try {
    const cart = await Cart.findOne({ user_id: req.user.id });
    if (!cart) {
      throw new AppError('Cart not found', 404);
    }
    res.json({ status: 'success', cart });
  } catch (error) {
    next(error);  // Pass to error handler
  }
};
```

---

### C. Refactor Frontend into Modules

**Current (❌)**: 400+ lines in main.js

**Recommended (✅)**:
```
frontend/assets/js/
├── modules/
│   ├── auth.js              // Login/logout
│   ├── cart.js              // Cart manager
│   ├── products.js          // Product management
│   ├── orders.js            // Order management
│   └── utils/
│       ├── api.js           // API calls
│       ├── validation.js    // Input validation
│       └── format.js        // Formatting helpers
├── pages/
│   ├── home.js
│   ├── cart.js
│   ├── products.js
│   └── checkout.js
└── main.js                  // Entry point

// auth.js
const AuthModule = (() => {
  const login = async (email, password) => { ... };
  const logout = () => { ... };
  const getCurrentUser = () => { ... };
  
  return { login, logout, getCurrentUser };
})();

// Usage in pages/home.js
const home = (() => {
  return {
    init: () => {
      if (!AuthModule.getCurrentUser()) {
        window.location.href = 'login.html';
      }
    }
  };
})();
```

---

## 5️⃣ DOCUMENTATION RECOMMENDATIONS

### A. JSDoc Comments for Frontend

```javascript
/**
 * Display all cart items with quantity controls
 * @async
 * @function displayCart
 * @description Renders cart items from cartManager.cart to the DOM.
 * Updates summary totals and shows empty state if no items.
 * 
 * @returns {Promise<void>}
 * @throws {Error} If cartItems container not found in DOM
 * 
 * @example
 * // Called when page loads or cart changes
 * displayCart();
 */
async function displayCart() { ... }

/**
 * Increase product quantity in cart
 * @async
 * @function increaseQuantity
 * @param {string} productId - The product ID to increase
 * @param {Event} event - Click event object
 * @returns {Promise<void>}
 * 
 * @throws {Error} If product not found in cart
 * 
 * @example
 * // Called from "+#" button onclick
 * await increaseQuantity('507f1f77bcf86cd799439011', event);
 */
async function increaseQuantity(productId, event) { ... }
```

---

### B. API Documentation (OpenAPI/Swagger)

```yaml
# openapi.yaml
openapi: 3.0.0
info:
  title: Fresh Grocery API
  version: 1.0.0
  description: Online grocery delivery API

servers:
  - url: http://localhost:5000/api
    description: Development server
  - url: https://api.freshgrocery.com/api
    description: Production server

paths:
  /cart:
    get:
      summary: Get user's shopping cart
      tags:
        - Cart
      security:
        - bearerAuth: []
      responses:
        '200':
          description: Cart retrieved successfully
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    example: success
                  cart:
                    type: object
                    properties:
                      items:
                        type: array
                      subtotal:
                        type: number
                      tax:
                        type: number
                      total:
                        type: number
        '401':
          description: Unauthorized (no token)

  /cart/add:
    post:
      summary: Add product to cart
      tags:
        - Cart
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
                - product_id
                - quantity
              properties:
                product_id:
                  type: string
                  example: 507f1f77bcf86cd799439011
                quantity:
                  type: integer
                  minimum: 1
                  maximum: 999
                  example: 2
      responses:
        '200':
          description: Product added successfully
        '400':
          description: Invalid input or insufficient stock
        '401':
          description: Unauthorized
        '404':
          description: Product not found
```

---

### C. README for Developers

```markdown
# Fresh Grocery - Developer Guide

## Project Structure

```
fresh-grocery/
├── backend/                    # Node.js + Express API
│   ├── controllers/           # Business logic
│   ├── models/                # Mongoose schemas
│   ├── routes/                # API endpoints
│   ├── middleware/            # Auth, validation, etc.
│   ├── config/                # Configuration & constants
│   ├── __tests__/             # Test files
│   └── server.js              # Entry point
│
├── frontend/                   # HTML + JavaScript
│   ├── assets/
│   │   ├── css/               # Stylesheets
│   │   └── js/                # JavaScript modules
│   ├── pages/                 # HTML pages
│   └── __tests__/             # Test files
│
└── database/                  # Database schemas
    ├── schema.sql             # MySQL version
    └── mongodb-schema.js      # MongoDB version
```

## Getting Started

### Backend Setup
```bash
cd backend
npm install
cp .env.example .env          # Configure database
npm run seed                  # Seed test data
npm run dev                   # Start development server
```

### Frontend Setup
```bash
cd frontend
# Option 1: Use Python HTTP server
python -m http.server 8000

# Option 2: Use Node http-server
npx http-server

# Open http://localhost:8000 in browser
```

### Test Accounts
- Admin: `admin@example.com` / `admin123`
- Customer: `customer@example.com` / `password123`
- Rider: `rider1@example.com` / `rider123`

## Running Tests
```bash
npm test                      # Run all tests
npm run test:unit             # Unit tests only
npm run test:integration      # Integration tests
npm run test:coverage         # With coverage report
```

## API Endpoints

### Authentication
- `POST /api/auth/register` - Register new account
- `POST /api/auth/login` - Login with email/password
- `GET /api/auth/me` - Get current user profile

### Products
- `GET /api/products` - List all products (paginated)
- `GET /api/products/:id` - Get product details
- `GET /api/products/search/:query` - Search products

### Cart
- `GET /api/cart` - Get user's cart
- `POST /api/cart/add` - Add product to cart
- `PUT /api/cart/update` - Update quantity
- `POST /api/cart/remove` - Remove from cart

### Orders
- `POST /api/orders` - Create order from cart
- `GET /api/orders/my-orders` - Get user's orders
- `GET /api/orders/:id` - Get order details

## Key Concepts

### Authentication Flow
1. User registers → Password hashed with bcryptjs
2. User logs in → JWT token generated (7-day expiry)
3. Token stored in localStorage
4. Sent with each request: `Authorization: Bearer {token}`
5. Backend verifies JWT signature and expiry

### Cart System
- Stored in LOCAL STORAGE (frontend)
- Synced with BACKEND (one-way, backend is source of truth)
- On page load: Load from localStorage, then fetch from backend
- On operations: Update locally, sync asynchronously

### Role-Based Access
- `customer` - Browse products, create orders
- `admin` - Manage products, users, orders
- `delivery_rider` - Track and update deliveries

## Error Handling

### Standardized Response Format
```json
// Success
{ "status": "success", "data": {...}, "message": "..." }

// Error
{ "status": "error", "message": "..." }
```

### HTTP Status Codes
- `200` - Success
- `201` - Created
- `400` - Bad request (validation error)
- `401` - Unauthorized (missing/invalid token)
- `403` - Forbidden (insufficient permission)
- `404` - Not found
- `500` - Server error

## Development Workflow

1. Create a feature branch: `git checkout -b feature/cart-fix`
2. Make changes following code style
3. Write/update tests
4. Verify tests pass: `npm test`
5. Commit with CLEAR message: `git commit -m "Fix: Prevent cart quantity > 999"`
6. Push and create Pull Request
7. Code review before merge

## Deployment

### Backend (Production)
```bash
NODE_ENV=production npm start
# Ensure .env configured for production values
# Database backups automated
# Monitoring/logging enabled
```

### Frontend (Production)
```bash
# Build (if using build tools)
npm run build

# Deploy static files to CDN or web server
# Configure CORS for production API domain
# Enable gzip compression
# Set cache headers appropriately
```
```

---

## 📊 Quick Fixes Priority

| Issue | Severity | Effort | Impact |
|-------|----------|--------|--------|
| No tests | 🔴 HIGH | 🔴 BIG | 🟢 HIGH |
| CORS open | 🔴 HIGH | 🟢 SMALL | 🟢 HIGH |
| XSS risk | 🟠 MEDIUM | 🟢 SMALL | 🟠 MEDIUM |
| No logging | 🟠 MEDIUM | 🟡 MEDIUM | 🟡 MEDIUM |
| N+1 queries | 🟠 MEDIUM | 🟢 SMALL | 🟡 MEDIUM |
| No documentation | 🟡 LOW | 🔴 BIG | 🟡 MEDIUM |

---

**Document Version**: 1.0
**Date**: April 23, 2026
**Purpose**: Actionable recommendations with code examples
