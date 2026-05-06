# Shopping Cart Features - Complete Guide

## 📋 Overview
This document describes all the cart functionality implemented for customer users including add, remove, increase/decrease quantity, and checkout.

## ✅ Features Implemented

### 1. **Add to Cart**
- Users can add products to cart with a single click
- If product already exists in cart, quantity is incremented
- Backend syncs the addition with database
- Visual feedback with success notification

**Frontend Files:**
- `frontend/assets/js/main.js` - CartManager.addToCart()
- `frontend/pages/products.html` - Add to cart button

**Backend Endpoints:**
- `POST /api/cart/add` - Requires authentication token

### 2. **Remove Item from Cart**
- Users can remove items with a confirmation dialog
- Item is removed both from local storage and backend
- Updated total is calculated immediately
- Visual feedback with success notification

**Frontend Files:**
- `frontend/assets/js/cart.js` - removeItem()
- `frontend/assets/js/main.js` - CartManager.removeFromCart()

**Backend Endpoints:**
- `POST /api/cart/remove` - Requires authentication token

### 3. **Increase Quantity**
- Plus button (+) increases quantity by 1
- Maximum limit is 999 units
- Stock validation prevents exceeding available stock
- Real-time calculation of total price

**Frontend Files:**
- `frontend/assets/js/cart.js` - increaseQuantity()
- `frontend/assets/js/cart.html` - UI with plus button

**Backend Endpoints:**
- `PUT /api/cart/update` - Requires authentication token

### 4. **Decrease Quantity**
- Minus button (-) decreases quantity by 1
- Minimum quantity is 1
- If quantity reaches 0, item is removed with confirmation
- Real-time calculation of total price

**Frontend Files:**
- `frontend/assets/js/cart.js` - decreaseQuantity()
- `frontend/assets/js/cart.html` - UI with minus button

**Backend Endpoints:**
- `PUT /api/cart/update` - Requires authentication token

### 5. **Cart Summary**
- Subtotal: Sum of all item prices × quantities
- Delivery Charge: Fixed Rs. 150
- Tax: 15% of subtotal
- Total: Subtotal + Delivery + Tax

**Frontend Files:**
- `frontend/assets/js/cart.js` - updateSummary()
- `frontend/pages/cart.html` - Display section

## 🔧 Error Handling

### Frontend Error Handling
1. **Invalid Product ID** - Shows error notification
2. **Quantity Validation** - Ensures quantity is between 1-999
3. **Network Errors** - Falls back to local storage
4. **Missing Data** - Validates all required fields before operations

### Backend Error Handling
1. **Unauthorized** - Returns 401 if token missing
2. **Invalid Input** - Returns 400 for missing/invalid data
3. **Product Not Found** - Returns 404 if product doesn't exist
4. **Insufficient Stock** - Returns 400 with available stock info
5. **Cart Not Found** - Returns 404 for missing cart
6. **Server Errors** - Returns 500 with error message

## 🧪 Testing Guide

### Test Case 1: Add Product to Cart
```
1. Navigate to Products page
2. Click "Add to Cart" button on any product
3. Success notification appears
4. Cart count in navbar updates
5. Refresh page and verify item persists
```

### Test Case 2: Remove Item from Cart
```
1. Go to Cart page
2. Click "Remove" button on any item
3. Confirmation dialog appears
4. Click OK
5. Item is removed
6. Total is recalculated
```

### Test Case 3: Increase Quantity
```
1. Go to Cart page
2. Click "+" button on any item
3. Quantity increases by 1
4. Item total updates immediately
5. Cart total recalculates
6. Verify backend sync with network inspection
```

### Test Case 4: Decrease Quantity
```
1. Go to Cart page
2. Click "-" button on any item
3. Quantity decreases by 1
4. Item total updates immediately
5. If quantity was 1, confirmation appears
6. Confirm removal and verify item is deleted
```

### Test Case 5: Cart Summary Calculation
```
1. Add multiple items to cart
2. Verify Subtotal = Sum of (price × quantity)
3. Verify Tax = Subtotal × 0.15
4. Verify Total = Subtotal + 150 + Tax
5. Change quantities and re-verify calculations
```

### Test Case 6: Edge Cases
```
1. Try to add quantity > 999 (should show error)
2. Try to add quantity beyond available stock (should show error)
3. Go to cart with no items (should show empty state)
4. Logout and login to verify cart persists
5. Clear cart and verify all items removed
6. Test on mobile devices for responsive design
```

## 📦 API Endpoints

### Get Cart
```
GET /api/cart
Headers: Authorization: Bearer {token}
Response: { status: 'success', cart: { items: [...], subtotal, tax, total, itemCount } }
```

### Add to Cart
```
POST /api/cart/add
Headers: Authorization: Bearer {token}
Body: { product_id: string, quantity: number }
Response: { status: 'success', message: 'Item added to cart', cart: {...} }
```

### Update Cart Item
```
PUT /api/cart/update
Headers: Authorization: Bearer {token}
Body: { product_id: string, quantity: number }
Response: { status: 'success', message: 'Cart updated successfully', cart: {...} }
```

### Remove from Cart
```
POST /api/cart/remove
Headers: Authorization: Bearer {token}
Body: { product_id: string }
Response: { status: 'success', message: 'Item removed from cart', cart: {...} }
```

### Clear Cart
```
DELETE /api/cart/clear
Headers: Authorization: Bearer {token}
Response: { status: 'success', message: 'Cart cleared successfully', cart: {...} }
```

### Get Cart Summary
```
GET /api/cart/summary
Headers: Authorization: Bearer {token}
Response: { status: 'success', summary: { ... } }
```

## 🎨 UI Components

### Cart Item Display
- Product image/icon
- Product name
- Product price
- Quantity controls (-, quantity display, +)
- Item total (price × quantity)
- Remove button

### Cart Summary Component
- Sticky position on desktop
- Shows subtotal, delivery, tax, total
- "Proceed to Checkout" button
- "Continue Shopping" button

## 🛡️ Validation Rules

### Quantity Validation
- Minimum: 1 unit
- Maximum: 999 units
- Must be positive integer
- Cannot exceed available stock

### Stock Validation
- Backend checks available stock before operations
- Returns error with available quantity on insufficient stock
- Prevents overbooking

### Authentication
- All cart operations require valid JWT token
- Returns 401 Unauthorized if token missing/invalid
- Cart is user-specific

## 📲 Local Storage

### Cart Storage Key: `cart`
Structure:
```javascript
[
  {
    id: string,
    name: string,
    price: number,
    quantity: number,
    icon: string
  },
  ...
]
```

## 🔄 Sync Behavior

1. **Add to Cart**: Updates local storage immediately, syncs with backend asynchronously
2. **Remove Item**: Updates local storage immediately, syncs with backend asynchronously
3. **Update Quantity**: Updates local storage immediately, syncs with backend asynchronously
4. **Page Load**: Loads from local storage, then syncs with backend if authenticated
5. **Backend-First**: If backend returns error, user can continue with local copy

## ⚠️ Known Limitations and Notes

1. Cart is stored in local storage and backend database
2. Backend is source of truth for authenticated users
3. Unregistered users can populate local cart but can't proceed to checkout
4. Cart persists for 7 days in localStorage (browser default)
5. Different browser = different cart (unless user logs in)

## 🚀 Future Enhancements (Optional)

1. Save for later functionality
2. Wishlist integration
3. Bulk quantity updates
4. Auto-save cart recovery
5. Cart sharing between devices
6. Coupon code application
7. Cart timeline/history view

---

**Last Updated:** April 23, 2026
**Version:** 1.0
**Status:** Ready for Testing
