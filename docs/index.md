# Basket Trading

## 1. Introduction

The **Basket Trading** module is used to import basket configuration and stock allocation data from a JSON file into the MySQL database.

The import process reads basket-level information and associated stock details, validates the supplied data, maps stock ISINs to their corresponding NSE `InstrumentId`, and stores the processed information in the database.

The module is designed to provide a controlled and consistent process for creating or refreshing basket trading data in the database.

---

## 2. Document Overview

This document describes the Basket Trading data import process and provides the operational information required to use the basket import utility.

The document covers:

- Basket Trading overview
- Import utility components
- Input files
- Basket JSON structure
- Basket-level fields
- Stock-level fields
- Weightage validation
- Fund volatility validation
- NSE instrument mapping
- ISIN validation
- Instrument ID assignment
- Existing basket data handling
- Database insertion
- Transaction processing
- Error handling
- Database verification
- Troubleshooting
- Operational checklist

This document is intended for developers, system administrators, deployment engineers, and users responsible for managing Basket Trading configuration.

---

## 3. Import Utility

The Basket Trading import process is implemented using:

```text
DumpBasketData.py
```

The utility reads the basket configuration from:

```text
basket_dump.json
```

and uses:

```text
instruments.csv
```

to resolve NSE stock instruments.

The default input files are defined in the utility as `basket_dump.json` and `instruments.csv`.

---

## 4. System Components

The Basket Trading import process consists of the following components:

| Component | Description |
|---|---|
| `DumpBasketData.py` | Main basket data import utility |
| `basket_dump.json` | Contains basket and stock configuration |
| `instruments.csv` | NSE instrument master used for ISIN mapping |
| `BasketTradings` | Stores basket-level information |
| `BasketStockWeightages` | Stores stock-level allocation information |

---

## 5. Overall Data Flow

The Basket Trading import process follows the flow below:

<div class="data-flow">
<pre>
                  ┌──────────────────────┐
                  │   basket_dump.json   │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │   DumpBasketData.py  │
                  └──────────┬───────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
    ┌──────────────────┐         ┌──────────────────────┐
    │ Basket Validation│         │   instruments.csv    │
    │                  │         │   NSE Instruments    │
    └──────────┬───────┘         └──────────┬───────────┘
               │                            │
               │                            ▼
               │                    ISIN → InstrumentId
               │                            │
               └─────────────┬──────────────┘
                             ▼
                  ┌──────────────────────┐
                  │ Database Transaction │
                  └──────────┬───────────┘
                             │
                  ┌──────────┴───────────┐
                  │                      │
                  ▼                      ▼
        ┌──────────────────┐   ┌────────────────────────┐
        │  BasketTradings  │   │ BasketStockWeightages  │
        └──────────────────┘   └────────────────────────┘
</pre>
</div>

---

## 6. Database

The Basket Trading utility uses the MySQL database:

```text
Blitz
```

The database connection is configured in `DumpBasketData.py`.

The configured database connection contains the database host, port, username, password, and database name.

> **Security Note:** Database credentials must be handled securely. Do not expose passwords in documentation, source control, or publicly accessible configuration files.

---

## 7. Database Tables

The import process uses two database tables.

### 7.1 BasketTradings

The `BasketTradings` table stores the basket-level configuration.

The import utility inserts the following information:

| Field | Description |
|---|---|
| `FundName` | Name of the basket |
| `FundOwner` | Owner of the basket |
| `FundDetails` | Detailed basket information |
| `Overview` | Basket overview |
| `FundVolatility` | Basket volatility |
| `SubscriptionFee` | Basket subscription fee |
| `CreatedAt` | Creation timestamp |
| `UpdatedAt` | Last update timestamp |
| `MinimumInvestment` | Minimum investment amount |
| `KeyWords` | Basket keywords |
| `CAGR` | CAGR value |

These fields are used by the basket insertion operation.

---

### 7.2 BasketStockWeightages

The `BasketStockWeightages` table stores stock allocation information associated with a basket.

The import utility inserts:

| Field | Description |
|---|---|
| `FundName` | Basket name |
| `FundOwner` | Basket owner |
| `StockSymbol` | Stock symbol |
| `Weightage` | Stock allocation percentage |
| `ISIN` | Stock ISIN |
| `Sector` | Stock sector |
| `InstrumentId` | NSE instrument identifier |



---

## 8. Input Files

The utility requires two input files.

### 8.1 Basket JSON

```text
basket_dump.json
```

This file contains:

- Basket information
- Stock information

### 8.2 Instrument Master

```text
instruments.csv
```

This file provides the mapping between:

```text
ISIN
    ↓
InstrumentId
```

Only NSE instruments are considered during the mapping process. 

---

## 9. Basket JSON Structure

The JSON file contains two primary sections:

```text
basket
stocks
```

The general structure is:

```json
{
  "basket": {
    "FundName": "testing_basket",
    "FundOwner": "Priyanshu",
    "Operation": "REBALANCE",
    "FundDetails": "...",
    "Overview": "...",
    "FundVolatility": "High",
    "SubscriptionFee": 4999,
    "CreatedAt": "2026-02-24 10:00:00",
    "UpdatedAt": "2026-02-24 10:00:00",
    "MinimumInvestment": 50000,
    "KeyWords": "Quantitative,Equity,India,Growth",
    "CAGR": 18.5
  },
  "stocks": [
    {
      "StockSymbol": "RELIANCE",
      "Weightage": 12,
      "ISIN": "INE002A01018",
      "Sector": "Energy"
    }
  ]
}
```

The utility reads the `basket` object and the optional `stocks` array from the JSON file. 

---

## 10. Basket Information

The `basket` object contains information describing the basket.

| Field | Description |
|---|---|
| `FundName` | Unique name used to identify the basket |
| `FundOwner` | Owner associated with the basket |
| `Operation` | Operation information provided in the JSON |
| `FundDetails` | Detailed description of the basket |
| `Overview` | Short description of the basket |
| `FundVolatility` | Volatility classification |
| `SubscriptionFee` | Subscription fee |
| `CreatedAt` | Basket creation timestamp |
| `UpdatedAt` | Basket update timestamp |
| `MinimumInvestment` | Minimum investment amount |
| `KeyWords` | Basket keywords |
| `CAGR` | CAGR value |

`FundName` and `FundOwner` are required for processing. 

> `Operation` is read as part of the JSON basket object but is not included in the current `BasketTradings` INSERT operation.

---

## 11. Stock Information

The `stocks` section contains the individual stocks included in the basket.

Each stock contains:

| Field | Description |
|---|---|
| `StockSymbol` | Stock trading symbol |
| `Weightage` | Portfolio allocation |
| `ISIN` | Security identification number |
| `Sector` | Stock sector |

During processing, the utility adds:

```text
InstrumentId
```

to valid stock records. 

---

## 12. Basket Validation

Before database insertion, the basket data is validated.

The utility performs the following validations:

```text
FundName
FundOwner
FundVolatility
Total Stock Weightage
Stock ISIN
```

The validation process is performed before the database transaction begins. 


---

## 13. FundName and FundOwner Validation

Both fields are required:

```text
FundName
FundOwner
```

If either field is missing or empty, the import process stops with a validation error. 


---

## 14. Fund Volatility Validation

The supported volatility values are:

```text
High
Medium
Low
```

The supplied value is normalized using title case.

For example:

```text
high
HIGH
high
```

is normalized to:

```text
High
```

If the supplied value is not supported, the validation fails. 


---

## 15. Weightage Validation

The total weightage of all stocks must equal:

```text
100
```

The utility calculates the total using all stock `Weightage` values.

If the calculated total is not `100`, the import raises a validation error. 


Example:

```text
Stock 1    20%
Stock 2    30%
Stock 3    25%
Stock 4    25%
----------------
Total     100%
```

---

## 16. Instrument Master Processing

The instrument master is loaded from:

```text
instruments.csv
```

The utility performs the following operations:

1. Reads the CSV using Pandas.
2. Cleans the `ISIN` values.
3. Cleans the `Exchange` values.
4. Filters the records to NSE.
5. Creates an ISIN-to-InstrumentId mapping.




The mapping can be represented as:

```text
ISIN
 │
 ▼
NSE Instrument Master
 │
 ▼
InstrumentId
```

---

## 17. Stock ISIN Validation

Each stock is checked against the NSE instrument mapping.

### Missing ISIN

If a stock does not contain an ISIN:

```text
Stock <number> missing ISIN
```

The stock is excluded from the valid stock list.

### Invalid ISIN

If the ISIN is not found in the NSE instrument mapping:

```text
Invalid ISIN: <ISIN>
```

The stock is excluded.

### Valid ISIN

For a valid ISIN, the utility assigns the corresponding:

```text
InstrumentId
```




---

## 18. Database Processing

Once validation is complete and valid stocks are available, the utility connects to the MySQL database.

A database transaction is then started.

The processing sequence is:

```text
Start Transaction
       │
       ▼
Delete Existing Basket Data
       │
       ▼
Insert Basket
       │
       ▼
Insert Stock Weightages
       │
       ▼
Commit Transaction
```




---

## 19. Existing Basket Data

Before inserting the new basket configuration, existing data for the same basket is removed.

The basket is identified using:

```text
FundName
FundOwner
```

The deletion occurs in the following order:

### Step 1

Delete stock records from:

```text
BasketStockWeightages
```

### Step 2

Delete the basket record from:

```text
BasketTradings
```




This allows the new basket configuration to replace the existing configuration.

---

## 20. Basket Insertion

After existing data is removed, the new basket information is inserted into:

```text
BasketTradings
```

The `FundDetails` value is serialized into JSON before being stored. 


---

## 21. Stock Weightage Insertion

After the basket record is inserted, the valid stock records are inserted into:

```text
BasketStockWeightages
```

The utility uses a bulk database operation to insert the stock records. 


---

## 22. Transaction and Rollback

The database operations are performed as a single transaction.

### Successful Processing

```text
Delete Existing Data
        ↓
Insert Basket
        ↓
Insert Stock Records
        ↓
COMMIT
```

### Failed Processing

```text
Delete Existing Data
        ↓
Insert Basket
        ↓
Insert Stock Records
        ↓
ERROR
        ↓
ROLLBACK
```

If an error occurs, the transaction is rolled back so that the changes are not committed partially. 


---

## 23. Running the Utility

Place the following files in the required directory:

```text
DumpBasketData.py
basket_dump.json
instruments.csv
```

Run the utility using:

```bash
python DumpBasketData.py
```

or:

```bash
python3 DumpBasketData.py
```

The utility automatically reads:

```text
basket_dump.json
```

and:

```text
instruments.csv
```

---

## 24. Execution Sequence

During execution, the utility performs:

```text
1. Load basket JSON
2. Read basket information
3. Read stock information
4. Validate FundName and FundOwner
5. Validate FundVolatility
6. Validate total weightage
7. Load NSE instrument master
8. Validate stock ISINs
9. Attach InstrumentId
10. Connect to MySQL
11. Start transaction
12. Delete existing basket data
13. Insert basket
14. Insert stock weightages
15. Commit transaction
```

---

## 25. Successful Execution

After successful processing, the utility displays a success message.

Example:

```text
============================================================
✅ SUCCESS! Basket 'testing_basket' imported successfully!
   • 14 stocks inserted
============================================================
```

The displayed stock count represents the number of valid stocks inserted into the database. 


---

## 26. Database Verification

After successful execution, verify the basket record.

```sql
SELECT *
FROM BasketTradings
WHERE FundName = 'testing_basket'
  AND FundOwner = 'Priyanshu';
```

Verify the associated stock records:

```sql
SELECT *
FROM BasketStockWeightages
WHERE FundName = 'testing_basket'
  AND FundOwner = 'Priyanshu';
```

---

## 27. Verify Total Weightage

Use the following query:

```sql
SELECT SUM(Weightage) AS TotalWeightage
FROM BasketStockWeightages
WHERE FundName = 'testing_basket'
  AND FundOwner = 'Priyanshu';
```

Expected result:

```text
TotalWeightage
--------------
100
```

---

## 28. Verify Instrument IDs

Check that every imported stock has an `InstrumentId`:

```sql
SELECT
    StockSymbol,
    ISIN,
    InstrumentId,
    Weightage
FROM BasketStockWeightages
WHERE FundName = 'testing_basket'
  AND FundOwner = 'Priyanshu';
```

---

## 29. Troubleshooting

### 29.1 Basket JSON Not Found

Check whether the file exists:

```bash
ls -lh basket_dump.json
```

Ensure the file is available in the directory from which the utility is executed.

---

### 29.2 Instrument File Not Found

Check:

```bash
ls -lh instruments.csv
```

Ensure the instrument master is available.

---

### 29.3 Invalid Fund Volatility

Use one of the supported values:

```text
High
Medium
Low
```

---

### 29.4 Invalid Weightage

If the total stock weightage is not 100, review the `Weightage` values in the JSON.

---

### 29.5 Missing ISIN

Check the affected stock in `basket_dump.json`.

Every stock intended for import should contain a valid ISIN.

---

### 29.6 Invalid ISIN

Verify that the ISIN exists in the NSE section of:

```text
instruments.csv
```

---

### 29.7 No Valid Stocks

If no valid stocks remain after ISIN validation, the database insertion is aborted.

The utility displays:

```text
❌ No valid stocks found. Aborting DB insert.
```



---

### 29.8 Database Connection Error

Check:

- Database host
- Database port
- Database username
- Database password
- Database name
- MySQL server status
- Network connectivity
- Database permissions

---

### 29.9 Transaction Failure

If an error occurs during database processing, the transaction is rolled back.

Review the displayed error and correct the input data or database configuration before retrying.

---

## 30. Operational Checklist

### Before Import

- [ ] `DumpBasketData.py` is available.
- [ ] `basket_dump.json` is available.
- [ ] `instruments.csv` is available.
- [ ] JSON structure is valid.
- [ ] `FundName` is provided.
- [ ] `FundOwner` is provided.
- [ ] `FundVolatility` is valid.
- [ ] Total stock weightage is 100.
- [ ] Required ISINs exist in the NSE instrument master.
- [ ] MySQL is accessible.
- [ ] Required database tables exist.

### During Import

- [ ] Basket validation passes.
- [ ] NSE instrument mapping loads successfully.
- [ ] Stock ISIN validation completes.
- [ ] `InstrumentId` values are assigned.
- [ ] Existing basket data is removed.
- [ ] Basket record is inserted.
- [ ] Stock records are inserted.
- [ ] Transaction is committed.

### After Import

- [ ] Success message is displayed.
- [ ] Basket record exists in `BasketTradings`.
- [ ] Stock records exist in `BasketStockWeightages`.
- [ ] Number of inserted stocks is correct.
- [ ] Total weightage equals 100.
- [ ] `InstrumentId` values are present and correct.