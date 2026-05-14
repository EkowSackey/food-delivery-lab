# Food Delivery API — Test Plan

> Base URL: `http://localhost:8080`  
> Tool: `httpie` (`http` command)  
> All requests go through the API Gateway.  
> Run tests **in order** — later sections depend on tokens/IDs from earlier ones.

---

## Status Legend
- [ ] Not yet run
- [x] Passed
- [!] Failed — needs investigation

---

## Variables to capture as you go

```
CUSTOMER_TOKEN=   # JWT from customer login
OWNER_TOKEN=      # JWT from restaurant_owner login
CUSTOMER_ID=      # from register/login response
RESTAURANT_ID=    # from POST /api/restaurants
MENU_ITEM_ID=     # from POST /api/restaurants/{id}/menu
ORDER_ID=         # from POST /api/orders
DELIVERY_ID=      # from GET /api/orders/{id} after delivery is created
```

---

## 1. Auth — Public endpoints (no token)

### 1.1 Register a customer
```bash
http POST localhost:8080/api/auth/register \
  username=johndoe \
  email=john@example.com \
  password=secret123 \
  firstName=John \
  lastName=Doe \
  phone=+1234567890 \
  deliveryAddress="123 Main St" \
  city=Accra
```
**Expected:** `200 OK` — body contains `token`, `customerId`, `username`, `role=CUSTOMER`  
**Capture:** `CUSTOMER_TOKEN`, `CUSTOMER_ID`  
- [ ] Status:

### 1.2 Register a restaurant owner
```bash
http POST localhost:8080/api/auth/register \
  username=ownerjan \
  email=owner@example.com \
  password=secret123 \
  firstName=Jan \
  lastName=Owner
```
**Expected:** `200 OK` — `role=CUSTOMER` (role upgrade done separately or via DB)  
**Note:** Role is `CUSTOMER` by default. To get `RESTAURANT_OWNER`, update role directly in the DB or check if there's an admin endpoint.  
- [ ] Status:

### 1.3 Login as customer
```bash
http POST localhost:8080/api/auth/login \
  username=johndoe \
  password=secret123
```
**Expected:** `200 OK` — `token`, `customerId`, `username`, `role`  
**Capture:** `CUSTOMER_TOKEN`  
- [ ] Status:

### 1.4 Login with wrong password
```bash
http POST localhost:8080/api/auth/login \
  username=johndoe \
  password=wrongpassword
```
**Expected:** `401 Unauthorized`  
- [ ] Status:

### 1.5 Register duplicate username
```bash
http POST localhost:8080/api/auth/register \
  username=johndoe \
  email=other@example.com \
  password=secret123
```
**Expected:** `4xx` error (conflict/bad request)  
- [ ] Status:

---

## 2. Customer — Authenticated endpoints

### 2.1 Get own profile
```bash
http GET localhost:8080/api/customers/me \
  "Authorization:Bearer $CUSTOMER_TOKEN"
```
**Expected:** `200 OK` — customer object with all fields  
- [ ] Status:

### 2.2 Update own profile
```bash
http PUT localhost:8080/api/customers/me \
  "Authorization:Bearer $CUSTOMER_TOKEN" \
  firstName=Johnny \
  city=Kumasi
```
**Expected:** `200 OK` — updated customer object  
- [ ] Status:

### 2.3 Access profile without token
```bash
http GET localhost:8080/api/customers/me
```
**Expected:** `401 Unauthorized`  
- [ ] Status:

### 2.4 Get customer by ID (SERVICE role only — should be rejected from gateway)
```bash
http GET localhost:8080/api/customers/1 \
  "Authorization:Bearer $CUSTOMER_TOKEN"
```
**Expected:** `403 Forbidden` (CUSTOMER role cannot access this endpoint)  
- [ ] Status:

---

## 3. Promote ownerjan to RESTAURANT_OWNER

The default role on registration is CUSTOMER. To test restaurant endpoints, update the role directly:

```bash
docker exec -it postgres psql -U postgres -d customer_db \
  -c "UPDATE customers SET role='RESTAURANT_OWNER' WHERE username='ownerjan';"
```

Then re-login to get a new token with the updated role:

```bash
http POST localhost:8080/api/auth/login \
  username=ownerjan \
  password=secret123
```
**Capture:** `OWNER_TOKEN`  
- [ ] Status:

---

## 4. Restaurants — Public endpoints

### 4.1 List all restaurants (empty at first)
```bash
http GET localhost:8080/api/restaurants/search/all
```
**Expected:** `200 OK` — empty list `[]`  
- [ ] Status:

### 4.2 Search by city (no results)
```bash
http GET localhost:8080/api/restaurants/search/city/Accra
```
**Expected:** `200 OK` — empty list  
- [ ] Status:

### 4.3 Search by cuisine (no results)
```bash
http GET localhost:8080/api/restaurants/search/cuisine/Italian
```
**Expected:** `200 OK` — empty list  
- [ ] Status:

---

## 5. Restaurants — Owner-authenticated endpoints

### 5.1 Create a restaurant (requires RESTAURANT_OWNER)
```bash
http POST localhost:8080/api/restaurants \
  "Authorization:Bearer $OWNER_TOKEN" \
  name="Pizza Palace" \
  description="Best pizza in town" \
  cuisineType=Italian \
  address="45 Ring Road" \
  city=Accra \
  phone=+233200000001 \
  estimatedDeliveryMinutes:=30
```
**Expected:** `200 OK` — restaurant object with `id`, `ownerId`, `active=true`  
**Capture:** `RESTAURANT_ID`  
- [ ] Status:

### 5.2 Create restaurant with CUSTOMER token (should fail)
```bash
http POST localhost:8080/api/restaurants \
  "Authorization:Bearer $CUSTOMER_TOKEN" \
  name="Fake Restaurant" \
  cuisineType=Fast\ Food \
  address="1 Some St" \
  city=Accra
```
**Expected:** `403 Forbidden`  
- [ ] Status:

### 5.3 Add a menu item
```bash
http POST localhost:8080/api/restaurants/$RESTAURANT_ID/menu \
  "Authorization:Bearer $OWNER_TOKEN" \
  name="Margherita Pizza" \
  description="Classic tomato and mozzarella" \
  price:=12.99 \
  category=Pizza
```
**Expected:** `200 OK` — menu item with `id`, `available=true`, `restaurantId`  
**Capture:** `MENU_ITEM_ID`  
- [ ] Status:

### 5.4 Add a second menu item
```bash
http POST localhost:8080/api/restaurants/$RESTAURANT_ID/menu \
  "Authorization:Bearer $OWNER_TOKEN" \
  name="Pepperoni Pizza" \
  description="Loaded with pepperoni" \
  price:=15.99 \
  category=Pizza
```
**Expected:** `200 OK`  
**Capture:** `MENU_ITEM_ID_2`  
- [ ] Status:

### 5.5 Get restaurant menu (public)
```bash
http GET localhost:8080/api/restaurants/$RESTAURANT_ID/menu
```
**Expected:** `200 OK` — list with 2 items  
- [ ] Status:

### 5.6 Get single restaurant (public)
```bash
http GET localhost:8080/api/restaurants/$RESTAURANT_ID
```
**Expected:** `200 OK` — restaurant object with `menuItemCount=2`  
- [ ] Status:

### 5.7 Get single menu item (public)
```bash
http GET localhost:8080/api/menu-items/$MENU_ITEM_ID
```
**Expected:** `200 OK` — menu item object  
- [ ] Status:

### 5.8 Update a menu item
```bash
http PUT localhost:8080/api/restaurants/menu/$MENU_ITEM_ID \
  "Authorization:Bearer $OWNER_TOKEN" \
  name="Margherita Pizza" \
  description="Classic tomato and fresh mozzarella" \
  price:=13.99 \
  category=Pizza
```
**Expected:** `200 OK` — updated menu item  
- [ ] Status:

### 5.9 Toggle menu item availability
```bash
http PATCH localhost:8080/api/restaurants/menu/$MENU_ITEM_ID/toggle \
  "Authorization:Bearer $OWNER_TOKEN"
```
**Expected:** `204 No Content` (item now unavailable)  
- [ ] Status:

### 5.10 Search by city (now has results)
```bash
http GET localhost:8080/api/restaurants/search/city/Accra
```
**Expected:** `200 OK` — list with "Pizza Palace"  
- [ ] Status:

### 5.11 Search by cuisine type
```bash
http GET localhost:8080/api/restaurants/search/cuisine/Italian
```
**Expected:** `200 OK` — list with "Pizza Palace"  
- [ ] Status:

---

## 6. Orders — Customer endpoints

> **Note:** Toggle the unavailable menu item back on before placing orders.
> ```bash
> http PATCH localhost:8080/api/restaurants/menu/$MENU_ITEM_ID/toggle \
>   "Authorization:Bearer $OWNER_TOKEN"
> ```

### 6.1 Place an order
```bash
http POST localhost:8080/api/orders \
  "Authorization:Bearer $CUSTOMER_TOKEN" \
  restaurantId:=$RESTAURANT_ID \
  deliveryAddress="123 Main St, Accra" \
  specialInstructions="Extra cheese please" \
  items:='[{"menuItemId": '$MENU_ITEM_ID', "quantity": 2}, {"menuItemId": '$MENU_ITEM_ID_2', "quantity": 1}]'
```
**Expected:** `200 OK` — order with `status=PLACED`, `totalAmount`, `deliveryFee=2.99`, items list  
**Capture:** `ORDER_ID`  
- [ ] Status:

### 6.2 Get order by ID (as customer)
```bash
http GET localhost:8080/api/orders/$ORDER_ID \
  "Authorization:Bearer $CUSTOMER_TOKEN"
```
**Expected:** `200 OK` — full order object  
- [ ] Status:

### 6.3 List my orders
```bash
http GET localhost:8080/api/orders/my-orders \
  "Authorization:Bearer $CUSTOMER_TOKEN"
```
**Expected:** `200 OK` — list with the order just placed  
- [ ] Status:

### 6.4 Place order with unavailable item
```bash
# First toggle off item 1
http PATCH localhost:8080/api/restaurants/menu/$MENU_ITEM_ID/toggle \
  "Authorization:Bearer $OWNER_TOKEN"

# Then try to order it
http POST localhost:8080/api/orders \
  "Authorization:Bearer $CUSTOMER_TOKEN" \
  restaurantId:=$RESTAURANT_ID \
  items:='[{"menuItemId": '$MENU_ITEM_ID', "quantity": 1}]'
```
**Expected:** `4xx` error — item not available  
**Cleanup:** Toggle item back on after this test  
- [ ] Status:

### 6.5 Place order without token
```bash
http POST localhost:8080/api/orders \
  restaurantId:=1 \
  items:='[{"menuItemId": 1, "quantity": 1}]'
```
**Expected:** `401 Unauthorized`  
- [ ] Status:

### 6.6 Cancel an order
```bash
# Place a fresh order first (reuse 6.1 command), capture new ORDER_ID_2
http POST localhost:8080/api/orders/$ORDER_ID_2/cancel \
  "Authorization:Bearer $CUSTOMER_TOKEN"
```
**Expected:** `200 OK` — order with `status=CANCELLED`  
- [ ] Status:

---

## 7. Orders — Restaurant owner endpoints

### 7.1 Get orders for restaurant
```bash
http GET localhost:8080/api/orders/restaurant/$RESTAURANT_ID \
  "Authorization:Bearer $OWNER_TOKEN"
```
**Expected:** `200 OK` — list of orders for this restaurant  
- [ ] Status:

### 7.2 Update order status (PLACED → CONFIRMED)
```bash
http PATCH "localhost:8080/api/orders/$ORDER_ID/status?status=CONFIRMED" \
  "Authorization:Bearer $OWNER_TOKEN"
```
**Expected:** `200 OK` — order with `status=CONFIRMED`  
- [ ] Status:

### 7.3 Update order status (CONFIRMED → PREPARING)
```bash
http PATCH "localhost:8080/api/orders/$ORDER_ID/status?status=PREPARING" \
  "Authorization:Bearer $OWNER_TOKEN"
```
**Expected:** `200 OK` — order with `status=PREPARING`  
- [ ] Status:

### 7.4 Update order status (PREPARING → READY_FOR_PICKUP)
```bash
http PATCH "localhost:8080/api/orders/$ORDER_ID/status?status=READY_FOR_PICKUP" \
  "Authorization:Bearer $OWNER_TOKEN"
```
**Expected:** `200 OK` — order with `status=READY_FOR_PICKUP`  
**Note:** At this point the order-service should publish an `OrderPlacedEvent` to RabbitMQ (if not done at PLACED), and the delivery-service should have created a delivery record.  
- [ ] Status:

### 7.5 Customer cannot update order status
```bash
http PATCH "localhost:8080/api/orders/$ORDER_ID/status?status=CANCELLED" \
  "Authorization:Bearer $CUSTOMER_TOKEN"
```
**Expected:** `403 Forbidden`  
- [ ] Status:

---

## 8. Async flow — Verify delivery was created via RabbitMQ

> The `OrderPlacedEvent` is published when an order is placed. The delivery service consumes it and creates a delivery record. Delivery endpoints are SERVICE-role only (internal), so we check via the order's `deliveryId` field.

### 8.1 Poll order until deliveryId is populated
```bash
http GET localhost:8080/api/orders/$ORDER_ID \
  "Authorization:Bearer $CUSTOMER_TOKEN"
```
**Expected:** `deliveryId` field is not null (may take a few seconds after order placed)  
**Capture:** `DELIVERY_ID`  
- [ ] Status:

### 8.2 Verify RabbitMQ message flow via order status updates
After the delivery service picks up the event and updates its status, it publishes a `DeliveryStatusEvent` back to the order service. Progress the delivery status and observe:

```bash
# Check order status is updated (via delivery status events coming back)
http GET localhost:8080/api/orders/$ORDER_ID \
  "Authorization:Bearer $CUSTOMER_TOKEN"
```
**Expected:** Status progresses as delivery moves through `PENDING → ASSIGNED → PICKED_UP → IN_TRANSIT → DELIVERED`  
- [ ] Status:

---

## 9. Edge cases & validation

### 9.1 Register with invalid email
```bash
http POST localhost:8080/api/auth/register \
  username=baduser \
  email=not-an-email \
  password=secret123
```
**Expected:** `400 Bad Request` — validation error  
- [ ] Status:

### 9.2 Register with short password
```bash
http POST localhost:8080/api/auth/register \
  username=baduser2 \
  email=bad@example.com \
  password=abc
```
**Expected:** `400 Bad Request` — password min 6 chars  
- [ ] Status:

### 9.3 Register with short username
```bash
http POST localhost:8080/api/auth/register \
  username=ab \
  email=ab@example.com \
  password=secret123
```
**Expected:** `400 Bad Request` — username min 3 chars  
- [ ] Status:

### 9.4 Order from non-existent restaurant
```bash
http POST localhost:8080/api/orders \
  "Authorization:Bearer $CUSTOMER_TOKEN" \
  restaurantId:=99999 \
  items:='[{"menuItemId": 1, "quantity": 1}]'
```
**Expected:** `4xx` error — restaurant not found  
- [ ] Status:

### 9.5 Get non-existent restaurant
```bash
http GET localhost:8080/api/restaurants/99999
```
**Expected:** `404 Not Found`  
- [ ] Status:

### 9.6 Place order with empty items list
```bash
http POST localhost:8080/api/orders \
  "Authorization:Bearer $CUSTOMER_TOKEN" \
  restaurantId:=$RESTAURANT_ID \
  items:='[]'
```
**Expected:** `400 Bad Request` — items must not be empty  
- [ ] Status:

---

## 10. Full happy-path smoke test (quick re-run)

Run this sequence end-to-end to confirm everything wires together:

```bash
# 1. Register + login
http POST localhost:8080/api/auth/register username=smoketest email=smoke@test.com password=secret123
# capture token

# 2. Verify profile
http GET localhost:8080/api/customers/me "Authorization:Bearer $TOKEN"

# 3. Browse restaurants
http GET localhost:8080/api/restaurants/search/all

# 4. View menu
http GET localhost:8080/api/restaurants/$RESTAURANT_ID/menu

# 5. Place order
http POST localhost:8080/api/orders "Authorization:Bearer $TOKEN" \
  restaurantId:=$RESTAURANT_ID \
  items:='[{"menuItemId": '$MENU_ITEM_ID', "quantity": 1}]'
# capture ORDER_ID

# 6. Check order
http GET localhost:8080/api/orders/$ORDER_ID "Authorization:Bearer $TOKEN"

# 7. List my orders
http GET localhost:8080/api/orders/my-orders "Authorization:Bearer $TOKEN"
```
- [ ] Status:

---

## Known port remappings (local dev)

| Service | Internal Docker port | Host port |
|---------|---------------------|-----------|
| API Gateway | 8080 | 8080 |
| Customer Service | 8081 | 8081 |
| Restaurant Service | 8082 | 8082 |
| Order Service | 8083 | 8083 |
| Delivery Service | 8084 | 8084 |
| PostgreSQL | 5432 | **5433** |
| RabbitMQ AMQP | 5672 | **5673** |
| RabbitMQ UI | 15672 | **15673** |

> PostgreSQL and RabbitMQ host ports were remapped because local instances were already running on the standard ports.

---

## Resuming this session

If you need to pick this up in a new session:
1. Run `docker compose ps` to confirm all 8 containers are up
2. Log back in with existing credentials to get fresh JWTs
3. Query the DB for existing IDs: `docker exec postgres psql -U postgres -d restaurant_db -c "SELECT id, name FROM restaurants;"`
4. Resume from whichever section's checkbox is not yet checked

---

## Fix Log — 2026-05-14 (ownerName bug)

#### BUG-2 — FIXED: `ownerName` always null in RestaurantResponse

**Root cause (3 parts):**
1. `CustomerDto` in restaurant-service only declared `id` and `username` — no name fields to carry
2. `CustomerClient` only had `getCustomerByUsername` — no method to look up an owner by their stored `ownerId`
3. `RestaurantResponse.fromEntity()` is a static mapper with no access to the Feign client; `ownerName` was never set after mapping

**Fix:**
- Added `firstName` and `lastName` to `CustomerDto` (restaurant-service)
- Added `getCustomerById(Long id)` to `CustomerClient` (maps to existing `GET /api/customers/{id}` SERVICE-role endpoint, auth handled by `FeignConfig` internal-token interceptor)
- Added `enrichOwnerName(RestaurantResponse)` helper in `RestaurantService` that calls `customerClient.getCustomerById(ownerId)` and sets `ownerName` as `"firstName lastName"` (falls back to `firstName` alone, then `username`)
- Applied enrichment in all five read paths: `createRestaurant`, `getById`, `searchByCity`, `searchByCuisine`, `getAllActive`
- The enrichment is best-effort: any `FeignException` is swallowed so a customer-service outage doesn't break restaurant reads

**Verified:** `GET /api/restaurants/1` and all three search endpoints now return `"ownerName": "Jan Owner"`
