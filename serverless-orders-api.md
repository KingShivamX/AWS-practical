# Serverless Orders API — Lambda + API Gateway + DynamoDB
## Complete Step-by-Step Guide (Practical 5 / Activity 2 — 15 marks)

Read this top to bottom and do it side by side on the AWS console.

**Region:** use **`us-east-1`** (US East, N. Virginia) for everything.

---

## 🗂️ PART 0 — WHAT WE ARE BUILDING

```
Client (Postman / curl / browser)
        │
        │  HTTP request  (GET / POST / PUT / DELETE)
        ▼
  API Gateway  ──Proxy Integration──▶  Lambda (orders-handler)
  /orders resource                           │
  4 methods                                  │  boto3
                                             ▼
                                        DynamoDB
                                        Table: Orders
                                        PK: OrderID (String)
                                        SK: Timestamp (String)
```

**Proxy Integration** means API Gateway passes the entire HTTP request (method, headers, body, query params) to Lambda as-is and returns whatever Lambda returns. Lambda is responsible for all logic.

**Composite key** means every row in DynamoDB is identified by TWO fields together: `OrderID` (Partition Key) + `Timestamp` (Sort Key). Neither alone is unique — both together are.

---

## 🗂️ PART 1 — RUBRIC CHECKLIST (keep ticking as you go)

| Rubric requirement | How it is met | Where |
|---|---|---|
| GET / POST / PUT / DELETE mapped to Lambda | 4 methods created in API Gateway on `/orders` resource | Step 4 |
| Proxy Integration | "Use Lambda Proxy Integration" checked for each method | Step 4 |
| DynamoDB composite key: OrderID (PK) + Timestamp (SK) | Table created with both keys | Step 1 |
| All 4 CRUD operations working | `create_order`, `get_order`, `update_order`, `delete_order` in Lambda | Step 2 |
| Data persisted in DynamoDB | `put_item`, `get_item`, `update_item`, `delete_item` using boto3 | Step 2 |
| `try/except` for DB timeouts | `ClientError` caught, `ProvisionedThroughputExceededException` returns 429 | Step 2 |
| Validate required fields | `required` loop checks `customerName`, `product`, `quantity` on POST | Step 2 |

---

## 🗂️ PART 2 — AWS CONSOLE SETUP

### STEP 1: Create DynamoDB Table

1. Console search bar → **DynamoDB** → **Create table**

**Every option explained:**

| Option | Value | Explanation |
|---|---|---|
| **Table name** | `Orders` | Case-sensitive. Lambda code uses `'Orders'` — must match exactly |
| **Partition key** | `OrderID` — type **String** | The primary lookup field. Every item must have this. |
| **Sort key** | `Timestamp` — type **String** | Second part of the composite key. Allows multiple orders with same OrderID (e.g. versioning). Together with OrderID it uniquely identifies a row. |
| **Table settings** | `Default settings` | Fine for lab — uses On-Demand capacity, no provisioned throughput to manage |
| **Table class** | `DynamoDB Standard` | Default. `Standard-IA` is for infrequently accessed data (cheaper storage, higher read cost) |
| **Capacity mode** | `On-demand` (default) | AWS auto-scales read/write. No `ProvisionedThroughputExceededException` errors in testing. Alternative: `Provisioned` = you set fixed RCU/WCU — can throttle if exceeded |
| **Encryption at rest** | `Owned by Amazon DynamoDB` | Default. Your data is encrypted automatically. |
| **Tags** | Optional | Skip for lab |

2. Click **Create table**
3. Wait until **Status** shows **Active** (usually 30 seconds)

> **Composite key rule (viva point):** You can have 1000 items all with `OrderID = "abc"` as long as each has a different `Timestamp`. DynamoDB stores them sorted by `Timestamp` within that partition. This enables range queries (e.g. "all orders by customer in the last hour").

---

### STEP 2: Create the Lambda Function

#### 2a — Create function

1. Console → **Lambda** → **Create function**

| Option | Value | Explanation |
|---|---|---|
| **Author from scratch** | Selected | We write code directly |
| **Function name** | `orders-handler` | Must match what you reference in API Gateway |
| **Runtime** | `Python 3.12` | Latest stable Python on Lambda. boto3 (AWS SDK) is pre-installed — no need to upload it |
| **Architecture** | `x86_64` | Default. `arm64` (Graviton) is cheaper but not needed here |
| **Execution role** | `Create a new role with basic Lambda permissions` | Creates a role that can write to CloudWatch Logs. We add DynamoDB access next |
| **Advanced settings** | Leave default | No VPC, no code signing needed for this lab |

2. Click **Create function**

---

#### 2b — Paste the Lambda code

Click the **Code** tab. Select all the placeholder code and replace with this:

```python
import json
import boto3
import uuid
from datetime import datetime
from botocore.exceptions import ClientError

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Orders')


def lambda_handler(event, context):
    method = event['httpMethod']

    try:
        if method == 'POST':
            return create_order(event)
        elif method == 'GET':
            return get_order(event)
        elif method == 'PUT':
            return update_order(event)
        elif method == 'DELETE':
            return delete_order(event)
        else:
            return build_response(405, {'error': 'Method not allowed'})

    except ClientError as e:
        code = e.response['Error']['Code']
        if code == 'ProvisionedThroughputExceededException':
            return build_response(429, {'error': 'Too many requests, try again later'})
        return build_response(500, {'error': str(e)})

    except Exception as e:
        return build_response(500, {'error': str(e)})


# ── CREATE (POST) ────────────────────────────────────────
def create_order(event):
    body = parse_body(event)

    # Required field validation
    required = ['customerName', 'product', 'quantity']
    for field in required:
        if field not in body:
            return build_response(400, {'error': f'Missing required field: {field}'})

    order_id  = str(uuid.uuid4())
    timestamp = datetime.utcnow().isoformat()

    item = {
        'OrderID':      order_id,
        'Timestamp':    timestamp,
        'customerName': body['customerName'],
        'product':      body['product'],
        'quantity':     body['quantity'],
        'status':       body.get('status', 'PENDING')
    }

    table.put_item(Item=item)
    return build_response(201, {'message': 'Order created', 'order': item})


# ── READ (GET) ───────────────────────────────────────────
def get_order(event):
    params = event.get('queryStringParameters') or {}
    order_id  = params.get('OrderID')
    timestamp = params.get('Timestamp')

    if not order_id or not timestamp:
        return build_response(400, {'error': 'Missing OrderID or Timestamp query params'})

    result = table.get_item(Key={'OrderID': order_id, 'Timestamp': timestamp})
    item   = result.get('Item')

    if not item:
        return build_response(404, {'error': 'Order not found'})

    return build_response(200, item)


# ── UPDATE (PUT) ─────────────────────────────────────────
def update_order(event):
    body = parse_body(event)

    order_id  = body.get('OrderID')
    timestamp = body.get('Timestamp')

    if not order_id or not timestamp:
        return build_response(400, {'error': 'Missing OrderID or Timestamp in body'})

    result = table.update_item(
        Key={'OrderID': order_id, 'Timestamp': timestamp},
        UpdateExpression='SET #s = :s, #p = :p, #q = :q',
        ExpressionAttributeNames={'#s': 'status', '#p': 'product', '#q': 'quantity'},
        ExpressionAttributeValues={
            ':s': body.get('status', 'UPDATED'),
            ':p': body.get('product', 'N/A'),
            ':q': body.get('quantity', 0)
        },
        ConditionExpression='attribute_exists(OrderID)',
        ReturnValues='ALL_NEW'
    )
    return build_response(200, {'message': 'Order updated', 'order': result['Attributes']})


# ── DELETE (DELETE) ──────────────────────────────────────
def delete_order(event):
    body = parse_body(event)

    order_id  = body.get('OrderID')
    timestamp = body.get('Timestamp')

    if not order_id or not timestamp:
        return build_response(400, {'error': 'Missing OrderID or Timestamp in body'})

    table.delete_item(Key={'OrderID': order_id, 'Timestamp': timestamp})
    return build_response(200, {'message': 'Order deleted'})


# ── HELPERS ──────────────────────────────────────────────
def parse_body(event):
    try:
        return json.loads(event.get('body') or '{}')
    except json.JSONDecodeError:
        return {}


def build_response(status_code, body):
    return {
        'statusCode': status_code,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps(body)
    }
```

3. Click **Deploy** (orange button, top right of code editor)
4. Wait for "Changes deployed" confirmation

---

#### 2c — Give Lambda permission to access DynamoDB

1. Lambda → **Configuration** tab → **Permissions**
2. Under **Execution role**, click the role name link (opens IAM in a new tab)
3. IAM → **Add permissions** → **Attach policies**
4. Search: `AmazonDynamoDBFullAccess` → check it → **Add permissions**

> **Why this?** Lambda runs with an IAM role. By default it can only write to CloudWatch Logs. It cannot touch DynamoDB until you attach this policy. `AmazonDynamoDBFullAccess` covers `GetItem`, `PutItem`, `UpdateItem`, `DeleteItem`, and everything else.

---

### STEP 3: Test Lambda directly (before API Gateway)

This catches bugs before you add API Gateway complexity.

1. Lambda → **Test** tab → **Create new test event**
2. **Event name**: `test-post`
3. Paste:

```json
{
  "httpMethod": "POST",
  "body": "{\"customerName\": \"Shivam\", \"product\": \"Laptop\", \"quantity\": 2}",
  "queryStringParameters": null
}
```

4. Click **Test**
5. Expected result: **Execution result: succeeded** and response body:
```json
{
  "statusCode": 201,
  "body": "{\"message\": \"Order created\", \"order\": {...}}"
}
```

6. Go to **DynamoDB** → **Tables** → **Orders** → **Explore table items** → confirm the row is there.

**Copy the `OrderID` and `Timestamp` values** from the DynamoDB item — you'll need them for GET / PUT / DELETE tests.

---

### STEP 4: Create API Gateway

#### 4a — Create the REST API

1. Console → **API Gateway** → **Create API**

| Option | Value | Explanation |
|---|---|---|
| **API type** | `REST API` (click **Build** under it) | REST API uses resources + methods. `HTTP API` is simpler but less configurable — exam uses REST. **Do not** choose WebSocket or HTTP API. |
| **API name** | `OrdersAPI` | Any name |
| **API endpoint type** | `Regional` | For single-region labs. `Edge-optimized` uses CloudFront (global CDN, higher latency for local testing). `Private` = only reachable inside VPC |
| **Description** | Optional | |

2. Click **Create API**

---

#### 4b — Create the `/orders` resource

1. In the left tree you see `/` (root). Click it to select it.
2. Click **Create resource** (button in top right area of the resources panel)

| Option | Value | Explanation |
|---|---|---|
| **Resource path** | `/` | Parent — should already be `/` |
| **Resource name** | `orders` | Becomes `/orders` in the URL |
| **Resource path** (auto-filled) | `/orders` | Final path |
| **CORS** | Leave unchecked for now | Enable only if a browser frontend needs to call this API |

3. Click **Create resource**

---

#### 4c — Create the 4 methods (do this 4 times)

Make sure `/orders` is **selected** (highlighted) in the resource tree on the left.

Click **Create method** for each of: **GET**, **POST**, **PUT**, **DELETE**

For **each** method, fill in:

| Option | Value | Explanation |
|---|---|---|
| **Method type** | `GET` (then repeat for POST, PUT, DELETE) | The HTTP verb this method handles |
| **Integration type** | `Lambda function` | Lambda is the backend that handles the request |
| **Lambda proxy integration** | ✅ **Enable** (check the box) | **This is the key rubric point.** Proxy = API Gateway passes the raw HTTP event to Lambda unchanged. Lambda controls the response format. Without this, API Gateway parses/transforms the request itself |
| **Lambda function** | `us-east-1` / `orders-handler` | Region dropdown + function name. Type `orders` and it should autocomplete |
| **Default timeout** | Leave default (29 seconds) | Max time API Gateway waits for Lambda to respond |

> ⚠️ After clicking **Save**, AWS shows a popup: **"Add permission to Lambda function"** — click **OK**. This lets API Gateway invoke Lambda.

Repeat for all 4 methods. When done, the `/orders` resource should show GET, POST, PUT, DELETE all listed under it.

---

#### 4d — Deploy the API

1. Click **Deploy API** button (top right)

| Option | Value | Explanation |
|---|---|---|
| **Stage** | `[New stage]` | Create a new deployment stage |
| **Stage name** | `prod` | Can be anything: `dev`, `v1`, `prod`. Becomes part of the URL. |
| **Stage description** | Optional | |
| **Deployment description** | Optional | |

2. Click **Deploy**
3. You get an **Invoke URL** like:
```
https://abc123xyz.execute-api.us-east-1.amazonaws.com/prod
```
4. Your full API endpoint is:
```
https://abc123xyz.execute-api.us-east-1.amazonaws.com/prod/orders
```

> **Save this URL** — you need it for all tests below.

---

#### 4e — API Gateway console options explained (important for viva)

**Stage settings** (API Gateway → Stages → `prod`):

| Setting | What it controls | Interview angle |
|---|---|---|
| **Stage variables** | Key-value pairs available to Lambda via `event['stageVariables']` | Used to point different stages at different Lambda aliases or tables |
| **Throttling** | Rate limit (requests/sec) and burst limit | Prevents abuse; returns 429 if exceeded |
| **Cache settings** | Cache GET responses at the API level for N seconds | Reduces Lambda invocations; not free |
| **Logs/tracing** | CloudWatch log group + X-Ray tracing | Enable for debugging: see every request/response |
| **Client certificate** | Mutual TLS between API Gateway and Lambda | Advanced security |

**Method settings** (per-method):

| Setting | What it controls |
|---|---|
| **Authorization** | `NONE` (public), `IAM`, `Cognito User Pool`, `Lambda authorizer` |
| **API Key required** | Requires caller to send `x-api-key` header |
| **Request validator** | API Gateway validates body/params before calling Lambda |
| **Integration timeout** | 50ms–29s; Lambda has 15 min max but API Gateway only waits 29s |

---

## 🧪 PART 3 — TESTING ALL 4 OPERATIONS

Use the **API Gateway Test** tab in the console, **Postman**, or `curl`. Examples below use `curl` — run in any terminal.

Replace `YOUR_URL` with your Invoke URL + `/orders`.

---

### TEST 1 — POST (Create an order)

```bash
curl -X POST https://YOUR_URL/prod/orders \
  -H "Content-Type: application/json" \
  -d '{"customerName": "Shivam", "product": "Laptop", "quantity": 2}'
```

**Expected response (201):**
```json
{
  "message": "Order created",
  "order": {
    "OrderID": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "Timestamp": "2026-05-12T15:30:00.123456",
    "customerName": "Shivam",
    "product": "Laptop",
    "quantity": 2,
    "status": "PENDING"
  }
}
```

> **Copy the `OrderID` and `Timestamp` from this response** — needed for next 3 tests.

---

### TEST 2 — GET (Read the order)

```bash
curl "https://YOUR_URL/prod/orders?OrderID=f47ac10b-58cc-4372-a567-0e02b2c3d479&Timestamp=2026-05-12T15:30:00.123456"
```

> Put the URL in quotes when using `&` in a shell — `&` means "run in background" otherwise.

**Expected response (200):**
```json
{
  "OrderID": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "Timestamp": "2026-05-12T15:30:00.123456",
  "customerName": "Shivam",
  "product": "Laptop",
  "quantity": 2,
  "status": "PENDING"
}
```

---

### TEST 3 — PUT (Update the order)

```bash
curl -X PUT https://YOUR_URL/prod/orders \
  -H "Content-Type: application/json" \
  -d '{
    "OrderID": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "Timestamp": "2026-05-12T15:30:00.123456",
    "status": "SHIPPED",
    "product": "Laptop",
    "quantity": 3
  }'
```

**Expected response (200):**
```json
{
  "message": "Order updated",
  "order": {
    "OrderID": "...",
    "Timestamp": "...",
    "status": "SHIPPED",
    "product": "Laptop",
    "quantity": 3
  }
}
```

---

### TEST 4 — DELETE (Remove the order)

```bash
curl -X DELETE https://YOUR_URL/prod/orders \
  -H "Content-Type: application/json" \
  -d '{
    "OrderID": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "Timestamp": "2026-05-12T15:30:00.123456"
  }'
```

**Expected response (200):**
```json
{
  "message": "Order deleted"
}
```

---

### TEST 5 — Validation errors (proves error handling works)

**Missing required field (POST without `product`):**
```bash
curl -X POST https://YOUR_URL/prod/orders \
  -H "Content-Type: application/json" \
  -d '{"customerName": "Shivam", "quantity": 2}'
```
Expected: `400` `{"error": "Missing required field: product"}`

**GET without params:**
```bash
curl "https://YOUR_URL/prod/orders"
```
Expected: `400` `{"error": "Missing OrderID or Timestamp query params"}`

**GET a non-existent order:**
```bash
curl "https://YOUR_URL/prod/orders?OrderID=doesnotexist&Timestamp=2000-01-01"
```
Expected: `404` `{"error": "Order not found"}`

---

### Using API Gateway Test tab (no curl needed)

1. API Gateway → your API → **Resources** → click a method (e.g. **POST**)
2. Click **Test** (lightning bolt icon)
3. Fill in:
   - **Request body** (for POST/PUT/DELETE): paste the JSON
   - **Query strings** (for GET): `OrderID=xxx&Timestamp=yyy`
4. Click **Test** — response shows Status, Headers, Body, Logs all in one panel

---

## 🔵 PART 4 — CODE EXPLAINED (LINE BY LINE for viva)

### Lambda handler routing

```python
method = event['httpMethod']   # 'GET', 'POST', 'PUT', or 'DELETE'
```

API Gateway Proxy Integration puts the HTTP method in `event['httpMethod']`. The handler reads it and routes to the right function — this is the "single Lambda for all methods" pattern.

---

### Error handling layers

```python
except ClientError as e:
    code = e.response['Error']['Code']
    if code == 'ProvisionedThroughputExceededException':
        return build_response(429, {'error': 'Too many requests, try again later'})
    return build_response(500, {'error': str(e)})
except Exception as e:
    return build_response(500, {'error': str(e)})
```

| Exception | What triggers it | Response |
|---|---|---|
| `ProvisionedThroughputExceededException` | DynamoDB table is provisioned and you exceed its read/write units | 429 (Too Many Requests) |
| Other `ClientError` | DynamoDB auth issues, table doesn't exist, condition failed | 500 |
| Generic `Exception` | JSON parse errors, missing key, anything else | 500 |

> **Viva point:** Always catch `ClientError` from `botocore.exceptions`, not a generic `Exception` for AWS SDK errors. `ClientError` is the AWS SDK base class for all DynamoDB/S3/etc errors.

---

### Composite key usage

Every DynamoDB operation on this table needs **both** keys:

```python
# GET
table.get_item(Key={'OrderID': order_id, 'Timestamp': timestamp})

# UPDATE
table.update_item(Key={'OrderID': order_id, 'Timestamp': timestamp}, ...)

# DELETE
table.delete_item(Key={'OrderID': order_id, 'Timestamp': timestamp})
```

If you pass only one key, DynamoDB returns `ValidationException: The provided key element does not match the schema`.

---

### ConditionExpression on PUT

```python
ConditionExpression='attribute_exists(OrderID)'
```

This means: **only update the item if it already exists**. If you try to update an `OrderID` that was never created, the operation fails with `ConditionalCheckFailedException` instead of silently creating a blank item. This prevents "phantom updates".

---

### build_response helper

```python
def build_response(status_code, body):
    return {
        'statusCode': status_code,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps(body)
    }
```

**Proxy Integration requires this exact format.** If you return anything else (e.g. just a dict), API Gateway shows a 502 Bad Gateway. The three required keys are `statusCode` (int), `headers` (dict), and `body` (**string**, not dict — use `json.dumps`).

---

### parse_body helper

```python
def parse_body(event):
    try:
        return json.loads(event.get('body') or '{}')
    except json.JSONDecodeError:
        return {}
```

API Gateway sends the HTTP request body as a **string** inside `event['body']`. You must `json.loads()` it to get a dict. If the body is `null` or malformed JSON, this returns `{}` safely instead of crashing.

---

## 🧠 PART 5 — VIVA QUESTIONS + ANSWERS

**Q: What is Lambda Proxy Integration?**
> API Gateway passes the entire HTTP request — method, headers, body, path, query params — to Lambda as a single JSON event object. Lambda returns a response with `statusCode`, `headers`, and `body`. API Gateway forwards that response directly to the client. Without proxy integration, you define mapping templates to transform the request/response manually.

**Q: What is the difference between Partition Key and Sort Key in DynamoDB?**
> Partition Key (PK) determines which partition (server) stores the item. Sort Key (SK) sorts items within that partition. Together they form a **composite primary key** — every item must have a unique PK+SK combination. You can query all items with the same PK and filter/sort by SK.

**Q: Why use `OrderID` (UUID) as Partition Key instead of `customerName`?**
> UUID is globally unique and distributes data evenly across partitions (high cardinality). `customerName` would put all orders for one customer on one partition (hot partition), which causes throttling at scale.

**Q: What happens if `ValidateService` fails?**
> That's for CodeDeploy — not this practical. For Lambda: if Lambda throws an unhandled exception, API Gateway returns 502. If Lambda returns a 4xx or 5xx statusCode in the response object, that is passed to the client.

**Q: What HTTP status codes does this API return?**

| Code | When |
|---|---|
| 200 | Successful GET, PUT, DELETE |
| 201 | Successful POST (item created) |
| 400 | Missing required fields or missing key params |
| 404 | Item not found in DynamoDB |
| 405 | HTTP method not handled |
| 429 | DynamoDB throughput exceeded |
| 500 | Any unexpected server error |

**Q: What is `uuid.uuid4()`?**
> Generates a random, globally unique 128-bit identifier. Used as `OrderID` so each order has a unique key without needing a database sequence/counter. Example: `f47ac10b-58cc-4372-a567-0e02b2c3d479`.

**Q: Why does `event['body']` need `json.loads()`?**
> API Gateway sends the HTTP body as a **string**, not a parsed object. Even if the client sends `{"product": "Laptop"}`, Lambda receives `'{"product": "Laptop"}'` — a string. `json.loads()` converts it to a Python dict.

**Q: What is `ReturnValues='ALL_NEW'` in `update_item`?**
> After updating, DynamoDB returns the entire updated item instead of just the changed attributes. `ALL_OLD` returns the pre-update values. `NONE` returns nothing (default, no extra cost).

**Q: What is `ExpressionAttributeNames` in the update call?**
> DynamoDB has reserved words (like `status`, `name`, `count`). If your attribute name matches a reserved word, the query fails. `ExpressionAttributeNames` maps a placeholder (e.g. `#s`) to the real name (`status`) to avoid conflicts.

**Q: What is `ConditionExpression` on `update_item`?**
> A guard condition that must be true for the operation to proceed. `attribute_exists(OrderID)` means "only update if this item exists". If the condition fails, DynamoDB throws `ConditionalCheckFailedException` instead of creating a blank item.

**Q: What is the difference between REST API and HTTP API in API Gateway?**
> REST API is the older, more feature-rich type (supports request/response transformation, usage plans, API keys, WAF integration). HTTP API is newer, simpler, cheaper, but fewer features. Exam typically asks about REST API.

**Q: What is a Stage in API Gateway?**
> A named snapshot of your API deployment (e.g. `dev`, `prod`). The stage name becomes part of the URL. You can have different configurations per stage (throttling, logging, stage variables pointing to different Lambda functions).

**Q: Can one Lambda function handle multiple routes?**
> Yes — this is the "monolithic Lambda" pattern. One function routes based on `event['httpMethod']` or `event['path']`. Alternative: separate Lambda per method (microservice pattern). Monolithic is simpler; micro is more scalable and independently deployable.

---

## 🚀 PART 6 — AWS CONSOLE VERIFICATION MAP

| Service | Navigation | What to verify |
|---|---|---|
| **DynamoDB** | DynamoDB → Tables → Orders → **Explore table items** | Rows created after POST; deleted after DELETE |
| **Lambda** | Lambda → orders-handler → **Monitor** tab | Invocation count, error count, duration graph |
| **Lambda logs** | Lambda → Monitor → **View CloudWatch logs** | Per-request log output; error stack traces |
| **API Gateway** | API Gateway → OrdersAPI → **Stages** → prod | Invoke URL; check stage settings |
| **API Gateway logs** | API Gateway → Stages → prod → **Logs/Tracing** tab | Enable `CloudWatch Logs` here if you want request-level logging |
| **IAM** | IAM → Roles → find `orders-handler-role-*` | Confirm `AmazonDynamoDBFullAccess` is attached |
| **CloudWatch** | CloudWatch → Log groups → `/aws/lambda/orders-handler` | Lambda execution logs; find errors here first |

---

## ⚡ PART 7 — COMMON ERRORS AND FIXES

| Error | Cause | Fix |
|---|---|---|
| `502 Bad Gateway` from API Gateway | Lambda returned invalid response format (not `{statusCode, headers, body}`) | Make sure `build_response` is always called; `body` must be a **string** (`json.dumps`), not a dict |
| `AccessDeniedException` in Lambda logs | Lambda role doesn't have DynamoDB permission | IAM → Lambda role → attach `AmazonDynamoDBFullAccess` |
| `ResourceNotFoundException: Table: Orders not found` | Lambda code table name `'Orders'` doesn't match the DynamoDB table name | Check exact table name in DynamoDB; case-sensitive |
| `ValidationException: The provided key element does not match the schema` | Passing only one key to `get_item` / `delete_item` on a composite-key table | Always pass both `OrderID` and `Timestamp` in the `Key` dict |
| `ConditionalCheckFailedException` on PUT | `ConditionExpression='attribute_exists(OrderID)'` failed — item doesn't exist | First POST the order, then PUT it |
| `{"message": "Internal server error"}` with no detail | Unhandled exception in Lambda (check CloudWatch Logs) | Go to CloudWatch → `/aws/lambda/orders-handler` → latest log stream |
| `{"message": "Missing Authentication Token"}` | Calling a URL that doesn't exist (wrong path or not deployed) | Check the URL includes `/prod/orders` exactly; redeploy if you added methods after last deploy |
| DynamoDB item not appearing after POST | Lambda test succeeded but API Gateway not deployed after changes | API Gateway → **Deploy API** → `prod` after every change |

---

## ✅ PART 8 — FINAL SUBMISSION CHECKLIST

Before submitting, run through this end to end:

```
□ DynamoDB table "Orders" exists with OrderID (PK, String) + Timestamp (SK, String)
□ Lambda "orders-handler" deployed, Python 3.12
□ Lambda role has AmazonDynamoDBFullAccess
□ API Gateway "OrdersAPI" has /orders resource
□ /orders has GET, POST, PUT, DELETE methods
□ ALL 4 methods use Lambda Proxy Integration (checked)
□ API deployed to "prod" stage
□ POST creates order → 201 response, item in DynamoDB
□ GET retrieves order → 200 with correct data
□ PUT updates order → 200, DynamoDB item changed
□ DELETE removes order → 200, item gone from DynamoDB
□ POST without customerName/product/quantity → 400 error
□ GET without params → 400 error
□ GET non-existent order → 404 error
```

---

That's the complete Serverless Orders API from zero. The one Lambda function handles all 4 methods through Proxy Integration, DynamoDB uses the composite key, and every rubric point is covered.
