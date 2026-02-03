# Design Document

## Overview

RetailMind AI is a comprehensive retail analytics platform built on a microservices architecture that delivers intelligent commerce decision-making capabilities. The system integrates machine learning-powered demand forecasting, natural language AI assistance, competitive market intelligence, automated inventory optimization, and secure multi-tenant user management.

The platform is designed as a cloud-native solution with event-driven communication, API-first design principles, and comprehensive observability. Each core service operates independently while maintaining data consistency through well-defined interfaces and asynchronous messaging patterns.

## Architecture

### High-Level Architecture

The system follows a layered microservices architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                Frontend Layer (React + Next.js)             │
│              Web Dashboard + AI Copilot Interface           │
└────────────────────┬────────────────────────────────────────┘
                     │ HTTPS/WebSocket
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              API Gateway (Kong/AWS API Gateway)             │
│         Authentication, Rate Limiting, Routing              │
└───┬──────────────┬──────────────┬──────────────┬───────────┘
    │              │              │              │
    ▼              ▼              ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│Forecast  │  │Market    │  │AI        │  │Inventory │
│Service   │  │Intel     │  │Copilot   │  │Service   │
│(Python)  │  │Service   │  │Service   │  │(Node.js) │
└────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │             │
     ▼             ▼             ▼             ▼
┌─────────────────────────────────────────────────────────────┐
│                   Data Layer                                 │
│  PostgreSQL  │  TimescaleDB  │  MongoDB  │  Redis  │ Vector │
│  (Metadata)  │ (Time Series) │(Documents)│ (Cache) │   DB   │
└─────────────────────────────────────────────────────────────┘
```

### Service Communication Patterns

**Synchronous Communication:**
- Frontend ↔ API Gateway ↔ Services (REST/GraphQL)
- Critical real-time operations requiring immediate consistency

**Asynchronous Communication:**
- Event-driven workflows via message queues (RabbitMQ/Kafka)
- Long-running ML training and batch processing
- Notification distribution and alert processing

**Message Topics:**
- `forecast.generated` - New predictions available
- `inventory.low` - Stock below reorder point
- `price.changed` - Competitor price updates
- `alert.triggered` - System notifications

## Components and Interfaces

### Forecast Service

**Technology Stack:** Python 3.11+, FastAPI, scikit-learn, XGBoost, Prophet, TensorFlow

**Core Components:**
- **Model Manager:** Handles ARIMA, Prophet, XGBoost, LSTM, and ensemble models
- **Feature Engineer:** Creates lag features, rolling statistics, seasonality indicators
- **Data Validator:** Ensures data quality and schema compliance
- **Training Orchestrator:** Manages async model training workflows
- **Prediction Engine:** Generates forecasts with confidence intervals

**Key Interfaces:**
```python
class ForecastService:
    async def train_models(self, product_id: str, sales_data: DataFrame) -> TrainingResult
    async def generate_forecast(self, product_id: str, horizon_days: int) -> ForecastResult
    async def evaluate_model_performance(self, model_id: str) -> ModelMetrics
    async def get_feature_importance(self, model_id: str) -> FeatureImportance
```

**Algorithm Selection Logic:**
- Time series length < 100 points: Simple exponential smoothing
- Seasonal patterns detected: Prophet or SARIMA
- Complex non-linear patterns: XGBoost or LSTM
- High accuracy requirements: Ensemble of top 3 models

### AI Copilot Service

**Technology Stack:** Python 3.11+, FastAPI, OpenAI GPT-4, Pinecone/Weaviate, LangChain

**Core Components:**
- **Query Processor:** Intent classification and entity extraction
- **Context Retriever:** Semantic search in vector database
- **Response Generator:** LLM inference with business context
- **Visualization Engine:** Auto-generates charts and dashboards
- **Knowledge Base:** Maintains retail domain expertise

**RAG Pipeline:**
```python
class AICopilot:
    async def process_query(self, query: str, user_context: UserContext) -> CopilotResponse
    async def classify_intent(self, query: str) -> QueryIntent
    async def retrieve_context(self, query: str, top_k: int = 5) -> List[Document]
    async def generate_response(self, query: str, context: List[Document]) -> Response
```

**Intent Categories:**
- **Data Query:** "What were my top products last month?"
- **Insight Request:** "Why did sales drop in Q3?"
- **Recommendation:** "How should I price this product?"
- **Comparison:** "How do I compare to competitors?"

### Market Intelligence Service

**Technology Stack:** Python 3.11+, FastAPI, BeautifulSoup, Selenium, HuggingFace Transformers

**Core Components:**
- **Web Scraper Manager:** Platform-specific scrapers (Amazon, Walmart, Target)
- **Price Monitor:** Tracks competitor pricing with change detection
- **Sentiment Analyzer:** Processes reviews with aspect-based analysis
- **Data Aggregator:** Consolidates market intelligence data
- **Alert Engine:** Triggers notifications on significant changes

**Scraping Architecture:**
```python
class MarketIntelligenceService:
    async def monitor_competitor_price(self, product_url: str, frequency: str) -> MonitoringJob
    async def scrape_product_data(self, url: str) -> ProductData
    async def analyze_sentiment(self, reviews: List[str]) -> SentimentAnalysis
    async def detect_price_changes(self, product_id: str) -> List[PriceChange]
```

**Sentiment Analysis Pipeline:**
- Overall sentiment classification (positive/negative/neutral)
- Aspect-based sentiment (quality, price, service, delivery)
- Confidence scoring and review aggregation
- Trend analysis over time periods

### Inventory Service

**Technology Stack:** TypeScript, Node.js, NestJS, PostgreSQL

**Core Components:**
- **Reorder Calculator:** Computes optimal reorder points and safety stock
- **EOQ Optimizer:** Calculates Economic Order Quantity
- **Stock Monitor:** Tracks inventory levels against thresholds
- **Demand Analyzer:** Processes historical sales patterns
- **Alert Generator:** Creates low stock and reorder notifications

**Optimization Algorithms:**
```typescript
class InventoryOptimizationService {
  calculateReorderPoint(sku: string): Promise<ReorderPoint>
  calculateSafetyStock(demand_history: number[], lead_time: number): number
  calculateEOQ(annual_demand: number, ordering_cost: number, holding_cost: number): number
  optimizeInventoryLevels(products: Product[]): Promise<OptimizationResult>
}
```

**Reorder Point Formula:**
- Reorder Point = (Average Daily Demand × Lead Time) + Safety Stock
- Safety Stock = Z-score × Demand Std Dev × √Lead Time
- Service Level: 95% (Z-score = 1.65)

### User Authentication Service

**Technology Stack:** Node.js, Express, PostgreSQL, JWT, bcrypt

**Core Components:**
- **Authentication Manager:** Handles login, registration, password reset
- **Authorization Engine:** Role-based access control (RBAC)
- **Session Manager:** JWT token generation and validation
- **Multi-tenant Isolator:** Ensures data separation between organizations
- **Audit Logger:** Tracks authentication and authorization events

**Security Features:**
```typescript
class AuthService {
  async authenticate(email: string, password: string): Promise<AuthResult>
  async authorize(user_id: string, resource: string, action: string): Promise<boolean>
  generateJWT(user: User): string
  validateJWT(token: string): Promise<TokenPayload>
  enforceDataIsolation(user_id: string, resource_id: string): Promise<boolean>
}
```

**Role Hierarchy:**
- **Owner:** Full access to all features and data
- **Manager:** Access to analytics, forecasting, inventory management
- **Analyst:** Read-only access to reports and insights
- **Viewer:** Limited dashboard access

## Data Models

### Core Entities

**User Entity:**
```typescript
interface User {
  id: string;
  email: string;
  password_hash: string;
  full_name: string;
  business_name: string;
  industry: string;
  role: 'owner' | 'manager' | 'analyst' | 'viewer';
  subscription_tier: 'free' | 'pro' | 'enterprise';
  created_at: Date;
  is_active: boolean;
}
```

**Product Entity:**
```typescript
interface Product {
  id: string;
  user_id: string;
  sku: string;
  name: string;
  description: string;
  category: string;
  brand: string;
  current_price: number;
  cost_price: number;
  current_stock: number;
  reorder_point: number;
  safety_stock: number;
  eoq: number;
  supplier_id: string;
  lead_time_days: number;
  created_at: Date;
  is_active: boolean;
}
```

**Sales Entity (Time-Series):**
```typescript
interface SalesRecord {
  time: Date;
  user_id: string;
  product_id: string;
  sku: string;
  quantity: number;
  unit_price: number;
  total_amount: number;
  location: string;
  channel: 'online' | 'store' | 'marketplace';
  discount_amount: number;
  customer_id: string;
}
```

**Forecast Entity:**
```typescript
interface Forecast {
  id: string;
  user_id: string;
  product_id: string;
  forecast_date: Date;
  predicted_demand: number;
  lower_bound: number;
  upper_bound: number;
  confidence_level: number;
  model_used: string;
  model_accuracy: number;
  created_at: Date;
}
```

**Market Intelligence Entity:**
```typescript
interface MarketIntelligence {
  id: string;
  product_url: string;
  competitor_name: string;
  product_name: string;
  price: number;
  currency: string;
  in_stock: boolean;
  rating: number;
  review_count: number;
  scraped_at: Date;
  price_history: PricePoint[];
  sentiment_analysis: SentimentData;
}
```

### Database Schema Design

**PostgreSQL (Relational Data):**
- Users, Products, Forecasts tables with proper indexing
- Foreign key relationships ensuring referential integrity
- Optimized queries with composite indexes on frequently accessed columns

**TimescaleDB (Time-Series Data):**
- Sales hypertable partitioned by time
- Continuous aggregates for daily/weekly/monthly metrics
- Automatic data retention policies for historical data

**MongoDB (Document Store):**
- Market intelligence data with flexible schema
- Review and sentiment analysis documents
- Geospatial indexes for location-based analysis

**Redis (Cache & Sessions):**
- User session storage with TTL
- Forecast result caching
- Rate limiting counters
- Real-time alert queues

### Data Flow Patterns

**Forecast Generation Flow:**
1. User uploads sales data → Data validation → Feature engineering
2. Model training (async) → Model selection → Prediction generation
3. Results stored in TimescaleDB → Cache in Redis → Notify user

**AI Copilot Query Flow:**
1. User query → Intent classification → Context retrieval from vector DB
2. LLM inference with business context → Response formatting
3. Visualization generation → Return structured response

**Market Intelligence Flow:**
1. Scheduled scraping jobs → Data extraction → Price change detection
2. Sentiment analysis → Data aggregation → Alert generation
3. Store in MongoDB → Cache recent data → Update dashboards