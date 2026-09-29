# WireMock LAB

REST API mocking laboratory for QA Engineers.

WireMock LAB is a small local training environment for practicing backend and API testing without depending on a real external service.

The project uses WireMock as a mock HTTP server and models a simplified wallet/deposit service. An API client or application under test sends requests to WireMock, which returns predefined responses based on request properties.

The goal is to reproduce positive and negative integration scenarios locally and consistently.

## Project goals

- Learn the core WireMock concept of stub mappings.
- Practice request matching by HTTP method, URL, headers and JSON body.
- Simulate positive and negative REST API scenarios.
- Practice functional, negative and boundary-value testing.
- Explore authorization, duplicate transaction, timeout and unavailable-service scenarios.
- Use a deterministic mock service during backend and integration testing.

## What you can practice

- REST API testing
- Request and response validation
- HTTP status-code validation
- Header validation
- JSON body validation
- Authorization checks
- Boundary-value analysis
- Equivalence partitioning
- Duplicate transaction and idempotency scenarios
- Timeout and downstream-service failure handling
- Integration testing
- API debugging
- Test data preparation
- Creating and maintaining WireMock stubs

## Architecture

The LAB is intentionally simple:

```text
API client / application under test
              |
              | HTTP POST /wallet/deposit
              v
        WireMock :9090
              |
              | request matching
              v
       predefined stub
              |
              v
        HTTP response
```

WireMock runs in Docker.

Local port mapping:

```text
localhost:9090 -> WireMock:8080
```

The local `mappings` directory is mounted into the WireMock container so the stubs are loaded from the repository.

## Project structure

```text
WireMock-LAB/
├── docker-compose.yml
├── README.md
└── mappings/
    └── wallet/
        ├── 01-success-usd.json
        ├── 02-success-eur.json
        ├── 03-invalid-currency.json
        ├── 05-min-amount.json
        ├── 06-max-amount.json
        ├── 100-wallet-success.json
        ├── 11-missing-token.json
        ├── 12-invalid-token.json
        ├── 13-duplicate-transaction.json
        ├── 14-wallet-timeout.json
        └── 15-wallet-service-unavailable.json
```

### docker-compose.yml

Starts the WireMock container and exposes it on host port `9090`.

It also mounts:

- `./mappings` -> `/home/wiremock/mappings`
- `./__files` -> `/home/wiremock/__files`

### mappings/wallet

Contains WireMock mappings for the `POST /wallet/deposit` endpoint.

Each mapping defines request conditions and the response WireMock should return.

Depending on the scenario, matching can use:

- HTTP method
- URL
- headers
- JSONPath conditions
- the `X-Scenario` header

## Available scenarios

| Mapping | Scenario | Expected result |
|---|---|---:|
| `01-success-usd.json` | Valid USD deposit with required fields and Bearer authorization | `200` |
| `02-success-eur.json` | Valid EUR deposit | `200` |
| `03-invalid-currency.json` | Unsupported `BTC` currency | `400` |
| `05-min-amount.json` | Amount below `100` | `400` |
| `06-max-amount.json` | Explicit `max-amount` scenario | `409` |
| `100-wallet-success.json` | Amount from `100` through `10000` | `200` |
| `11-missing-token.json` | Missing-token scenario | `401` |
| `12-invalid-token.json` | Invalid-token scenario | `401` |
| `13-duplicate-transaction.json` | Duplicate transaction scenario | `409` |
| `14-wallet-timeout.json` | Wallet timeout with 15-second fixed delay | `504` |
| `15-wallet-service-unavailable.json` | Wallet service unavailable | `503` |

The mappings are intentionally request-driven. Some scenarios are selected by JSON body conditions, while others use the `X-Scenario` header.

## Requirements

You only need:

- Docker Desktop or Docker Engine
- Docker Compose v2
- Git

No Python, Node.js or Java runtime is required to start the current LAB.

## Run locally

### 1. Clone the repository

```bash
git clone https://github.com/Dantesssinferno/WireMock-LAB.git
cd WireMock-LAB
```

### 2. Start WireMock

```bash
docker compose up -d
```

Check the container:

```bash
docker compose ps
```

You should see the `wiremock` service running.

### 3. Open WireMock

Open:

http://localhost:9090

The LAB endpoint is:

```text
POST http://localhost:9090/wallet/deposit
```

## Send a basic request

A successful USD request:

```bash
curl -X POST http://localhost:9090/wallet/deposit \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer test-token" \
  -d '{"transactionId":"tx-001","userId":"user-001","amount":150,"currency":"USD"}'
```

Expected response:

```json
{
  "code": "SUCCESS",
  "message": "Deposit completed",
  "currency": "USD"
}
```

## Negative scenarios

### Unsupported currency

Send:

```json
"currency": "BTC"
```

Expected:

```text
400 Bad Request
code: UNSUPPORTED_CURRENCY
```

### Amount below the minimum

Try:

```json
"amount": 50
```

Expected:

```text
400 Bad Request
code: MIN_AMOUNT
```

### Invalid token

Add the header:

```text
X-Scenario: invalid-token
```

Expected:

```text
401 Unauthorized
code: INVALID_TOKEN
```

### Missing-token scenario

Add:

```text
X-Scenario: missing-token
```

Expected:

```text
401 Unauthorized
```

### Duplicate transaction

Add:

```text
X-Scenario: duplicate
```

Expected:

```text
409 Conflict
code: DUPLICATE_TRANSACTION
```

This is useful for practicing duplicate request handling and idempotency.

### Wallet service unavailable

Add:

```text
X-Scenario: service-unavailable
```

Expected:

```text
503 Service Unavailable
code: SERVICE_UNAVAILABLE
```

### Wallet timeout

Add:

```text
X-Scenario: timeout
```

WireMock waits 15 seconds before returning:

```text
504 Gateway Timeout
code: TIMEOUT
```

This scenario is useful for checking client timeout settings, retry behaviour and error handling.

## QA training scenarios

### 1. Happy path

Verify that a valid deposit:

- returns HTTP 200;
- returns the expected JSON body;
- returns the correct currency;
- satisfies the required request conditions.

### 2. Boundary-value analysis

The general success mapping accepts amounts from `100` through `10000`.

Practice:

```text
99
100
101
9999
10000
10001
```

Compare the actual behaviour with the expected business rule.

### 3. Equivalence partitioning

Create partitions for:

Currency:

```text
Supported -> USD, EUR
Unsupported -> BTC
```

Amount:

```text
Below minimum
Valid range
Above maximum
```

Authorization:

```text
Valid
Invalid
Missing / scenario-controlled
```

### 4. Duplicate transaction

Send requests representing the same logical transaction and investigate:

- whether the system retries;
- whether a duplicate financial operation can occur;
- how transaction identifiers are handled;
- how the `409` response is propagated.

### 5. Timeout

Use `X-Scenario: timeout` and investigate:

- client timeout;
- retry behaviour;
- repeated requests;
- final transaction state;
- user-facing error handling.

### 6. Service unavailable

Use `X-Scenario: service-unavailable` and check:

- error mapping;
- retry policy;
- fallback behaviour, if any;
- logging and diagnostics.

### 7. Authorization

Use the token-related scenarios to verify that an unauthenticated or invalid request is not treated as a successful deposit.

### 8. Contract testing

Validate:

- HTTP method;
- endpoint;
- required headers;
- JSON structure;
- required fields;
- data types;
- accepted values;
- status codes;
- response structure.

## WireMock concepts covered

### Stub mapping

A mapping defines the conditions under which WireMock should return a response.

For example, the USD success mapping checks the HTTP method and URL, requires JSON content type and Bearer authorization, and verifies the presence of transactionId, userId and amount plus the USD currency value.

### Request matching

This LAB demonstrates matching by method, URL, headers and JSONPath expressions.

### Priorities

Several mappings use explicit priorities. This is important when more than one mapping could potentially match the same request.

A useful exercise is to compare a specific negative scenario with a broader success mapping and understand which stub wins.

## Debugging

Show running containers:

```bash
docker compose ps
```

Follow WireMock logs:

```bash
docker compose logs -f wiremock
```

Stop the LAB:

```bash
docker compose down
```

Start again:

```bash
docker compose up -d
```

If port `9090` is busy, change only the host-side mapping in `docker-compose.yml`, for example:

```yaml
ports:
  - "9091:8080"
```

Then use:

http://localhost:9091

## Suggested learning path

```text
1. Start WireMock
       ↓
2. Send a valid deposit
       ↓
3. Validate status + body
       ↓
4. Change one request parameter
       ↓
5. Explore negative scenarios
       ↓
6. Practice boundary values
       ↓
7. Test timeout and 503
       ↓
8. Test duplicate transactions
       ↓
9. Inspect WireMock logs
       ↓
10. Create your own stub
```

For the final exercise, add a new mapping for a scenario that is not currently covered, for example:

- malformed or incomplete payload;
- zero amount;
- negative amount;
- another unsupported currency;
- `500 Internal Server Error`.

## Limitations

This is a training project, not a production wallet implementation.

The current repository provides a small set of static WireMock mappings. It does not implement a real wallet database, real payment processing or a real external provider.

The purpose is deterministic API behaviour for QA practice.

## Possible future extensions

- More HTTP error scenarios
- Malformed and incomplete payloads
- Payment and refund endpoints
- Stateful WireMock scenarios
- Automated API tests
- Postman collection
- Newman execution
- CI checks
- Request journal examples
- A small application under test that consumes the mock wallet API

## Repository

https://github.com/Dantesssinferno/WireMock-LAB

## License

Educational and QA practice project.
