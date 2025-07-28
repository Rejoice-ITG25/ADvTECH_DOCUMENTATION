## Database Design

### 5.1 Write-Optimized DynamoDB Schema

**Transaction Table (Primary Write Destination):**
```typescript
interface TransactionRecord {
  PK: string;              // TRANSACTION#{transactionId}
  SK: string;              // METADATA#{timestamp}
  transactionId: string;
  studentId: string;
  paymentMethod: string;
  amount: number;
  currency: string;
  feeAllocations: FeeAllocation[];
  status: TransactionStatus;
  gatewayResponse: object;
  timestamp: string;
  campusId: string;
  posDeviceId: string;
  cashierId: string;
  receiptUrl?: string;
  simsUpdateStatus: 'pending' | 'completed' | 'failed';
  retryCount: number;
  TTL?: number;           // For data lifecycle management
}

interface FeeAllocation {
  feeType: string;
  amount: number;
  description: string;
  academicYear: string;
  glAccount?: string;     // For accounting integration
}
```

**Student Lookup Cache Table:**
```typescript
interface StudentCache {
  PK: string;              // STUDENT#{studentId}
  SK: string;              // PROFILE#{timestamp}
  studentId: string;
  firstName: string;
  lastName: string;
  email: string;
  phone: string;
  outstandingBalance: number;
  feeStructure: FeeStructure[];
  paymentHistory: PaymentSummary[];
  accountStatus: 'active' | 'suspended' | 'inactive';
  lastUpdated: string;
  TTL: number;            // 30-minute cache expiry
}
```

**Reconciliation State Table:**
```typescript
interface ReconciliationRecord {
  PK: string;              // RECON#{date}
  SK: string;              // STATUS#{timestamp}
  date: string;
  totalTransactions: number;
  totalAmount: number;
  reconciledTransactions: number;
  pendingTransactions: number;
  failedTransactions: number;
  lastReconcileTime: string;
  exceptions: ReconciliationException[];
}
```

### 5.2 Read-Optimized RDS Schema

**Analytics and Reporting Database (PostgreSQL):**
```sql
-- Optimized for complex queries and aggregations
CREATE TABLE transactions (
    transaction_id VARCHAR(50) PRIMARY KEY,
    student_id VARCHAR(20) NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'ZAR',
    payment_method VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL,
    processed_at TIMESTAMP WITH TIME ZONE NOT NULL,
    campus_id VARCHAR(10) NOT NULL,
    pos_device_id VARCHAR(20) NOT NULL,
    cashier_id VARCHAR(20) NOT NULL,
    gateway_transaction_id VARCHAR(100),
    sims_update_status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_transactions_student_date ON transactions(student_id, processed_at);
CREATE INDEX idx_transactions_campus_date ON transactions(campus_id, processed_at);
CREATE INDEX idx_transactions_status ON transactions(status) WHERE status != 'completed';
CREATE INDEX idx_transactions_sims_status ON transactions(sims_update_status) WHERE sims_update_status != 'completed';

-- Fee allocations for detailed reporting
CREATE TABLE fee_allocations (
    allocation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transaction_id VARCHAR(50) REFERENCES transactions(transaction_id),
    fee_type VARCHAR(50) NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    description TEXT,
    academic_year VARCHAR(10) NOT NULL,
    gl_account VARCHAR(20),
    allocation_date TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Student payment history for analytics
CREATE TABLE student_payments (
    student_id VARCHAR(20) NOT NULL,
    transaction_id VARCHAR(50) REFERENCES transactions(transaction_id),
    payment_date DATE NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    outstanding_balance DECIMAL(10,2),
    payment_method VARCHAR(20) NOT NULL,
    PRIMARY KEY (student_id, transaction_id)
);

-- Daily aggregations for performance
CREATE MATERIALIZED VIEW daily_transaction_summary AS
SELECT 
    DATE(processed_at) as transaction_date,
    campus_id,
    payment_method,
    COUNT(*) as transaction_count,
    SUM(amount) as total_amount,
    AVG(amount) as average_amount,
    COUNT(CASE WHEN status = 'completed' THEN 1 END) as successful_count,
    COUNT(CASE WHEN status = 'failed' THEN 1 END) as failed_count
FROM transactions 
GROUP BY DATE(processed_at), campus_id, payment_method;

-- Refresh strategy for materialized views
CREATE INDEX ON daily_transaction_summary (transaction_date, campus_id);
```

### 5.3 Data Synchronization Strategy

**DynamoDB to RDS Pipeline:**
- DynamoDB Streams trigger Lambda functions for real-time CDC
- Change Data Capture with guaranteed delivery using SQS
- Batch processing for efficiency (100 records/batch)
- Error handling with dead letter queues
- Automatic retry with exponential backoff
- Data validation and transformation during sync

**Data Consistency Patterns:**
- Eventually consistent reads from DynamoDB
- Strong consistency for critical financial operations
- Conflict resolution using last-write-wins with timestamps
- Data reconciliation jobs for detecting and fixing inconsistencies
- Monitoring and alerting for data sync failures

---
