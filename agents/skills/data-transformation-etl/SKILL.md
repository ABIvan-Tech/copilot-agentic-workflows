# Data Transformation & ETL

## Overview
Data Transformation and ETL (Extract, Transform, Load) processes form the backbone of modern data engineering. This skill covers best practices for parsing, validating, transforming, and loading data across various formats and systems. Whether you're building batch pipelines, real-time streams, or data integration workflows, proper ETL design ensures data quality, reliability, and maintainability.

This guide focuses on practical patterns in Python and TypeScript/Node.js, covering everything from basic CSV parsing to complex pipeline orchestration, with emphasis on error handling, performance optimization, and production-ready code.

## When to Use This Skill
- Building data pipelines that extract from multiple sources
- Transforming raw data into structured, validated formats
- Integrating third-party data feeds (APIs, files, databases)
- Cleaning and normalizing messy or inconsistent data
- Implementing incremental data synchronization
- Processing large datasets with memory constraints
- Validating data schemas and enforcing data quality
- Creating audit trails and maintaining data lineage
- Handling errors and retries in data workflows
- Optimizing pipeline performance and throughput

## Data Parsing & Validation

### CSV/Excel Parsing with Pandas (Python)

```python
# ✅ Good - Robust CSV parsing with validation
import pandas as pd
from typing import Optional
import logging

def parse_csv_safely(
    file_path: str,
    expected_columns: list[str],
    date_columns: Optional[list[str]] = None
) -> pd.DataFrame:
    """
    Parse CSV with validation and error handling.
    """
    try:
        df = pd.read_csv(
            file_path,
            encoding='utf-8-sig',  # Handle BOM
            dtype=str,  # Read everything as string initially
            na_values=['', 'NULL', 'null', 'N/A'],
            parse_dates=date_columns or [],
            date_format='ISO8601',
            on_bad_lines='skip'  # Skip malformed rows
        )
        
        # Validate expected columns
        missing_cols = set(expected_columns) - set(df.columns)
        if missing_cols:
            raise ValueError(f"Missing required columns: {missing_cols}")
        
        # Strip whitespace from string columns
        str_cols = df.select_dtypes(include=['object']).columns
        df[str_cols] = df[str_cols].apply(lambda x: x.str.strip())
        
        logging.info(f"Parsed {len(df)} rows from {file_path}")
        return df[expected_columns]  # Return only expected columns
        
    except pd.errors.EmptyDataError:
        logging.warning(f"Empty file: {file_path}")
        return pd.DataFrame(columns=expected_columns)
    except Exception as e:
        logging.error(f"Failed to parse {file_path}: {e}")
        raise

# ❌ Bad - Fragile parsing without error handling
def parse_csv_bad(file_path: str) -> pd.DataFrame:
    df = pd.read_csv(file_path)  # No encoding, no validation
    return df  # Returns whatever is in the file

# Usage
df = parse_csv_safely(
    'data/sales.csv',
    expected_columns=['order_id', 'customer_id', 'amount', 'date'],
    date_columns=['date']
)
```

### JSON Parsing with Validation (Python)

```python
# ✅ Good - Type-safe JSON parsing with Pydantic
from pydantic import BaseModel, Field, field_validator, ValidationError
from datetime import datetime
from typing import Optional
import json

class UserRecord(BaseModel):
    """Validated user record schema."""
    user_id: int = Field(gt=0)
    email: str
    name: str
    signup_date: datetime
    age: Optional[int] = Field(None, ge=0, le=150)
    metadata: dict = Field(default_factory=dict)
    
    @field_validator('email')
    @classmethod
    def validate_email(cls, v: str) -> str:
        if '@' not in v or '.' not in v.split('@')[1]:
            raise ValueError('Invalid email format')
        return v.lower()
    
    @field_validator('name')
    @classmethod
    def validate_name(cls, v: str) -> str:
        cleaned = v.strip()
        if len(cleaned) < 2:
            raise ValueError('Name too short')
        return cleaned

def parse_json_records(json_data: str) -> list[UserRecord]:
    """Parse and validate JSON records."""
    try:
        raw_data = json.loads(json_data)
        records = []
        errors = []
        
        for idx, item in enumerate(raw_data):
            try:
                record = UserRecord(**item)
                records.append(record)
            except ValidationError as e:
                errors.append(f"Row {idx}: {e}")
        
        if errors and len(errors) > len(raw_data) * 0.5:
            # More than 50% failed - abort
            raise ValueError(f"Too many validation errors: {len(errors)}")
        
        if errors:
            logging.warning(f"Skipped {len(errors)} invalid records")
        
        return records
    except json.JSONDecodeError as e:
        raise ValueError(f"Invalid JSON: {e}")

# ❌ Bad - No validation or error handling
def parse_json_bad(json_data: str):
    return json.loads(json_data)  # Hope for the best
```

### CSV Parsing in Node.js/TypeScript

```typescript
// ✅ Good - Streaming CSV parser with validation
import { createReadStream } from 'fs';
import { parse } from 'csv-parse';
import { z } from 'zod';

// Define schema with Zod
const SalesRecordSchema = z.object({
  order_id: z.string().min(1),
  customer_id: z.string().uuid(),
  amount: z.coerce.number().positive(),
  date: z.coerce.date(),
  status: z.enum(['pending', 'completed', 'cancelled'])
});

type SalesRecord = z.infer<typeof SalesRecordSchema>;

async function parseCsvStream(
  filePath: string
): Promise<{ valid: SalesRecord[]; errors: Array<{ row: number; error: string }> }> {
  const valid: SalesRecord[] = [];
  const errors: Array<{ row: number; error: string }> = [];
  let rowNum = 0;

  const parser = createReadStream(filePath)
    .pipe(parse({
      columns: true,
      skip_empty_lines: true,
      trim: true,
      cast: false  // We'll handle type casting with Zod
    }));

  for await (const record of parser) {
    rowNum++;
    try {
      const validated = SalesRecordSchema.parse(record);
      valid.push(validated);
    } catch (error) {
      if (error instanceof z.ZodError) {
        errors.push({
          row: rowNum,
          error: error.errors.map(e => `${e.path}: ${e.message}`).join(', ')
        });
      }
    }
  }

  console.log(`Parsed ${valid.length} valid records, ${errors.length} errors`);
  return { valid, errors };
}

// ❌ Bad - Loading entire file into memory
async function parseCsvBad(filePath: string): Promise<any[]> {
  const fs = require('fs');
  const content = fs.readFileSync(filePath, 'utf-8');  // Memory risk!
  const lines = content.split('\n');
  return lines.map(line => line.split(','));  // No validation, fragile parsing
}
```

### XML Parsing (Node.js)

```typescript
// ✅ Good - Streaming XML parser with validation
import { createReadStream } from 'fs';
import { XMLParser } from 'fast-xml-parser';
import { z } from 'zod';

const ProductSchema = z.object({
  id: z.string(),
  name: z.string().min(1),
  price: z.number().positive(),
  inventory: z.object({
    quantity: z.number().int().nonnegative(),
    warehouse: z.string()
  })
});

type Product = z.infer<typeof ProductSchema>;

async function parseXmlFeed(xmlPath: string): Promise<Product[]> {
  const parser = new XMLParser({
    ignoreAttributes: false,
    attributeNamePrefix: '@_',
    parseAttributeValue: true,
    trimValues: true
  });

  const xmlContent = await fs.promises.readFile(xmlPath, 'utf-8');
  const parsed = parser.parse(xmlContent);
  
  const products: Product[] = [];
  const items = parsed.catalog?.product || [];
  const itemsArray = Array.isArray(items) ? items : [items];

  for (const item of itemsArray) {
    try {
      const validated = ProductSchema.parse(item);
      products.push(validated);
    } catch (error) {
      console.warn(`Skipping invalid product: ${item.id}`, error);
    }
  }

  return products;
}

// ❌ Bad - No validation or structure checking
function parseXmlBad(xmlPath: string): any {
  const parser = new XMLParser();
  const content = fs.readFileSync(xmlPath, 'utf-8');
  return parser.parse(content);  // Return raw, unvalidated data
}
```

### Data Type Coercion & Normalization

```python
# ✅ Good - Safe type coercion with fallbacks
import pandas as pd
import numpy as np
from decimal import Decimal
from datetime import datetime

class DataNormalizer:
    """Safely normalize and coerce data types."""
    
    @staticmethod
    def normalize_numeric(series: pd.Series, default=0.0) -> pd.Series:
        """Convert to numeric with safe fallback."""
        # Remove currency symbols and commas
        if series.dtype == object:
            series = series.str.replace(r'[$,]', '', regex=True)
        
        # Coerce to numeric, invalid becomes NaN
        numeric = pd.to_numeric(series, errors='coerce')
        
        # Fill NaN with default
        return numeric.fillna(default)
    
    @staticmethod
    def normalize_dates(series: pd.Series) -> pd.Series:
        """Parse dates flexibly with multiple formats."""
        formats = [
            '%Y-%m-%d',
            '%m/%d/%Y',
            '%d-%m-%Y',
            '%Y%m%d'
        ]
        
        result = pd.Series(index=series.index, dtype='datetime64[ns]')
        
        for fmt in formats:
            mask = result.isna()
            result[mask] = pd.to_datetime(
                series[mask],
                format=fmt,
                errors='coerce'
            )
        
        return result
    
    @staticmethod
    def normalize_strings(
        series: pd.Series,
        lowercase: bool = True,
        strip: bool = True
    ) -> pd.Series:
        """Normalize string values."""
        result = series.fillna('').astype(str)
        
        if strip:
            result = result.str.strip()
        if lowercase:
            result = result.str.lower()
        
        # Replace multiple spaces with single space
        result = result.str.replace(r'\s+', ' ', regex=True)
        
        return result

# ❌ Bad - Aggressive coercion without error handling
def normalize_bad(df: pd.DataFrame) -> pd.DataFrame:
    for col in df.columns:
        try:
            df[col] = pd.to_numeric(df[col])  # Forces everything to numeric
        except:
            pass  # Silent failures
    return df
```

## Data Transformation Patterns

### Filtering & Mapping Operations

```python
# ✅ Good - Readable, type-safe transformations
from dataclasses import dataclass
from typing import List, Callable
import pandas as pd

@dataclass
class OrderRecord:
    order_id: str
    customer_id: str
    amount: float
    status: str
    created_at: datetime

class OrderTransformer:
    """Type-safe order data transformations."""
    
    @staticmethod
    def filter_valid_orders(orders: List[OrderRecord]) -> List[OrderRecord]:
        """Filter for valid, completed orders."""
        return [
            order for order in orders
            if order.amount > 0
            and order.status == 'completed'
            and order.customer_id.strip()
        ]
    
    @staticmethod
    def enrich_with_revenue_tier(orders: List[OrderRecord]) -> List[dict]:
        """Add revenue tier classification."""
        def get_tier(amount: float) -> str:
            if amount >= 1000:
                return 'premium'
            elif amount >= 100:
                return 'standard'
            return 'basic'
        
        return [
            {
                **order.__dict__,
                'revenue_tier': get_tier(order.amount),
                'is_high_value': order.amount >= 500
            }
            for order in orders
        ]
    
    @staticmethod
    def aggregate_by_customer(df: pd.DataFrame) -> pd.DataFrame:
        """Aggregate orders by customer with metrics."""
        return df.groupby('customer_id').agg({
            'order_id': 'count',
            'amount': ['sum', 'mean', 'max'],
            'created_at': ['min', 'max']
        }).rename(columns={
            'order_id': 'total_orders',
            'amount': 'revenue'
        })

# ❌ Bad - Fragile, hard-to-maintain transformations
def transform_bad(data):
    result = []
    for item in data:
        if item[2] > 0:  # Magic index - what is item[2]?
            result.append({
                'x': item[0],
                'y': item[1],
                'z': item[2] * 1.1  # Magic multiplier
            })
    return result
```

### Data Cleaning (Python)

```python
# ✅ Good - Comprehensive data cleaning pipeline
import pandas as pd
import numpy as np
from typing import Optional

class DataCleaner:
    """Production-grade data cleaning utilities."""
    
    @staticmethod
    def remove_duplicates(
        df: pd.DataFrame,
        subset: Optional[list] = None,
        keep: str = 'first'
    ) -> pd.DataFrame:
        """Remove duplicates with logging."""
        initial_count = len(df)
        df_clean = df.drop_duplicates(subset=subset, keep=keep)
        removed = initial_count - len(df_clean)
        
        if removed > 0:
            logging.info(f"Removed {removed} duplicate rows ({removed/initial_count*100:.1f}%)")
        
        return df_clean
    
    @staticmethod
    def handle_outliers(
        df: pd.DataFrame,
        column: str,
        method: str = 'iqr',
        threshold: float = 1.5
    ) -> pd.DataFrame:
        """Remove or cap outliers using IQR or z-score."""
        if method == 'iqr':
            Q1 = df[column].quantile(0.25)
            Q3 = df[column].quantile(0.75)
            IQR = Q3 - Q1
            
            lower_bound = Q1 - threshold * IQR
            upper_bound = Q3 + threshold * IQR
            
            # Cap outliers instead of removing
            df[column] = df[column].clip(lower=lower_bound, upper=upper_bound)
            
        elif method == 'zscore':
            z_scores = np.abs((df[column] - df[column].mean()) / df[column].std())
            df = df[z_scores < threshold]
        
        return df
    
    @staticmethod
    def handle_missing_values(
        df: pd.DataFrame,
        strategy: dict[str, str]
    ) -> pd.DataFrame:
        """
        Handle missing values with column-specific strategies.
        
        strategy = {
            'age': 'median',
            'name': 'drop',
            'country': 'mode',
            'description': 'constant:Unknown'
        }
        """
        df_clean = df.copy()
        
        for column, method in strategy.items():
            if column not in df_clean.columns:
                continue
            
            if method == 'drop':
                df_clean = df_clean.dropna(subset=[column])
            elif method == 'mean':
                df_clean[column].fillna(df_clean[column].mean(), inplace=True)
            elif method == 'median':
                df_clean[column].fillna(df_clean[column].median(), inplace=True)
            elif method == 'mode':
                df_clean[column].fillna(df_clean[column].mode()[0], inplace=True)
            elif method.startswith('constant:'):
                value = method.split(':', 1)[1]
                df_clean[column].fillna(value, inplace=True)
        
        return df_clean

# ❌ Bad - Destructive cleaning without strategy
def clean_bad(df):
    df = df.dropna()  # Drops any row with any null
    df = df.drop_duplicates()  # No subset specified
    return df  # May have removed 90% of data!
```

### Type-Safe Transformations (TypeScript)

```typescript
// ✅ Good - Type-safe transformation pipeline
import { z } from 'zod';

interface RawOrderData {
  orderId: string;
  customerId: string;
  items: string;  // JSON string
  total: string;  // String number
  date: string;
}

interface ProcessedOrder {
  orderId: string;
  customerId: string;
  items: OrderItem[];
  total: number;
  date: Date;
  itemCount: number;
  averageItemPrice: number;
}

interface OrderItem {
  sku: string;
  quantity: number;
  price: number;
}

class OrderTransformPipeline {
  /**
   * Transform raw order data with validation at each step.
   */
  static transform(rawOrders: RawOrderData[]): ProcessedOrder[] {
    return rawOrders
      .map(order => this.parseOrder(order))
      .filter((order): order is ProcessedOrder => order !== null)
      .map(order => this.enrichOrder(order));
  }

  private static parseOrder(raw: RawOrderData): ProcessedOrder | null {
    try {
      // Parse items JSON
      const items: OrderItem[] = JSON.parse(raw.items);
      
      // Validate item structure
      const itemSchema = z.array(z.object({
        sku: z.string(),
        quantity: z.number().int().positive(),
        price: z.number().positive()
      }));
      
      const validatedItems = itemSchema.parse(items);
      
      return {
        orderId: raw.orderId,
        customerId: raw.customerId,
        items: validatedItems,
        total: parseFloat(raw.total),
        date: new Date(raw.date),
        itemCount: validatedItems.length,
        averageItemPrice: this.calculateAveragePrice(validatedItems)
      };
    } catch (error) {
      console.warn(`Failed to parse order ${raw.orderId}:`, error);
      return null;
    }
  }

  private static enrichOrder(order: ProcessedOrder): ProcessedOrder {
    // Add computed fields
    return {
      ...order,
      itemCount: order.items.length,
      averageItemPrice: this.calculateAveragePrice(order.items)
    };
  }

  private static calculateAveragePrice(items: OrderItem[]): number {
    if (items.length === 0) return 0;
    const totalPrice = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
    const totalQuantity = items.reduce((sum, item) => sum + item.quantity, 0);
    return totalPrice / totalQuantity;
  }
}

// ❌ Bad - No type safety, error-prone
function transformBad(orders: any[]): any[] {
  return orders.map(o => ({
    ...o,
    items: JSON.parse(o.items),  // Can throw
    total: parseFloat(o.total),   // Can be NaN
    date: new Date(o.date)        // Can be Invalid Date
  }));  // No validation or error handling
}
```

### Stream Processing (Node.js)

```typescript
// ✅ Good - Memory-efficient stream processing
import { Transform, pipeline } from 'stream';
import { createReadStream, createWriteStream } from 'fs';
import { parse } from 'csv-parse';
import { stringify } from 'csv-stringify';
import { promisify } from 'util';

const pipelineAsync = promisify(pipeline);

class DataTransformStream extends Transform {
  private rowCount = 0;
  private errorCount = 0;

  constructor(
    private validator: (row: any) => boolean,
    private transformer: (row: any) => any
  ) {
    super({ objectMode: true });
  }

  _transform(chunk: any, encoding: string, callback: Function) {
    this.rowCount++;
    
    try {
      if (this.validator(chunk)) {
        const transformed = this.transformer(chunk);
        this.push(transformed);
      } else {
        this.errorCount++;
      }
    } catch (error) {
      this.errorCount++;
      console.error(`Row ${this.rowCount} transformation error:`, error);
    }
    
    callback();
  }

  _flush(callback: Function) {
    console.log(`Processed ${this.rowCount} rows, ${this.errorCount} errors`);
    callback();
  }
}

async function processLargeDataset(inputPath: string, outputPath: string) {
  const validator = (row: any) => {
    return row.amount && parseFloat(row.amount) > 0;
  };

  const transformer = (row: any) => ({
    ...row,
    amount: parseFloat(row.amount),
    processed_at: new Date().toISOString()
  });

  await pipelineAsync(
    createReadStream(inputPath),
    parse({ columns: true }),
    new DataTransformStream(validator, transformer),
    stringify({ header: true }),
    createWriteStream(outputPath)
  );
}

// ❌ Bad - Loading everything into memory
async function processBad(inputPath: string, outputPath: string) {
  const fs = require('fs');
  const content = fs.readFileSync(inputPath, 'utf-8');  // OOM risk for large files
  const rows = content.split('\n').map(line => {
    // Process each row
    return line.split(',').join('|');
  });
  fs.writeFileSync(outputPath, rows.join('\n'));
}
```

## ETL Pipeline Design

### Batch Processing Pipeline

```python
# ✅ Good - Robust batch ETL pipeline
from dataclasses import dataclass
from datetime import datetime
from typing import Optional, Callable, Any
import logging
from pathlib import Path

@dataclass
class PipelineRun:
    """Track pipeline execution metadata."""
    run_id: str
    start_time: datetime
    end_time: Optional[datetime] = None
    status: str = 'running'
    records_processed: int = 0
    records_failed: int = 0
    error_message: Optional[str] = None

class ETLPipeline:
    """Production-grade ETL pipeline with checkpointing."""
    
    def __init__(self, name: str, checkpoint_dir: Path):
        self.name = name
        self.checkpoint_dir = checkpoint_dir
        self.checkpoint_dir.mkdir(parents=True, exist_ok=True)
    
    def execute(
        self,
        extract_fn: Callable,
        transform_fn: Callable,
        load_fn: Callable,
        run_id: Optional[str] = None
    ) -> PipelineRun:
        """Execute ETL pipeline with error handling and checkpointing."""
        run_id = run_id or f"{self.name}_{datetime.now().isoformat()}"
        run = PipelineRun(run_id=run_id, start_time=datetime.now())
        
        try:
            # Check if we can resume from checkpoint
            checkpoint = self._load_checkpoint(run_id)
            if checkpoint:
                logging.info(f"Resuming from checkpoint: {checkpoint}")
                data = checkpoint['data']
            else:
                # Extract phase
                logging.info(f"Extracting data for {run_id}")
                data = extract_fn()
                self._save_checkpoint(run_id, 'extracted', data)
            
            # Transform phase
            logging.info(f"Transforming data for {run_id}")
            transformed, errors = transform_fn(data)
            run.records_processed = len(transformed)
            run.records_failed = len(errors)
            
            if errors:
                self._log_errors(run_id, errors)
            
            self._save_checkpoint(run_id, 'transformed', transformed)
            
            # Load phase
            logging.info(f"Loading data for {run_id}")
            load_fn(transformed)
            
            # Success
            run.status = 'completed'
            run.end_time = datetime.now()
            self._cleanup_checkpoint(run_id)
            
        except Exception as e:
            run.status = 'failed'
            run.error_message = str(e)
            run.end_time = datetime.now()
            logging.error(f"Pipeline failed: {e}", exc_info=True)
            raise
        finally:
            self._save_run_metadata(run)
        
        return run
    
    def _save_checkpoint(self, run_id: str, stage: str, data: Any):
        """Save pipeline checkpoint for resumability."""
        checkpoint_file = self.checkpoint_dir / f"{run_id}_{stage}.checkpoint"
        import pickle
        with open(checkpoint_file, 'wb') as f:
            pickle.dump({'stage': stage, 'data': data}, f)
    
    def _load_checkpoint(self, run_id: str) -> Optional[dict]:
        """Load last checkpoint if exists."""
        checkpoints = list(self.checkpoint_dir.glob(f"{run_id}_*.checkpoint"))
        if checkpoints:
            latest = max(checkpoints, key=lambda p: p.stat().st_mtime)
            import pickle
            with open(latest, 'rb') as f:
                return pickle.load(f)
        return None
    
    def _cleanup_checkpoint(self, run_id: str):
        """Remove checkpoints after successful completion."""
        for checkpoint in self.checkpoint_dir.glob(f"{run_id}_*.checkpoint"):
            checkpoint.unlink()

# ❌ Bad - No error handling, checkpointing, or logging
def etl_bad(extract_fn, transform_fn, load_fn):
    data = extract_fn()  # If this fails, lose everything
    transformed = transform_fn(data)  # No error tracking
    load_fn(transformed)  # No way to resume if this fails
```

### Incremental Loading Strategy

```python
# ✅ Good - Efficient incremental loads with watermarks
from datetime import datetime, timedelta
from typing import Optional
import sqlite3

class IncrementalLoader:
    """Load only new/changed data using watermarks."""
    
    def __init__(self, state_db: str):
        self.state_db = state_db
        self._init_state_table()
    
    def _init_state_table(self):
        """Initialize state tracking table."""
        with sqlite3.connect(self.state_db) as conn:
            conn.execute("""
                CREATE TABLE IF NOT EXISTS watermarks (
                    source TEXT PRIMARY KEY,
                    last_synced_at TIMESTAMP,
                    last_record_id TEXT,
                    records_processed INTEGER
                )
            """)
    
    def get_watermark(self, source: str) -> Optional[datetime]:
        """Get last sync timestamp for a source."""
        with sqlite3.connect(self.state_db) as conn:
            result = conn.execute(
                "SELECT last_synced_at FROM watermarks WHERE source = ?",
                (source,)
            ).fetchone()
            
            return datetime.fromisoformat(result[0]) if result else None
    
    def update_watermark(
        self,
        source: str,
        timestamp: datetime,
        record_id: Optional[str] = None,
        count: int = 0
    ):
        """Update watermark after successful load."""
        with sqlite3.connect(self.state_db) as conn:
            conn.execute("""
                INSERT OR REPLACE INTO watermarks
                (source, last_synced_at, last_record_id, records_processed)
                VALUES (?, ?, ?, ?)
            """, (source, timestamp.isoformat(), record_id, count))
            conn.commit()
    
    def load_incremental(
        self,
        source: str,
        fetch_fn: Callable[[Optional[datetime]], list],
        load_fn: Callable[[list], None]
    ):
        """Load only records since last watermark."""
        last_sync = self.get_watermark(source)
        
        if last_sync:
            logging.info(f"Loading {source} since {last_sync}")
        else:
            logging.info(f"Initial load for {source}")
        
        # Fetch new records
        new_records = fetch_fn(last_sync)
        
        if not new_records:
            logging.info(f"No new records for {source}")
            return
        
        # Load to destination
        load_fn(new_records)
        
        # Update watermark
        max_timestamp = max(r['updated_at'] for r in new_records)
        self.update_watermark(source, max_timestamp, count=len(new_records))
        
        logging.info(f"Loaded {len(new_records)} new records for {source}")

# ❌ Bad - Full load every time
def load_all_data_bad(source: str):
    data = fetch_all_data(source)  # Fetches millions of unchanged records
    load_to_warehouse(data)  # Inefficient, slow, expensive
```

### Error Handling & Retry Logic

```typescript
// ✅ Good - Comprehensive error handling with exponential backoff
interface RetryConfig {
  maxRetries: number;
  initialDelayMs: number;
  maxDelayMs: number;
  backoffMultiplier: number;
}

class RetryableETL {
  private defaultConfig: RetryConfig = {
    maxRetries: 3,
    initialDelayMs: 1000,
    maxDelayMs: 30000,
    backoffMultiplier: 2
  };

  async executeWithRetry<T>(
    operation: () => Promise<T>,
    operationName: string,
    config: Partial<RetryConfig> = {}
  ): Promise<T> {
    const cfg = { ...this.defaultConfig, ...config };
    let lastError: Error | undefined;
    
    for (let attempt = 0; attempt <= cfg.maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error as Error;
        
        if (attempt === cfg.maxRetries) {
          console.error(`${operationName} failed after ${cfg.maxRetries} retries`);
          break;
        }
        
        // Check if error is retryable
        if (!this.isRetryable(error)) {
          console.error(`${operationName} failed with non-retryable error`);
          throw error;
        }
        
        // Calculate delay with exponential backoff
        const delay = Math.min(
          cfg.initialDelayMs * Math.pow(cfg.backoffMultiplier, attempt),
          cfg.maxDelayMs
        );
        
        console.warn(
          `${operationName} failed (attempt ${attempt + 1}/${cfg.maxRetries}), ` +
          `retrying in ${delay}ms...`,
          error
        );
        
        await this.sleep(delay);
      }
    }
    
    throw lastError!;
  }

  private isRetryable(error: any): boolean {
    // Network errors, timeouts, rate limits are retryable
    const retryablePatterns = [
      /ETIMEDOUT/,
      /ECONNREFUSED/,
      /ENOTFOUND/,
      /429/,  // Too Many Requests
      /502/,  // Bad Gateway
      /503/,  // Service Unavailable
      /504/   // Gateway Timeout
    ];
    
    const errorString = error.toString();
    return retryablePatterns.some(pattern => pattern.test(errorString));
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Usage
const etl = new RetryableETL();

async function fetchApiData(url: string) {
  return etl.executeWithRetry(
    async () => {
      const response = await fetch(url);
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      return response.json();
    },
    'API Fetch',
    { maxRetries: 5 }
  );
}

// ❌ Bad - No retry logic
async function fetchBad(url: string) {
  const response = await fetch(url);  // Fails on network blip
  return response.json();
}
```

### Idempotency in ETL

```python
# ✅ Good - Idempotent upsert operations
import hashlib
import json
from typing import Any, Dict

class IdempotentLoader:
    """Ensure ETL operations can be safely retried."""
    
    @staticmethod
    def generate_record_hash(record: Dict[str, Any]) -> str:
        """Generate consistent hash for record deduplication."""
        # Sort keys for consistent hashing
        normalized = json.dumps(record, sort_keys=True)
        return hashlib.sha256(normalized.encode()).hexdigest()
    
    def upsert_records(self, records: list[Dict], table: str, key_columns: list[str]):
        """
        Idempotent upsert - can run multiple times safely.
        Uses composite key for deduplication.
        """
        # Example with SQLite (adapt for your DB)
        import sqlite3
        
        placeholders = ', '.join(['?'] * len(records[0]))
        columns = list(records[0].keys())
        
        # Build upsert query
        update_clause = ', '.join([
            f"{col} = excluded.{col}"
            for col in columns if col not in key_columns
        ])
        
        query = f"""
            INSERT INTO {table} ({', '.join(columns)})
            VALUES ({placeholders})
            ON CONFLICT({', '.join(key_columns)})
            DO UPDATE SET {update_clause}
        """
        
        with sqlite3.connect('database.db') as conn:
            for record in records:
                values = [record[col] for col in columns]
                conn.execute(query, values)
            conn.commit()
    
    def load_with_deduplication(
        self,
        records: list[Dict],
        existing_hashes: set[str]
    ) -> tuple[list[Dict], list[Dict]]:
        """
        Separate new records from duplicates.
        Returns: (new_records, duplicate_records)
        """
        new_records = []
        duplicates = []
        
        for record in records:
            record_hash = self.generate_record_hash(record)
            
            if record_hash in existing_hashes:
                duplicates.append(record)
            else:
                new_records.append(record)
                existing_hashes.add(record_hash)
        
        return new_records, duplicates

# ❌ Bad - Non-idempotent inserts
def load_bad(records: list):
    for record in records:
        # If this runs twice, creates duplicates
        db.execute(f"INSERT INTO table VALUES (?)", record)
```

## Performance & Optimization

### Memory Management for Large Datasets

```python
# ✅ Good - Chunked processing for memory efficiency
import pandas as pd
from typing import Iterator, Callable

class ChunkedProcessor:
    """Process large files in memory-efficient chunks."""
    
    def __init__(self, chunk_size: int = 10000):
        self.chunk_size = chunk_size
    
    def process_csv_chunks(
        self,
        file_path: str,
        transform_fn: Callable[[pd.DataFrame], pd.DataFrame],
        output_path: str
    ):
        """Process CSV file in chunks to manage memory."""
        first_chunk = True
        total_processed = 0
        
        for chunk in pd.read_csv(file_path, chunksize=self.chunk_size):
            # Transform chunk
            transformed = transform_fn(chunk)
            
            # Write to output (append mode after first chunk)
            mode = 'w' if first_chunk else 'a'
            header = first_chunk
            transformed.to_csv(
                output_path,
                mode=mode,
                header=header,
                index=False
            )
            
            total_processed += len(transformed)
            first_chunk = False
            
            logging.info(f"Processed {total_processed} records")
    
    def aggregate_chunks(
        self,
        file_path: str,
        group_by: str,
        agg_funcs: dict
    ) -> pd.DataFrame:
        """Perform aggregations on chunks, then combine."""
        chunk_results = []
        
        for chunk in pd.read_csv(file_path, chunksize=self.chunk_size):
            chunk_agg = chunk.groupby(group_by).agg(agg_funcs)
            chunk_results.append(chunk_agg)
        
        # Combine chunk results
        combined = pd.concat(chunk_results)
        
        # Re-aggregate the combined results
        final = combined.groupby(level=0).agg(agg_funcs)
        
        return final

# ❌ Bad - Loading entire file into memory
def process_bad(file_path: str):
    df = pd.read_csv(file_path)  # May be GBs of data!
    return df.groupby('category').sum()  # OOM error
```

### Parallel Processing (Python)

```python
# ✅ Good - Parallel processing with multiprocessing
from multiprocessing import Pool, cpu_count
from functools import partial
import pandas as pd
from typing import List, Callable

class ParallelProcessor:
    """Process data in parallel across CPU cores."""
    
    def __init__(self, n_workers: int = None):
        self.n_workers = n_workers or cpu_count()
    
    def process_dataframe_parallel(
        self,
        df: pd.DataFrame,
        transform_fn: Callable[[pd.DataFrame], pd.DataFrame],
        n_splits: int = None
    ) -> pd.DataFrame:
        """
        Split DataFrame and process chunks in parallel.
        Useful for CPU-bound transformations.
        """
        n_splits = n_splits or self.n_workers
        
        # Split DataFrame into chunks
        chunk_size = len(df) // n_splits
        chunks = [
            df.iloc[i:i + chunk_size]
            for i in range(0, len(df), chunk_size)
        ]
        
        # Process in parallel
        with Pool(self.n_workers) as pool:
            results = pool.map(transform_fn, chunks)
        
        # Combine results
        return pd.concat(results, ignore_index=True)
    
    def process_files_parallel(
        self,
        file_paths: List[str],
        process_fn: Callable[[str], pd.DataFrame]
    ) -> pd.DataFrame:
        """Process multiple files in parallel."""
        with Pool(self.n_workers) as pool:
            results = pool.map(process_fn, file_paths)
        
        return pd.concat(results, ignore_index=True)

# Example usage
def expensive_transformation(df: pd.DataFrame) -> pd.DataFrame:
    """CPU-intensive transformation."""
    df = df.copy()
    df['complex_calc'] = df['value'].apply(lambda x: x ** 2 * 1.5)
    df['category'] = df['text'].apply(lambda x: custom_nlp_function(x))
    return df

processor = ParallelProcessor(n_workers=4)
result = processor.process_dataframe_parallel(large_df, expensive_transformation)

# ❌ Bad - Single-threaded processing
def process_sequential_bad(files: List[str]):
    results = []
    for file in files:  # Processes one at a time
        results.append(process_file(file))
    return pd.concat(results)
```

### Efficient Data Structures

```typescript
// ✅ Good - Efficient data structures for lookups
class EfficientDataJoiner {
  /**
   * Join datasets using Map for O(1) lookups instead of O(n²) nested loops.
   */
  static leftJoin<T extends Record<string, any>, U extends Record<string, any>>(
    leftData: T[],
    rightData: U[],
    leftKey: keyof T,
    rightKey: keyof U
  ): Array<T & Partial<U>> {
    // Build index for right dataset - O(m)
    const rightIndex = new Map<any, U>();
    for (const item of rightData) {
      rightIndex.set(item[rightKey], item);
    }
    
    // Join using index - O(n)
    return leftData.map(leftItem => {
      const rightItem = rightIndex.get(leftItem[leftKey]);
      return { ...leftItem, ...rightItem };
    });
  }

  /**
   * Remove duplicates efficiently using Set.
   */
  static deduplicate<T>(
    items: T[],
    keyFn: (item: T) => string
  ): T[] {
    const seen = new Set<string>();
    const result: T[] = [];
    
    for (const item of items) {
      const key = keyFn(item);
      if (!seen.has(key)) {
        seen.add(key);
        result.push(item);
      }
    }
    
    return result;
  }

  /**
   * Group by key efficiently.
   */
  static groupBy<T>(
    items: T[],
    keyFn: (item: T) => string
  ): Map<string, T[]> {
    const groups = new Map<string, T[]>();
    
    for (const item of items) {
      const key = keyFn(item);
      if (!groups.has(key)) {
        groups.set(key, []);
      }
      groups.get(key)!.push(item);
    }
    
    return groups;
  }
}

// ❌ Bad - Nested loops with O(n²) complexity
function joinBad(left: any[], right: any[]): any[] {
  return left.map(leftItem => {
    // O(n²) - loops through right array for every left item
    const match = right.find(r => r.id === leftItem.id);
    return { ...leftItem, ...match };
  });
}
```

### Monitoring and Logging

```python
# ✅ Good - Comprehensive pipeline monitoring
import logging
import time
from contextlib import contextmanager
from dataclasses import dataclass
from typing import Optional
import json

@dataclass
class PipelineMetrics:
    """Track pipeline performance metrics."""
    stage: str
    start_time: float
    end_time: Optional[float] = None
    records_in: int = 0
    records_out: int = 0
    errors: int = 0
    
    @property
    def duration_seconds(self) -> float:
        if self.end_time:
            return self.end_time - self.start_time
        return time.time() - self.start_time
    
    @property
    def throughput(self) -> float:
        """Records per second."""
        duration = self.duration_seconds
        return self.records_out / duration if duration > 0 else 0

class MonitoredPipeline:
    """ETL pipeline with built-in monitoring."""
    
    def __init__(self, name: str):
        self.name = name
        self.metrics: list[PipelineMetrics] = []
        
        # Configure structured logging
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        )
        self.logger = logging.getLogger(name)
    
    @contextmanager
    def monitor_stage(self, stage_name: str, records_in: int = 0):
        """Context manager to track stage metrics."""
        metrics = PipelineMetrics(
            stage=stage_name,
            start_time=time.time(),
            records_in=records_in
        )
        
        self.logger.info(f"Starting stage: {stage_name}")
        
        try:
            yield metrics
        except Exception as e:
            metrics.errors += 1
            self.logger.error(f"Stage {stage_name} failed: {e}", exc_info=True)
            raise
        finally:
            metrics.end_time = time.time()
            self.metrics.append(metrics)
            
            self.logger.info(
                f"Completed stage: {stage_name} | "
                f"Duration: {metrics.duration_seconds:.2f}s | "
                f"Throughput: {metrics.throughput:.0f} rec/s | "
                f"Records: {metrics.records_in} → {metrics.records_out} | "
                f"Errors: {metrics.errors}"
            )
    
    def export_metrics(self) -> dict:
        """Export metrics in structured format."""
        return {
            'pipeline': self.name,
            'total_duration': sum(m.duration_seconds for m in self.metrics),
            'stages': [
                {
                    'stage': m.stage,
                    'duration_seconds': m.duration_seconds,
                    'records_in': m.records_in,
                    'records_out': m.records_out,
                    'errors': m.errors,
                    'throughput': m.throughput
                }
                for m in self.metrics
            ]
        }

# Usage
pipeline = MonitoredPipeline('customer_etl')

with pipeline.monitor_stage('extract', records_in=0) as metrics:
    data = extract_data()
    metrics.records_out = len(data)

with pipeline.monitor_stage('transform', records_in=len(data)) as metrics:
    transformed = transform_data(data)
    metrics.records_out = len(transformed)

print(json.dumps(pipeline.export_metrics(), indent=2))

# ❌ Bad - No monitoring or observability
def pipeline_bad():
    data = extract_data()
    transformed = transform_data(data)  # No idea how long this takes
    load_data(transformed)  # No error tracking
```

## Common Pitfalls

### Memory Leaks with Large Files

```python
# ❌ Bad - Reading entire file into memory
def process_large_file_bad(file_path: str):
    with open(file_path, 'r') as f:
        content = f.read()  # Loads entire 10GB file into RAM!
    return process(content)

# ✅ Good - Process line by line
def process_large_file_good(file_path: str):
    results = []
    with open(file_path, 'r') as f:
        for line in f:  # Streams one line at a time
            result = process_line(line)
            results.append(result)
    return results
```

### Schema Drift Handling

```typescript
// ✅ Good - Graceful schema evolution handling
interface SchemaVersion {
  version: number;
  fields: Set<string>;
}

class SchemaValidator {
  private knownSchemas: Map<number, SchemaVersion> = new Map();

  registerSchema(version: number, fields: string[]) {
    this.knownSchemas.set(version, {
      version,
      fields: new Set(fields)
    });
  }

  validateOrMigrate(record: any, targetVersion: number): any {
    const recordFields = new Set(Object.keys(record));
    const targetSchema = this.knownSchemas.get(targetVersion);
    
    if (!targetSchema) {
      throw new Error(`Unknown schema version: ${targetVersion}`);
    }

    // Check for missing required fields
    const missingFields = [...targetSchema.fields].filter(
      field => !recordFields.has(field)
    );

    if (missingFields.length > 0) {
      // Apply default values for missing fields
      return this.applyDefaults(record, missingFields);
    }

    // Check for extra fields (forward compatibility)
    const extraFields = [...recordFields].filter(
      field => !targetSchema.fields.has(field)
    );

    if (extraFields.length > 0) {
      console.warn(`Record contains unknown fields: ${extraFields}`);
    }

    return record;
  }

  private applyDefaults(record: any, missingFields: string[]): any {
    const defaults: Record<string, any> = {
      created_at: new Date().toISOString(),
      status: 'active',
      metadata: {}
    };

    return {
      ...record,
      ...Object.fromEntries(
        missingFields.map(field => [field, defaults[field] ?? null])
      )
    };
  }
}

// ❌ Bad - Assumes schema never changes
function processBad(record: any) {
  return {
    id: record.id,  // Breaks if field renamed
    name: record.name  // Breaks if field removed
  };
}
```

### Transaction Boundaries

```python
# ✅ Good - Proper transaction management
import sqlite3
from contextlib import contextmanager

@contextmanager
def atomic_batch_load(db_path: str, batch_size: int = 1000):
    """
    Ensure all-or-nothing batch loading.
    """
    conn = sqlite3.connect(db_path)
    conn.execute("BEGIN TRANSACTION")
    
    try:
        yield conn
        conn.commit()
        logging.info("Transaction committed successfully")
    except Exception as e:
        conn.rollback()
        logging.error(f"Transaction rolled back: {e}")
        raise
    finally:
        conn.close()

# Usage
with atomic_batch_load('data.db') as conn:
    for batch in chunk_records(records, batch_size=1000):
        conn.executemany(
            "INSERT INTO table VALUES (?, ?, ?)",
            batch
        )
# All batches committed together or none at all

# ❌ Bad - Partial commits on failure
def load_bad(records, conn):
    for record in records:
        conn.execute("INSERT INTO table VALUES (?)", record)
        conn.commit()  # Commits each record individually!
    # If crash on record 5000, first 4999 are committed (partial state)
```

### Error Handling Anti-patterns

```typescript
// ❌ Bad - Swallowing errors silently
async function processBad(records: any[]) {
  for (const record of records) {
    try {
      await processRecord(record);
    } catch (error) {
      // Silent failure - no logging, no retry, no tracking
    }
  }
}

// ✅ Good - Explicit error handling with tracking
async function processGood(records: any[]): Promise<{
  successful: any[];
  failed: Array<{ record: any; error: string }>;
}> {
  const successful: any[] = [];
  const failed: Array<{ record: any; error: string }> = [];

  for (const record of records) {
    try {
      const result = await processRecord(record);
      successful.push(result);
    } catch (error) {
      const errorMsg = error instanceof Error ? error.message : String(error);
      console.error(`Failed to process record ${record.id}:`, errorMsg);
      
      failed.push({
        record,
        error: errorMsg
      });
    }
  }

  // Report summary
  console.log(
    `Processed ${successful.length} records, ${failed.length} failures`
  );

  // Write failures to dead letter queue for investigation
  if (failed.length > 0) {
    await writeToDLQ(failed);
  }

  return { successful, failed };
}
```

## Checklist

### Data Parsing & Validation
- [ ] Use appropriate libraries for file formats (pandas, csv-parse, xml2js)
- [ ] Define explicit schemas with validation (Pydantic, Zod)
- [ ] Handle encoding issues (UTF-8, BOM)
- [ ] Validate data types and constraints
- [ ] Handle missing or null values explicitly
- [ ] Log validation errors with context

### Data Transformation
- [ ] Use type-safe transformations
- [ ] Preserve data lineage and traceability
- [ ] Handle edge cases (empty data, nulls, outliers)
- [ ] Document transformation logic clearly
- [ ] Test transformations with realistic data
- [ ] Use efficient data structures (Map, Set for lookups)

### ETL Pipeline Design
- [ ] Implement proper error handling and retries
- [ ] Use checkpointing for long-running pipelines
- [ ] Make operations idempotent
- [ ] Track pipeline metrics (duration, throughput, errors)
- [ ] Implement incremental loading where possible
- [ ] Log pipeline progress and failures
- [ ] Use transactions for atomic operations

### Performance & Optimization
- [ ] Process large files in chunks or streams
- [ ] Use parallel processing for CPU-bound tasks
- [ ] Optimize database queries (indexes, batch inserts)
- [ ] Monitor memory usage
- [ ] Profile slow transformations
- [ ] Cache expensive computations when possible

### Error Handling
- [ ] Never swallow errors silently
- [ ] Distinguish retryable vs non-retryable errors
- [ ] Implement exponential backoff for retries
- [ ] Log errors with full context
- [ ] Write failed records to dead letter queue
- [ ] Alert on high error rates

### Testing & Monitoring
- [ ] Test with realistic data volumes
- [ ] Test edge cases (empty files, malformed data)
- [ ] Set up monitoring and alerting
- [ ] Track data quality metrics
- [ ] Implement data validation tests
- [ ] Create runbooks for common failures
