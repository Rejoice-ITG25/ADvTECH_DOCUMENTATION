## 6. API Specifications

### 6.1 POS Transaction APIs

**POST /api/v1/transactions/process**
```typescript
interface ProcessPaymentRequest {
  studentId: string;
  amount: number;
  paymentMethod: 'card' | 'cash';
  feeAllocations: FeeAllocation[];
  posDeviceId: string;
  cashierId: string;
  paymentDetails?: {
    cardToken?: string;
    last4Digits?: string;
    cashReceived?: number;
    changeGiven?: number;
  };
  metadata?: {
    deviceLocation?: string;
    transactionNotes?: string;
  };
}

interface ProcessPaymentResponse {
  transactionId: string;
  status: 'processing' | 'completed' | 'failed';
  receiptUrl: string;
  remainingBalance: number;
  estimatedProcessingTime: number; // seconds
  paymentReference: string;
  gatewayTransactionId?: string;
}
```

**GET /api/v1/students/{studentId}/lookup**
```typescript
interface StudentLookupResponse {
  studentId: string;
  firstName: string;
  lastName: string;
  email: string;
  phone: string;
  outstandingBalance: number;
  feeStructure: FeeStructure[];
  paymentHistory: PaymentSummary[];
  accountStatus: 'active' | 'suspended' | 'inactive';
  lastPaymentDate?: string;
  nextDueDate?: string;
}

interface FeeStructure {
  feeType: string;
  description: string;
  amount: number;
  dueDate: string;
  academicYear: string;
  status: 'outstanding' | 'paid' | 'overdue';
}
```

**GET /api/v1/transactions/{transactionId}/status**
```typescript
interface TransactionStatusResponse {
  transactionId: string;
  status: TransactionStatus;
  amount: number;
  studentId: string;
  paymentMethod: string;
  processedAt?: string;
  simsUpdateStatus: 'pending' | 'completed' | 'failed';
  errorMessage?: string;
  retryCount: number;
  receiptUrl?: string;
}
```

### 6.2 Reconciliation APIs

**GET /api/v1/reconciliation/transactions**
```typescript
interface ReconciliationQuery {
  startDate: string;
  endDate: string;
  status?: TransactionStatus[];
  paymentMethod?: string[];
  campusId?: string;
  cashierId?: string;
  studentId?: string;
  amountRange?: {
    min: number;
    max: number;
  };
  limit: number;
  offset: number;
  sortBy?: 'timestamp' | 'amount' | 'status';
  sortOrder?: 'asc' | 'desc';
}

interface ReconciliationResponse {
  transactions: TransactionSummary[];
  summary: {
    totalCount: number;
    totalAmount: number;
    successfulCount: number;
    failedCount: number;
    pendingCount: number;
    averageTransactionTime: number;
  };
  pagination: PaginationInfo;
  filters: AppliedFilters;
}
```

**POST /api/v1/reconciliation/manual-reconcile**
```typescript
interface ManualReconcileRequest {
  transactionId: string;
  action: 'retry' | 'mark_completed' | 'cancel';
  reason: string;
  userId: string;
  notes?: string;
}

interface ManualReconcileResponse {
  success: boolean;
  transactionId: string;
  newStatus: TransactionStatus;
  auditTrail: AuditEntry;
}
```

**GET /api/v1/reconciliation/dashboard**
```typescript
interface DashboardResponse {
  realTimeMetrics: {
    transactionsToday: number;
    revenueToday: number;
    averageTransactionValue: number;
    successRate: number;
    pendingTransactions: number;
  };
  trends: {
    hourlyVolume: HourlyMetric[];
    paymentMethodBreakdown: PaymentMethodMetric[];
    feeTypeDistribution: FeeTypeMetric[];
  };
  alerts: SystemAlert[];
  systemHealth: HealthStatus;
}
```

### 6.3 SIMS Integration APIs

**POST /api/v1/sims/balance-update**
```typescript
interface BalanceUpdateRequest {
  studentId: string;
  transactionId: string;
  amount: number;
  feeAllocations: FeeAllocation[];
  paymentReference: string;
  paymentMethod: string;
  processedAt: string;
}

interface BalanceUpdateResponse {
  success: boolean;
  studentId: string;
  newBalance: number;
  updateReference: string;
  timestamp: string;
}
```

**GET /api/v1/sims/student/{studentId}/validate**
```typescript
interface StudentValidationResponse {
  isValid: boolean;
  studentDetails?: {
    studentId: string;
    firstName: string;
    lastName: string;
    email: string;
    accountStatus: string;
    currentBalance: number;
  };
  errorCode?: string;
  errorMessage?: string;
}
```

---
