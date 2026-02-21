# BlackRock Auto-Saving API
## Overview

This API implements an auto-saving strategy where expenses are rounded up to the next multiple of 100, and the difference (the "remanent") is invested for retirement. The system supports complex temporal constraints (q, p, k periods) and calculates inflation-adjusted returns for NPS and NIFTY 50 Index Fund investments.

## Project Structure

```
blackrock-challenge/
├── app/
│   ├── main.py           # FastAPI app and router setup
│   ├── models.py         # Pydantic request/response schemas
│   ├── services.py       # Core business logic (rounding, tax, returns)
│   └── routers/
│       ├── transactions.py   # Parse, validate, filter endpoints
│       ├── returns.py        # NPS and Index Fund return calculations
│       └── performance.py    # System metrics endpoint
├── test/
│   └── test_api.py       # Integration tests (pytest)
├── Dockerfile
├── requirements.txt
└── README.md
```

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/blackrock/challenge/v1/transactions:parse` | Enrich expenses with ceiling and remanent |
| POST | `/blackrock/challenge/v1/transactions:validator` | Validate transactions (negatives, duplicates) |
| POST | `/blackrock/challenge/v1/transactions:filter` | Apply q/p/k temporal constraints |
| POST | `/blackrock/challenge/v1/returns:nps` | Calculate NPS investment returns |
| POST | `/blackrock/challenge/v1/returns:index` | Calculate Index Fund returns |
| GET  | `/blackrock/challenge/v1/performance` | System performance metrics |

---

## Running Locally (without Docker)

### Prerequisites
- Python 3.11+

### Setup

```bash
# Clone / navigate to the project
cd blackrock-challenge

# Create a virtual environment
python -m venv venv
source venv/bin/activate   # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Start the server
uvicorn app.main:app --host 0.0.0.0 --port 5477 --reload
```

Visit `http://localhost:5477/docs` for the interactive Swagger UI.

---

## Running with Docker

### Build the image

```bash
# Replace {name-lastname} with your details
docker build -t blk-hacking-ind-{name-lastname} .
```

### Run the container

```bash
docker run -d -p 5477:5477 blk-hacking-ind-{name-lastname}
```

### Verify it's running

```bash
curl http://localhost:5477/
```

---

## Running Tests

Make sure the server is running first, then:

```bash
# Install test dependencies
pip install pytest requests

# Run all tests
python -m pytest test/test_api.py -v
```

---

## Business Logic Summary

### Rounding (ceiling/remanent)
- Each expense is rounded up to the **next multiple of 100**
- If amount is already an exact multiple, still rounds to the next one (e.g., 500 → 600)
- `remanent = ceiling - amount`

### Period Rules (Processing Order)
1. **q periods** – replace remanent with a fixed amount for transactions in that date range
2. **p periods** – add an extra amount on top of the (possibly q-adjusted) remanent
3. **k periods** – group transactions by date ranges and sum their remanents

### Investment Returns
- **NPS**: 7.11% p.a. compounded annually. Tax benefit calculated using simplified Indian tax slabs.
- **Index Fund (NIFTY 50)**: 14.49% p.a. compounded annually. No restrictions or tax benefits.
- Returns are **inflation-adjusted** using: `A_real = A / (1 + inflation)^t`
- Investment horizon: `t = 60 - current_age` (minimum 5 years if already 60+)

### Tax Slabs (Simplified)
| Income Range | Rate |
|-------------|------|
| ₹0 – ₹7,00,000 | 0% |
| ₹7,00,001 – ₹10,00,000 | 10% on amount above ₹7L |
| ₹10,00,001 – ₹12,00,000 | 15% on amount above ₹10L |
| ₹12,00,001 – ₹15,00,000 | 20% on amount above ₹12L |
| Above ₹15,00,000 | 30% on amount above ₹15L |

---

## Example Request (Returns - NPS)

```json
POST /blackrock/challenge/v1/returns:nps

{
  "age": 29,
  "wage": 50000,
  "inflation": 5.5,
  "q": [{"fixed": 0, "start": "2023-07-01 00:00:00", "end": "2023-07-31 23:59:59"}],
  "p": [{"extra": 25, "start": "2023-10-01 08:00:00", "end": "2023-12-31 19:59:59"}],
  "k": [{"start": "2023-01-01 00:00:00", "end": "2023-12-31 23:59:59"}],
  "transactions": [
    {"date": "2023-10-12 20:15:30", "amount": 250},
    {"date": "2023-02-28 15:49:20", "amount": 375},
    {"date": "2023-07-01 21:59:00", "amount": 620},
    {"date": "2023-12-17 08:09:45", "amount": 480}
  ]
}
```

Expected `amount` for the full-year k period: **145.0**
