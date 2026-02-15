# Design: DemandIQ – Autonomous Market Intelligence Copilot for SMEs

## 1. Design Principles

- **Modularity**: Separate concerns across data, agent orchestration, and ML services
- **Explainability**: Every recommendation includes natural language explanations via LLaMA 4
- **Scalability**: Stateless services with async processing for long-running analyses
- **Security-First**: JWT authentication, row-level security, encrypted data at rest and in transit
- **Simplicity**: Minimize complexity to deliver working prototype within hackathon timeline

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Frontend Layer                          │
│                    (Angular Dashboard)                          │
└────────────────────────────┬────────────────────────────────────┘
                             │ HTTPS/REST
┌────────────────────────────┴────────────────────────────────────┐
│                         API Gateway                             │
│                    (FastAPI + JWT Auth)                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
┌───────▼────────┐  ┌────────▼────────┐  ┌───────▼────────┐
│  Data Service  │  │  Agent Service  │  │  ML Service    │
│   (FastAPI)    │  │   (Node.js)     │  │  (FastAPI)     │
└───────┬────────┘  └────────┬────────┘  └───────┬────────┘
        │                    │                    │
        │           ┌────────▼────────┐           │
        │           │  Multi-Agent    │           │
        │           │  Orchestrator   │           │
        │           │  - Planner      │           │
        │           │  - Retrieval    │           │
        │           │  - Executor     │           │
        │           └────────┬────────┘           │
        │                    │                    │
┌───────▼────────────────────▼────────────────────▼────────┐
│                    Data Layer                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  PostgreSQL  │  │   MongoDB    │  │   AWS S3     │  │
│  │ (Structured) │  │ (Logs/Msgs)  │  │ (CSV Files)  │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### 2.2 Component Responsibilities

**Frontend**: Angular SPA with Chart.js for visualizations, responsive design for desktop/tablet

**API Gateway**: FastAPI REST endpoints, JWT authentication, request validation

**Data Service**: CSV upload/validation, data storage, user management

**Agent Service**: Multi-agent orchestration, task decomposition, agent communication

**ML Service**: Demand forecasting (ARIMA), inventory classification, pricing optimization

**Data Layer**:
- PostgreSQL: Users, products, sales, forecasts, recommendations
- MongoDB: Agent execution logs and inter-agent messages
- S3: Raw CSV file storage

## 3. Multi-Agent Architecture

### 3.1 Agent Communication Flow

```
User Request → API Gateway → Agent Orchestrator
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              Planner Agent   Retrieval Agent  Executor Agent
                    │               │               │
                    └───────────────┴───────────────┘
                                    │
                            Result Aggregation
                                    │
                            Response to User
```

### 3.2 Agent Specifications

#### Planner Agent
**Purpose**: Decompose business objectives into executable tasks

**Implementation**: LLaMA 4 for natural language understanding and task decomposition

**Example Output**:
```json
{
  "objective": "Analyze inventory health",
  "tasks": [
    {
      "id": "task_1",
      "type": "data_retrieval",
      "description": "Fetch sales data for last 90 days",
      "agent": "retrieval"
    },
    {
      "id": "task_2",
      "type": "ml_analysis",
      "description": "Calculate sales velocity per product",
      "agent": "executor",
      "depends_on": ["task_1"]
    },
    {
      "id": "task_3",
      "type": "classification",
      "description": "Classify inventory risk levels",
      "agent": "executor",
      "depends_on": ["task_2"]
    }
  ]
}
```

#### Retrieval Agent
**Purpose**: Fetch relevant data from databases based on Planner specifications

**Capabilities**:
- SQL query builder for PostgreSQL (sales, products, inventory)
- MongoDB query interface for historical agent logs
- Data aggregation and filtering by date range, product, user

#### Executor Agent
**Purpose**: Perform ML tasks and generate recommendations with explanations

**Capabilities**:
- Demand forecasting using ARIMA/Prophet models
- Inventory risk classification (slow-moving, overstock, stockout)
- Price optimization using elasticity-based algorithms
- Revenue impact estimation
- Natural language explanation generation via LLaMA 4

### 3.3 Agent Communication Protocol

**Message Format**:
```json
{
  "message_id": "uuid",
  "timestamp": "ISO8601",
  "from_agent": "planner|retrieval|executor",
  "to_agent": "planner|retrieval|executor",
  "message_type": "request|response|error",
  "payload": {
    "task_id": "string",
    "data": {}
  }
}
```

All agent messages are logged to MongoDB for auditability and debugging.

## 4. Data Models

### 4.1 PostgreSQL Schema

#### Users
```sql
CREATE TABLE users (
  user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Products
```sql
CREATE TABLE products (
  product_id VARCHAR(50) PRIMARY KEY,
  user_id UUID REFERENCES users(user_id),
  product_name VARCHAR(255),
  category VARCHAR(100),
  current_price DECIMAL(10, 2),
  current_stock INTEGER,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Sales
```sql
CREATE TABLE sales (
  sale_id SERIAL PRIMARY KEY,
  user_id UUID REFERENCES users(user_id),
  product_id VARCHAR(50) REFERENCES products(product_id),
  sale_date DATE NOT NULL,
  quantity INTEGER NOT NULL,
  unit_price DECIMAL(10, 2) NOT NULL,
  total_amount DECIMAL(10, 2) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_sales_user_date ON sales(user_id, sale_date);
CREATE INDEX idx_sales_product ON sales(product_id);
```

#### Forecasts
```sql
CREATE TABLE forecasts (
  forecast_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(user_id),
  product_id VARCHAR(50) REFERENCES products(product_id),
  forecast_date DATE NOT NULL,
  forecast_period INTEGER, -- 30, 60, or 90 days
  predicted_demand DECIMAL(10, 2),
  confidence_score DECIMAL(5, 2),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Recommendations
```sql
CREATE TABLE recommendations (
  recommendation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(user_id),
  product_id VARCHAR(50) REFERENCES products(product_id),
  recommendation_type VARCHAR(50), -- 'pricing', 'inventory', 'forecast'
  current_value DECIMAL(10, 2),
  recommended_value DECIMAL(10, 2),
  estimated_impact DECIMAL(10, 2),
  explanation TEXT,
  status VARCHAR(20) DEFAULT 'pending', -- 'pending', 'accepted', 'rejected'
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 4.2 MongoDB Collections

#### Agent Logs
```javascript
{
  _id: ObjectId,
  execution_id: "uuid",
  agent_type: "planner|retrieval|executor",
  timestamp: ISODate,
  user_id: "uuid",
  task_description: "string",
  input_data: {},
  output_data: {},
  execution_time_ms: Number,
  status: "success|error"
}
```

#### Agent Messages
```javascript
{
  _id: ObjectId,
  message_id: "uuid",
  execution_id: "uuid",
  from_agent: "string",
  to_agent: "string",
  message_type: "request|response|error",
  payload: {},
  timestamp: ISODate
}
```

## 5. API Specifications

### 5.1 Authentication

**POST /api/auth/register**
```json
Request: {"email": "user@example.com", "password": "securepassword"}
Response: {"user_id": "uuid", "token": "jwt_token"}
```

**POST /api/auth/login**
```json
Request: {"email": "user@example.com", "password": "securepassword"}
Response: {"user_id": "uuid", "token": "jwt_token", "expires_in": 3600}
```

### 5.2 Data Ingestion

**POST /api/data/upload** (multipart/form-data, requires auth)
```json
Response: {
  "upload_id": "uuid",
  "filename": "sales_data.csv",
  "records_processed": 1500,
  "validation_errors": [],
  "status": "success"
}
```

### 5.3 Analysis

**POST /api/analysis/forecast** (requires auth)
```json
Request: {"product_ids": ["PROD001"], "forecast_period": 30}
Response: {
  "execution_id": "uuid",
  "status": "processing",
  "estimated_completion": "2026-02-15T10:35:00Z"
}
```

**GET /api/analysis/forecast/{execution_id}** (requires auth)
```json
Response: {
  "execution_id": "uuid",
  "status": "completed",
  "forecasts": [{
    "product_id": "PROD001",
    "forecast_period": 30,
    "predicted_demand": 450,
    "confidence_score": 85,
    "explanation": "Based on historical sales trends showing 12% monthly growth..."
  }]
}
```

**POST /api/analysis/inventory-risk** (requires auth)
```json
Request: {"analysis_type": "all"}
Response: {
  "risks": [{
    "product_id": "PROD003",
    "risk_type": "slow_moving",
    "severity": "high",
    "current_stock": 500,
    "days_of_supply": 180,
    "recommendation": "Consider discount promotion",
    "explanation": "Sales velocity decreased 40% over last 60 days"
  }]
}
```

**POST /api/analysis/pricing** (requires auth)
```json
Request: {"product_ids": ["PROD001"]}
Response: {
  "recommendations": [{
    "product_id": "PROD001",
    "current_price": 29.99,
    "recommended_price": 27.99,
    "price_change_percent": -6.67,
    "estimated_revenue_impact": 1250.00,
    "explanation": "Price elasticity analysis suggests 6.7% reduction will increase volume by 15%"
  }]
}
```

### 5.4 Dashboard

**GET /api/dashboard/summary** (requires auth)
```json
Response: {
  "total_products": 50,
  "total_sales_last_30_days": 15000.00,
  "active_recommendations": 12,
  "inventory_alerts": 3,
  "forecast_accuracy": 82.5
}
```

**GET /api/dashboard/metrics?period=30** (requires auth)
```json
Response: {
  "period": 30,
  "sales_trend": [{"date": "2026-01-16", "amount": 500.00}],
  "top_products": [{"product_id": "PROD001", "revenue": 5000.00}],
  "inventory_health": {"healthy": 40, "at_risk": 8, "critical": 2}
}
```

### 5.5 Recommendations

**GET /api/recommendations?type=pricing&status=pending** (requires auth)
```json
Response: {
  "recommendations": [{
    "recommendation_id": "uuid",
    "type": "pricing",
    "product_id": "PROD001",
    "current_value": 29.99,
    "recommended_value": 27.99,
    "estimated_impact": 1250.00,
    "explanation": "...",
    "status": "pending",
    "created_at": "2026-02-15T10:00:00Z"
  }]
}
```

**PUT /api/recommendations/{recommendation_id}** (requires auth)
```json
Request: {"status": "accepted"}
Response: {"recommendation_id": "uuid", "status": "accepted"}
```

## 6. Machine Learning Models

### 6.1 Demand Forecasting

**Algorithm**: ARIMA (AutoRegressive Integrated Moving Average) via statsmodels

**Features**: Historical sales data (quantity, date), seasonality indicators, trend components

**Requirements**: Minimum 60 days of historical data

**Output**: Point forecast for 30/60/90 days with confidence intervals and confidence score (0-100)

**Implementation**:
```python
from statsmodels.tsa.arima.model import ARIMA

def forecast_demand(sales_data, periods=30):
    model = ARIMA(sales_data, order=(1,1,1))
    fitted = model.fit()
    forecast = fitted.forecast(steps=periods)
    confidence = calculate_confidence_score(fitted)
    return forecast, confidence
```

### 6.2 Inventory Classification

**Algorithm**: Rule-based classification with ML scoring

**Metrics**:
- Sales velocity: units sold per day
- Days of supply: current_stock / avg_daily_sales
- Trend: sales growth rate over last 30 days

**Classification Rules**:
- **Slow-moving**: velocity < 1 unit/day AND days_of_supply > 90
- **Overstock**: days_of_supply > 120
- **Stockout risk**: days_of_supply < 14 AND trend > 0

**Severity Levels**: High (>180 days), Medium (120-180 days), Low (<120 days)

### 6.3 Pricing Optimization

**Algorithm**: Price elasticity estimation + revenue maximization

**Model**:
```
demand = base_demand × (1 + elasticity × price_change_percent)
revenue = demand × new_price
```

**Elasticity Estimation**: Historical price-demand correlation via linear regression on log-transformed data (default elasticity: -1.5)

**Implementation**:
```python
def optimize_price(current_price, base_demand, elasticity):
    optimal_price = current_price * (1 + 1/(2*elasticity))
    estimated_demand = base_demand * (1 + elasticity * (optimal_price/current_price - 1))
    estimated_revenue = optimal_price * estimated_demand
    return optimal_price, estimated_revenue
```

## 7. Explainable AI Implementation

### 7.1 LLaMA 4 Integration

**Model**: LLaMA 4 (8B parameters) for natural language explanation generation

**Prompt Template**:
```
You are a business analyst explaining data-driven recommendations to SME owners.

Context:
- Product: {product_name}
- Current Metric: {current_value}
- Recommended Action: {recommendation}
- Supporting Data: {data_points}

Generate a clear, concise explanation (2-3 sentences) that:
1. States what the data shows
2. Explains why the recommendation makes sense
3. Mentions the expected business impact

Explanation:
```

**Example Output**: "Your sales data shows Product A has been selling 15% slower over the past 60 days, with current stock lasting 180 days. Reducing the price by 7% can stimulate demand and prevent inventory obsolescence. This adjustment is projected to increase revenue by $1,250 over the next 30 days."

**Quality Criteria**: References specific data points, mentions timeframes, includes quantified impact, uses business-friendly language (50-150 words)

## 8. Security Design

### 8.1 Authentication & Authorization

**JWT Authentication**:
- Token expiration: 1 hour
- Claims: user_id, email, issued_at, expires_at
- bcrypt password hashing (cost factor: 12)

**Row-Level Security**:
- All queries filtered by user_id
- PostgreSQL RLS policies enforce data isolation
- MongoDB queries include user_id filter

**Example RLS Policy**:
```sql
CREATE POLICY user_sales_policy ON sales
  FOR ALL TO authenticated_user
  USING (user_id = current_setting('app.user_id')::uuid);
```

### 8.2 Data Protection

**At Rest**: PostgreSQL encryption (AWS RDS), MongoDB encrypted collections, S3 server-side encryption (SSE-S3)

**In Transit**: HTTPS/TLS 1.3 for all API communication

**Password Security**: bcrypt hashing, minimum 8 characters, no passwords in logs

## 9. Frontend Design

### 9.1 Dashboard Layout

```
┌─────────────────────────────────────────────────────────┐
│  Header: DemandIQ | User Menu | Logout                  │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │ Total Sales │  │ Active Recs │  │ Alerts      │    │
│  │  $15,000    │  │     12      │  │     3       │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
├─────────────────────────────────────────────────────────┤
│  Sales Trend (30 days)                                  │
│  ┌───────────────────────────────────────────────────┐ │
│  │         [Line Chart - Chart.js]                   │ │
│  └───────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────┤
│  Inventory Health          │  Top Recommendations      │
│  ┌─────────────────────┐   │  ┌─────────────────────┐ │
│  │ [Donut Chart]       │   │  │ 1. Reduce price...  │ │
│  │ Healthy: 40         │   │  │ 2. Restock...       │ │
│  │ At Risk: 8          │   │  │ 3. Discount...      │ │
│  │ Critical: 2         │   │  └─────────────────────┘ │
│  └─────────────────────┘   │                          │
└─────────────────────────────────────────────────────────┘
```

### 9.2 Key Components

**MetricCard**: Single metric display with icon, color-coded status, click-to-drill-down

**ChartComponent**: Chart.js wrapper supporting line, bar, donut charts with responsive design

**RecommendationCard**: Recommendation details with Accept/Reject buttons and expandable explanation

**Technology**: Angular with lazy loading, route-based code splitting, Chart.js for visualizations

## 10. Deployment Architecture

### 10.1 AWS Infrastructure

```
Route 53 (DNS)
    ↓
CloudFront (CDN) → Angular static files
    ↓
Application Load Balancer
    ↓
┌─────────────┬─────────────┬─────────────┐
│ EC2 (FastAPI)│ EC2 (Node.js)│ EC2 (FastAPI)│
│ Data Service│ Agent Service│  ML Service  │
└─────────────┴─────────────┴─────────────┘
    ↓               ↓               ↓
┌───────────┬───────────────┬───────────┐
│ RDS       │ MongoDB Atlas │    S3     │
│(PostgreSQL)│   (Logs)     │ (Storage) │
└───────────┴───────────────┴───────────┘
```

### 10.2 Environment Configuration

**Hackathon Demo Environment**:
- 2× EC2 t3.medium instances (API + Agent services)
- RDS db.t3.small with Multi-AZ
- MongoDB Atlas M10 cluster
- S3 with versioning enabled
- CloudFront for static asset delivery

### 10.3 Performance Targets

- CSV processing: <30 seconds for 10,000 records
- Dashboard load: <2 seconds
- API response time: <500ms
- Concurrent users: 10+ simultaneous sessions

## 11. Testing Strategy

### 11.1 Unit Testing
- **Backend**: pytest for Python (>80% coverage), Jest for Node.js
- **Frontend**: Jasmine/Karma for Angular components
- Mock external dependencies (databases, LLaMA API)

### 11.2 Integration Testing
- Complete API request/response cycles
- JWT authentication flows
- Agent communication protocol validation
- Error handling and recovery

### 11.3 End-to-End Testing
- User flows: Registration → Login → Upload → Dashboard → Recommendations
- Test with synthetic datasets (100, 1,000, 10,000 records)
- Validate forecast accuracy and explanation quality

## 12. Success Criteria

### 12.1 Functional Completeness
✅ User registration and authentication
✅ CSV upload with validation
✅ Demand forecasting with confidence scores
✅ Inventory risk detection with severity levels
✅ Pricing recommendations with revenue impact
✅ Dashboard with interactive visualizations
✅ Natural language explanations for all recommendations

### 12.2 Performance & Quality
✅ Meet performance targets (processing time, response time, load time)
✅ Multi-agent orchestration working end-to-end
✅ Data security and user isolation enforced
✅ Professional UI/UX suitable for SME users

### 12.3 Demo Readiness
✅ End-to-end demo with synthetic dataset
✅ All core features functional in production environment
✅ Clear demonstration of business value and measurable impact
