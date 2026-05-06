# 🛒 Cart Operations - API Flow Diagram

## Complete Request/Response Flow

### 1. ADD TO CART FLOW
```
┌─────────────────────────────────────────────────────────────┐
│                   USER CLICKS "ADD TO CART"                 │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Frontend: cartManager.addToCart(product)                   │
│  ├─ Validate product data                                   │
│  ├─ Find existing item or create new                        │
│  ├─ Increment quantity if exists                            │
│  └─ Update localStorage                                     │
└────────────────────────┬────────────────────────────────────┘
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
    ┌─────────────────┐   ┌──────────────────┐
    │ LOCAL STORAGE   │   │ BACKEND SYNC     │
    │ UPDATED ✓       │   │ (Non-blocking)   │
    └────┬────────────┘   └────────┬─────────┘
         │                         │
         │                    POST /api/cart/add
         │                    ├─ Headers:
         │                    │  ├─ Authorization: Bearer {token}
         │                    │  └─ Content-Type: application/json
         │                    ├─ Body:
         │                    │  ├─ product_id: string
         │                    │  └─ quantity: number
         │                    │
         │                    ▼
         │            ┌──────────────────┐
         │            │ Backend Processing
         │            ├─ Verify user token
         │            ├─ Validate product
         │            ├─ Check stock
         │            ├─ Add/update in DB
         │            └─ Return cart
         │                    │
         │                    ▼
         │            ┌──────────────────┐
         │            │ Response 200 OK  │
         │            │ {                │
         │            │   status: success│
         │            │   cart: {...}    │
         │            │ }                │
         │            └──────────────────┘
         │
    Update CartCount ✓
    Show Notification ✓
```

---

### 2. REMOVE ITEM FLOW
```
┌─────────────────────────────────────────────────────────────┐
│              USER CLICKS "REMOVE" BUTTON                     │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  Show Confirmation Dialog                                    │
└────────────────────────┬────────────────────────────────────┘
         ┌───────────────┴───────────────┐
         │                               │
         ▼                               ▼
    User Confirms                  User Cancels
         │                               │
         ▼                               ▼
  ┌──────────────────┐           No Action
  │ removeItem()     │
  │                  │
  │ ├─ Find item     │
  │ ├─ Remove from   │
  │ │  localStorage  │
  │ ├─ Update UI     │
  │ └─ Notify user   │
  └────────┬─────────┘
           │
           ▼
  POST /api/cart/remove
  ├─ Headers:
  │  ├─ Authorization: Bearer {token}
  │  └─ Content-Type: application/json
  ├─ Body:
  │  └─ product_id: string
  │
  ▼
  Backend Processing
  ├─ Verify user token
  ├─ Find item in cart
  ├─ Remove from DB
  └─ Return updated cart
```

---

### 3. INCREASE QUANTITY FLOW
```
┌─────────────────────────────────────────────────────────────┐
│           USER CLICKS "+" (PLUS) BUTTON                      │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  increaseQuantity(productId)                                 │
│  └─ Get current quantity                                    │
│     Call updateItemQuantity(id, qty + 1)                    │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  updateItemQuantity(productId, newQuantity)                  │
│  ├─ Validate inputs                                         │
│  ├─ Check boundaries (1-999)                                │
│  ├─ Update local quantity                                   │
│  ├─ Recalculate totals                                      │
│  └─ Save to localStorage                                    │
└────────────────────────┬────────────────────────────────────┘
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
    ┌─────────────────┐   ┌──────────────────┐
    │ UI UPDATES ✓    │   │ BACKEND SYNC     │
    │ ├─ Qty display  │   │                  │
    │ ├─ Item total   │   │ PUT /api/cart/   │
    │ └─ Cart total   │   │ update           │
    └─────────────────┘   │                  │
                          │ Body:            │
                          │ {                │
                          │  product_id,     │
                          │  quantity        │
                          │ }                │
                          │                  │
                          └────────┬─────────┘
                                   │
                              Backend:
                              ├─ Verify token
                              ├─ Validate qty
                              ├─ Check stock
                              ├─ Update DB
                              └─ Return cart
```

---

### 4. DECREASE QUANTITY FLOW
```
┌─────────────────────────────────────────────────────────────┐
│           USER CLICKS "-" (MINUS) BUTTON                     │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────┐
│  decreaseQuantity(productId)                                │
│  ├─ Get current quantity                                   │
│  └─ Check if qty = 1                                       │
└────────────────────────┬────────────────────────────────────┘
         ┌───────────────┴───────────────┐
         │                               │
      qty > 1                         qty = 1
         │                               │
         ▼                               ▼
    Call updateItemQuantity      Show Confirmation:
    (id, qty - 1)                "Remove this item?"
         │                               │
         │                      ┌────────┴────────┐
         │                      │                 │
         │                      Yes               No
         │                      │                 │
         │                      ▼                 ▼
         │                 removeItem()      No action
         │                      │
         └──────────┬───────────┘
                    │
                    ▼
    Display updated cart
    Show success notification
```

---

## 📊 HTTP Status Codes Reference

### Success Responses (2xx)
```
200 OK
├─ Cart retrieved
├─ Item added
├─ Quantity updated
└─ Item removed
```

### Client Error Responses (4xx)
```
400 Bad Request
├─ Missing required fields (product_id, quantity)
├─ Invalid quantity value
├─ Insufficient stock
└─ Quantity exceeds limit (>999)

401 Unauthorized
├─ Missing token
├─ Invalid token
└─ Expired token

404 Not Found
├─ Product not found
├─ Cart not found
└─ Item not in cart
```

### Server Error Responses (5xx)
```
500 Internal Server Error
├─ Database errors
├─ Server processing errors
└─ Unexpected server issues
```

---

## 🔐 Security & Validation Points

### Frontend Validation:
1. ✅ Product ID validation
2. ✅ Quantity boundary checks (1-999)
3. ✅ NaN/undefined checks
4. ✅ Empty cart handling
5. ✅ Error message display

### Backend Validation:
1. ✅ Authentication token verification
2. ✅ Product existence check
3. ✅ Stock quantity validation
4. ✅ User authorization (cart belongs to user)
5. ✅ Input type validation
6. ✅ ObjectId string comparison

---

## 💾 Data Models

### Cart Item Structure (Frontend)
```javascript
{
  id: "507f1f77bcf86cd799439011",        // Product ID
  name: "Fresh Apples",                   // Product name
  price: 250,                             // Unit price
  quantity: 3,                            // Qty in cart
  icon: "🍎"                              // Category icon
}
```

### Cart Item Structure (Backend)
```javascript
{
  product_id: ObjectId,                   // Reference to product
  product_name: "Fresh Apples",           // Denormalized name
  price: 250,                             // Price snapshot
  quantity: 3,                            // Quantity
  added_at: Date                          // When added
}
```

---

## 🔄 Error Recovery Flow

### When Network Error Occurs:
```
┌─────────────────────┐
│  Backend Call Fails │
└────────────┬────────┘
             │
             ▼
┌─────────────────────────────────┐
│ Catch Error in try-catch         │
│ Log error to console             │
│ Continue with local update       │
└────────────┬────────────────────┘
             │
    ┌────────┴────────┐
    │                 │
    ▼                 ▼
 Local OK        Notify User
 Cart works      Try again
 Display new UI  (with fallback)
```

---

**Document Version**: 1.0
**Last Updated**: April 23, 2026
**Status**: Complete
