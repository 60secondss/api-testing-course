# 🍜 API Testing Lab: Auntie Som’s Noodle Stall

> **Objective:** Prove that Nephew Lek’s "it works on my machine" attitude is a recipe for disaster.

---

## 📖 The Backstory

Auntie Som runs a **legendary noodle stall**. Her Tom Yum is world-class, her customers are loyal, and her business is booming.  

Recently, her nephew **Lek** (who just finished a 2-week coding bootcamp) decided to "digitize" the business. He built a backend API to handle online orders. Auntie Som is thrilled, but the code… well, Lek says:  

> *"It’s fine lah Auntie! I tested it once with my own phone. No need for professional testing!"*

**You are the Last Line of Defense.**  
Before Auntie Som launches this app to 1,000+ hungry customers, you must audit the system, find the logic flaws, and automate the validation using **Bruno**.

---

## 🎯 Learning Objectives

By the end of this lab, you should be able to:
- 🖋️ Write **Bruno test scripts** for automated validation.
- 📦 Extract values from API responses using **Post-response scripts**.
- 🔐 Manage and use **Environment Variables**.
- 🧠 Verify **Business Rules**, not just HTTP status codes.
- 🚀 Catch regressions automatically before they hit production.

---

## 🛠️ Setup Instructions

### 1️⃣ Prepare the Project
- **Fork this repository** to your own GitHub account.
- **Clone it** locally to your machine.

### 2️⃣ Start the API Server
The backend is a lightweight Flask application.
- **Requirements:** Python 3.10+ installed.
- **Navigate** to the `auntie-som-noodle-api` folder.
- **Install dependencies:**  
  ```bash
  pip install flask
  ```
- **Run the server:**  
  ```bash
  python app.py
  ```
- The server will run at: `http://localhost:5000`

### 3️⃣ Setup Bruno
- Download and install [Bruno](https://www.usebruno.com).
- Create a **New Collection** named `Auntie Som Lab`.
- Create an **Environment** (e.g., `local`) and set a variable `baseUrl` to `http://localhost:5000`.

---

## 📑 API Reference

### 🔐 Authentication
`POST /auth/login` - Get an access token.
- **Body (JSON):**
  ```json
  {
    "username": "student",
    "password": "hungry"
  }
  ```
- **Response (200 OK):**
  ```json
  {
    "accessToken": "uuid-token-string",
    "expiresIn": 3600
  }
  ```

---

### 🍜 Menu
`GET /menu` - View available items and stock levels.
- **Response (200 OK):**
  ```json
  [
    { "id": 1, "name": "Tom Yum Noodles", "price": 50, "stock": 2 },
    { "id": 2, "name": "Fried Rice", "price": 45, "stock": 1 }
  ]
  ```

---

### 🛒 Orders
`POST /orders` - Place a new order.
- **Headers:** `Authorization: Bearer <token>`
- **Body (JSON):**
  ```json
  {
    "itemId": 1,
    "quantity": 1
  }
  ```
- **Response (200 OK):**
  ```json
  {
    "orderId": "ORD-abc123",
    "status": "created",
    "totalPrice": 45
  }
  ```

`GET /orders/<orderId>` - Retrieve order details.
- **Response (200 OK):**
  ```json
  {
    "orderId": "ORD-abc123",
    "status": "created",
    "totalPrice": 45
  }
  ```

---

## 🕵️ Your Mission: The Silent Auditor

Your task is to create a comprehensive Bruno collection that validates the entire workflow.  

⚠️ **The Catch:** Nephew Lek left several **logic bugs** in the system. Some endpoints might return `200 OK` even when the business logic is completely broken.

**Your collection must include:**
1.  **Auth Flow**: Automatically store the `accessToken` from login and use it for subsequent requests.
2.  **Order Validation**: Ensure orders can only be placed with valid auth, valid items, and available stock.
3.  **Data Integrity**: Check that the `totalPrice` calculated by the API matches your expectations.
4.  **Edge Cases**: What happens if you order more than what's in stock? What happens if you request a non-existent order?

> [!TIP]
> **Clicking "Send" is not testing.** Your tests must contain scripts (Assertions or Javascript) that **FAIL** when the system behaves incorrectly.

---

## 🧠 Rules of the Lab
- ❌ **Do NOT** modify the `app.py` backend code.
- ❌ **Do NOT** rely on manual checking.
- ✅ **Use environment variables** for the Base URL and Tokens.
- ✅ **Use Scripts/Assertions** for all validations.

---

## 📦 What to Submit

1.  **Push your Bruno collection folder** to your GitHub repository.
2.  **Update your README.md** (the one in your fork) with:
    - A summary of the bugs you discovered.
    - How your tests detect these bugs.
4.  **Send the GitHub link** to your instructor via Discord.

---
*Good luck. Auntie Som is counting on you!* 🍜🔥

---

## 🔎 Audit Findings

I ran the Bruno collection in [`auntie-som-noodle-api-test/`](./auntie-som-noodle-api-test/) against Lek's server and found **4 logic bugs**. All of them return `200 OK`, so they'd sail right past anyone who was just eyeballing status codes.

### Bug 1 — Overselling: stock vs. quantity is never compared
`POST /orders` only checks `if item["stock"] <= 0`. It never checks whether `quantity` is actually available. Order 11 bowls of Fried Rice when only 1 is in stock, and the API happily accepts it and drives `stock` negative.

- **Where:** `create_order()` in `app.py`, the stock check.
- **Caught by:** `3 - Orders/5 - Create Order - Exceeds Stock (BUG).bru` — reads the live stock count from `GET /menu`, requests `stock + 10`, and asserts the response should be `400`/`409`. It gets `200` instead, so the test fails and flags the bug.

### Bug 2 — Every order is silently short by 5 baht
`total_price = (item["price"] * quantity) - 5`. There's a flat `-5` baked into every single order, with no explanation anywhere in the API docs. A Tom Yum Noodles order (price 50, qty 1) returns `totalPrice: 45` instead of `50`.

- **Where:** `create_order()` in `app.py`, the `total_price` line.
- **Caught by:** `3 - Orders/1 - Create Order - Valid.bru` — asserts `totalPrice` equals `price * quantity` using the price pulled live from `/menu`. Fails with "expected 45 to equal 50".

### Bug 3 — Looking up a missing order returns 200, not 404
`GET /orders/<order_id>` returns `jsonify({"error": "Order not found"}), 200` when the order doesn't exist — an error payload wrapped in a success status code. Any client checking `response.ok` instead of the body would think the lookup succeeded.

- **Where:** `get_order()` in `app.py`.
- **Caught by:** `3 - Orders/6 - Get Order - Non-existent (BUG).bru` — requests a made-up `orderId` and asserts `404`. Gets `200` instead.

### Bug 4 — No validation on `quantity` at all
There's no check that `quantity` is a positive integer. Ordering `0` (or a negative number) is accepted: the stock calculation and `totalPrice` still run, producing nonsense like a `-5` total for an order of nothing.

- **Where:** `create_order()` in `app.py` — no guard before `item["stock"] -= quantity`.
- **Caught by:** `3 - Orders/7 - Create Order - Zero Quantity (BUG).bru` — orders `quantity: 0` and asserts `400`. Gets `200` instead.

### How the collection is organized
```
auntie-som-noodle-api-test/
├── environments/local.bru        # baseUrl -> http://localhost:5000
├── 1 - Auth Flow/                 # login, captures accessToken into env
├── 2 - Menu/                      # reads menu, captures live stock/price into env
└── 3 - Orders/                    # happy path + the 4 bugs above
```
The auth token and current stock/price are captured via `post-response` scripts into environment variables (`accessToken`, `tomYumPrice`, `friedRiceStock`, …) so the later requests never hardcode values that could drift — `Create Order - Exceeds Stock` always asks for "current stock + 10", for example.

### Result when run against Lek's server
Running the collection end-to-end: **17/21 tests pass, 4 fail** — and the 4 failures are exactly the 4 bugs above, each failing with the specific, readable assertion message quoted next to it. The happy-path, auth, and 404-for-bad-item tests all pass, confirming the collection isn't just failing everything by accident.
