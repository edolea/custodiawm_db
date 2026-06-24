
```sql
CREATE TABLE provider (
    provider_id INTEGER PRIMARY KEY,
    name TEXT NOT NULL UNIQUE
);

CREATE TABLE file_load (
    file_load_id INTEGER PRIMARY KEY,
    provider_id INTEGER NOT NULL,
    file_name TEXT NOT NULL,
    file_type TEXT NOT NULL,
    loaded_at TIMESTAMP NOT NULL,
    status TEXT NOT NULL,
    row_count INTEGER,
    checksum TEXT
);

CREATE TABLE portfolio (
    portfolio_id INTEGER PRIMARY KEY,
    provider_id INTEGER NOT NULL,
    source_portfolio_code TEXT NOT NULL,
    name TEXT,
    base_currency TEXT,
    file_load_id INTEGER NOT NULL
);

CREATE TABLE instrument (
    instrument_id INTEGER PRIMARY KEY,
    primary_isin TEXT,
    name TEXT,
    currency TEXT,
    issuer_name TEXT,
    asset_class TEXT,
    file_load_id INTEGER NOT NULL
);

CREATE TABLE instrument_identifier (
    instrument_identifier_id INTEGER PRIMARY KEY,
    instrument_id INTEGER NOT NULL,
    provider_id INTEGER NOT NULL,
    identifier_type TEXT NOT NULL,
    identifier_value TEXT NOT NULL,
    is_primary BOOLEAN DEFAULT FALSE
);

CREATE TABLE custody_account (
    custody_account_id INTEGER PRIMARY KEY,
    provider_id INTEGER NOT NULL,
    portfolio_id INTEGER,
    source_account_code TEXT NOT NULL,
    name TEXT,
    file_load_id INTEGER NOT NULL
);

CREATE TABLE cash_account (
    cash_account_id INTEGER PRIMARY KEY,
    provider_id INTEGER NOT NULL,
    portfolio_id INTEGER,
    source_account_code TEXT,
    iban TEXT,
    currency TEXT,
    file_load_id INTEGER NOT NULL
);

CREATE TABLE transaction_header (
    transaction_id INTEGER PRIMARY KEY,
    provider_id INTEGER NOT NULL,
    portfolio_id INTEGER,
    source_reference TEXT NOT NULL,
    transaction_type TEXT,
    trade_date DATE,
    value_date DATE,
    status TEXT,
    file_load_id INTEGER NOT NULL
);

CREATE TABLE movement (
    movement_id INTEGER PRIMARY KEY,
    transaction_id INTEGER NOT NULL,
    line_number INTEGER,
    instrument_id INTEGER,
    custody_account_id INTEGER,
    cash_account_id INTEGER,
    quantity NUMERIC,
    amount NUMERIC,
    currency TEXT,
    price NUMERIC
);

CREATE TABLE position_snapshot (
    position_id INTEGER PRIMARY KEY,
    provider_id INTEGER NOT NULL,
    portfolio_id INTEGER,
    custody_account_id INTEGER,
    instrument_id INTEGER,
    position_date DATE NOT NULL,
    quantity NUMERIC,
    market_value NUMERIC,
    market_value_currency TEXT,
    accrued_interest NUMERIC,
    book_value NUMERIC,
    file_load_id INTEGER NOT NULL
);
```