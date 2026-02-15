# Implementation Plan: DemandIQ – Autonomous Market Intelligence Copilot

## Overview

This implementation plan breaks down the DemandIQ system into discrete coding tasks across backend services (Python FastAPI, Node.js), frontend (Angular), and infrastructure setup. The system consists of three main backend services (Data Service, Agent Service, ML Service), a PostgreSQL database, MongoDB for logs, and an Angular dashboard.

The implementation follows a bottom-up approach: database setup → core services → agent orchestration → ML models → frontend → integration.

## Tasks

- [ ] 1. Database setup and schema implementation
  - [ ] 1.1 Create PostgreSQL schema with all tables
    - Implement users, products, sales, forecasts, recommendations tables
    - Add indexes for performance (user_id, sale_date, product_id)
    - Implement Row-Level Security (RLS) policies for user data isolation
    - _Requirements: 2.1.5, 2.8.3_

  - [ ] 1.2 Create MongoDB collections and indexes
    - Set up agent_logs and agent_messages collections
    - Add indexes on execution_id, user_id, timestamp
    - Configure encryption at rest
    - _Requirements: 2.5.5_

  - [ ] 1.3 Create database connection utilities
    - Implement PostgreSQL connection pool with asyncpg
    - Implement MongoDB connection with motor (async driver)
    - Add connection health checks and retry logic
    - _Requirements: 2.1.5, 2.5.5_

- [ ] 2. Authentication and user management (Python FastAPI)
  - [ ] 2.1 Implement user registration endpoint
    - Create POST /api/auth/register endpoint
    - Implement password hashing with bcrypt (cost factor 12)
    - Validate email format and password strength
    - Return JWT token on successful registration
    - _Requirements: 2.8.1, 2.8.4_

  - [ ] 2.2 Implement user login endpoint
    - Create POST /api/auth/login endpoint
    - Verify credentials against hashed passwords
    - Generate JWT token with 1-hour expiration
    - Include user_id, email, issued_at, expires_at in claims
    - _Requirements: 2.8.1, 2.8.5_

  - [ ] 2.3 Create JWT authentication middleware
    - Implement token validation decorator for protected endpoints
    - Extract user_id from token and inject into request context
    - Handle token expiration and invalid token errors
    - _Requirements: 2.8.2, 2.8.5_

  - [ ]* 2.4 Write unit tests for authentication
    - Test registration with valid/invalid inputs
    - Test login with correct/incorrect credentials
    - Test JWT token generation and validation
    - Test token expiration handling
    - _Requirements: 2.8.1, 2.8.4_

- [ ] 3. Data ingestion service (Python FastAPI)
  - [ ] 3.1 Implement CSV upload endpoint
    - Create POST /api/data/upload endpoint with multipart/form-data
    - Save uploaded file to AWS S3 with user_id prefix
    - Return upload_id and filename in response
    - _Requirements: 2.1.1, 2.1.5_

  - [ ] 3.2 Implement CSV validation logic
    - Validate required fields: date, product_id, quantity, price
    - Check data types and value ranges (quantity > 0, price > 0)
    - Validate date format (ISO 8601 or common formats)
    - Return clear error messages for validation failures
    - _Requirements: 2.1.2, 2.1.3_

  - [ ] 3.3 Implement CSV data processing and storage
    - Parse validated CSV into sales records
    - Insert sales data into PostgreSQL sales table
    - Upsert product records into products table
    - Return record count and processing status
    - _Requirements: 2.1.4, 2.1.5_

  - [ ]* 3.4 Write unit tests for CSV processing
    - Test valid CSV upload and parsing
    - Test validation error handling (missing fields, invalid types)
    - Test duplicate product handling
    - Test large file processing (10,000 records)
    - _Requirements: 2.1.1, 2.1.2, 2.1.3, 2.1.4_

- [ ] 4. Checkpoint - Ensure authentication and data ingestion work
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 5. ML Service - Demand forecasting (Python FastAPI)
  - [ ] 5.1 Implement demand forecasting function
    - Create forecast_demand() using ARIMA from statsmodels
    - Accept sales_data time series and forecast_period (30/60/90 days)
    - Return predicted demand values and confidence intervals
    - Calculate confidence score (0-100) based on model fit
    - _Requirements: 2.2.1, 2.2.2_

  - [ ] 5.2 Create forecast API endpoint
    - Create POST /api/analysis/forecast endpoint
    - Fetch historical sales data for specified products
    - Call forecast_demand() for each product
    - Store forecast results in forecasts table
    - Return execution_id and status
    - _Requirements: 2.2.1, 2.2.5_

  - [ ] 5.3 Create forecast retrieval endpoint
    - Create GET /api/analysis/forecast/{execution_id} endpoint
    - Fetch forecast results from database
    - Return forecasts with confidence scores
    - _Requirements: 2.2.1, 2.2.2_

  - [ ]* 5.4 Write unit tests for demand forecasting
    - Test ARIMA model with synthetic time series data
    - Test confidence score calculation
    - Test forecast for 30/60/90 day periods
    - Test handling of insufficient historical data
    - _Requirements: 2.2.1, 2.2.2_

- [ ] 6. ML Service - Inventory risk detection (Python FastAPI)
  - [ ] 6.1 Implement inventory classification logic
    - Calculate sales velocity (units per day) for each product
    - Calculate days of supply (current_stock / avg_daily_sales)
    - Calculate sales trend (growth rate over last 30 days)
    - Classify risks: slow-moving, overstock, stockout
    - Assign severity levels: high, medium, low
    - _Requirements: 2.3.1, 2.3.2, 2.3.3_

  - [ ] 6.2 Create inventory risk API endpoint
    - Create POST /api/analysis/inventory-risk endpoint
    - Fetch sales and inventory data for user's products
    - Apply classification logic to identify risks
    - Store recommendations in recommendations table
    - Return risk analysis with severity and recommendations
    - _Requirements: 2.3.1, 2.3.2, 2.3.3, 2.3.5_

  - [ ]* 6.3 Write unit tests for inventory classification
    - Test slow-moving detection (velocity < 1, days_of_supply > 90)
    - Test overstock detection (days_of_supply > 120)
    - Test stockout risk detection (days_of_supply < 14, trend > 0)
    - Test severity level assignment
    - _Requirements: 2.3.1, 2.3.2, 2.3.3_

- [ ] 7. ML Service - Pricing optimization (Python FastAPI)
  - [ ] 7.1 Implement price elasticity estimation
    - Calculate historical price-demand correlation
    - Use linear regression on log-transformed data
    - Default elasticity: -1.5 if insufficient data
    - _Requirements: 2.4.1_

  - [ ] 7.2 Implement pricing optimization function
    - Calculate optimal price using elasticity formula
    - Estimate demand change based on price change
    - Calculate projected revenue impact
    - Return recommended price and estimated impact
    - _Requirements: 2.4.1, 2.4.2, 2.4.3_

  - [ ] 7.3 Create pricing optimization API endpoint
    - Create POST /api/analysis/pricing endpoint
    - Fetch sales and pricing history for specified products
    - Call pricing optimization function
    - Store recommendations in recommendations table
    - Return pricing recommendations with revenue impact
    - _Requirements: 2.4.1, 2.4.2, 2.4.3_

  - [ ]* 7.4 Write unit tests for pricing optimization
    - Test elasticity calculation with sample data
    - Test optimal price calculation
    - Test revenue impact estimation
    - Test handling of products with limited price history
    - _Requirements: 2.4.1, 2.4.2, 2.4.3_

- [ ] 8. Checkpoint - Ensure ML services work independently
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 9. LLaMA 4 integration for explainable AI
  - [ ] 9.1 Implement LLaMA 4 API client
    - Create client for LLaMA 4 API (or local inference)
    - Implement prompt template for business explanations
    - Add retry logic and error handling
    - _Requirements: 2.6.3_

  - [ ] 9.2 Create explanation generation function
    - Accept recommendation context (product, metric, action, data)
    - Format prompt with context using template
    - Call LLaMA 4 API to generate explanation
    - Validate explanation quality (50-150 words, includes data points)
    - Return natural language explanation
    - _Requirements: 2.6.1, 2.6.2, 2.6.3_

  - [ ] 9.3 Integrate explanations into ML endpoints
    - Update forecast endpoint to generate explanations
    - Update inventory risk endpoint to generate explanations
    - Update pricing endpoint to generate explanations
    - Store explanations in recommendations table
    - _Requirements: 2.6.1, 2.6.2, 2.6.5_

  - [ ]* 9.4 Write unit tests for explanation generation
    - Test prompt formatting with various contexts
    - Test explanation quality validation
    - Test error handling for API failures
    - _Requirements: 2.6.1, 2.6.2, 2.6.3_

- [ ] 10. Agent Service - Multi-agent orchestration (Node.js)
  - [ ] 10.1 Implement agent message protocol
    - Define message schema (message_id, timestamp, from/to agent, payload)
    - Create message validation function
    - Implement message logging to MongoDB
    - _Requirements: 2.5.4, 2.5.5_

  - [ ] 10.2 Implement Planner Agent
    - Create Planner Agent class
    - Implement task decomposition using LLaMA 4
    - Generate execution plan with task dependencies
    - Return structured task list (JSON format)
    - _Requirements: 2.5.1_

  - [ ] 10.3 Implement Retrieval Agent
    - Create Retrieval Agent class
    - Implement SQL query builder for PostgreSQL
    - Implement MongoDB query interface
    - Add data aggregation and filtering by date range, product, user
    - _Requirements: 2.5.2_

  - [ ] 10.4 Implement Executor Agent
    - Create Executor Agent class
    - Integrate with ML Service endpoints (forecast, inventory, pricing)
    - Handle ML task execution and result aggregation
    - _Requirements: 2.5.3_

  - [ ] 10.5 Implement agent orchestrator
    - Create orchestrator to coordinate agent communication
    - Implement task execution flow (Planner → Retrieval → Executor)
    - Handle task dependencies and sequential execution
    - Aggregate results and return to API Gateway
    - Log all agent interactions to MongoDB
    - _Requirements: 2.5.1, 2.5.2, 2.5.3, 2.5.4, 2.5.5_

  - [ ]* 10.6 Write unit tests for agent orchestration
    - Test message protocol validation
    - Test Planner Agent task decomposition
    - Test Retrieval Agent data fetching
    - Test Executor Agent ML task execution
    - Test end-to-end orchestration flow
    - _Requirements: 2.5.1, 2.5.2, 2.5.3, 2.5.4, 2.5.5_

- [ ] 11. Dashboard and recommendations API (Python FastAPI)
  - [ ] 11.1 Create dashboard summary endpoint
    - Create GET /api/dashboard/summary endpoint
    - Calculate total products, sales, active recommendations, alerts
    - Calculate forecast accuracy metric
    - Return summary statistics
    - _Requirements: 2.7.3_

  - [ ] 11.2 Create dashboard metrics endpoint
    - Create GET /api/dashboard/metrics endpoint
    - Accept period parameter (30/60/90 days)
    - Fetch sales trend data, top products, inventory health
    - Return time series and aggregated metrics
    - _Requirements: 2.7.1, 2.7.2, 2.7.3_

  - [ ] 11.3 Create recommendations listing endpoint
    - Create GET /api/recommendations endpoint
    - Support filtering by type (pricing, inventory, forecast) and status
    - Return paginated recommendations with explanations
    - _Requirements: 2.4.5_

  - [ ] 11.4 Create recommendation update endpoint
    - Create PUT /api/recommendations/{recommendation_id} endpoint
    - Allow status updates (accepted, rejected)
    - Update recommendation status in database
    - _Requirements: 2.4.5_

  - [ ]* 11.5 Write unit tests for dashboard APIs
    - Test summary calculation with sample data
    - Test metrics aggregation for different periods
    - Test recommendation filtering and pagination
    - Test recommendation status updates
    - _Requirements: 2.7.1, 2.7.2, 2.7.3, 2.4.5_

- [ ] 12. Checkpoint - Ensure all backend services are integrated
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 13. Frontend - Angular dashboard setup
  - [ ] 13.1 Create Angular project structure
    - Initialize Angular project with routing
    - Set up lazy loading for feature modules
    - Configure Chart.js library
    - Create shared services (API, Auth)
    - _Requirements: 2.7.4_

  - [ ] 13.2 Implement authentication components
    - Create login component with form validation
    - Create registration component with form validation
    - Implement AuthService for JWT token management
    - Implement AuthGuard for route protection
    - Store JWT token in localStorage
    - _Requirements: 2.8.1_

  - [ ] 13.3 Implement API service
    - Create ApiService with HTTP interceptor for JWT
    - Implement methods for all backend endpoints
    - Add error handling and retry logic
    - _Requirements: 2.8.2_

- [ ] 14. Frontend - Dashboard components
  - [ ] 14.1 Create dashboard layout component
    - Implement responsive grid layout
    - Create header with user menu and logout
    - Create navigation structure
    - _Requirements: 2.7.1, 2.7.4_

  - [ ] 14.2 Create metric card component
    - Display single metric with icon and value
    - Add color-coded status indicators
    - Implement click-to-drill-down functionality
    - _Requirements: 2.7.2, 2.7.3_

  - [ ] 14.3 Create chart component wrapper
    - Wrap Chart.js with Angular component
    - Support line, bar, and donut chart types
    - Implement responsive design
    - Add interactive time range selection
    - _Requirements: 2.7.1_

  - [ ] 14.4 Create recommendation card component
    - Display recommendation details (type, current/recommended values)
    - Show explanation text with expand/collapse
    - Add Accept/Reject buttons
    - Implement status update on button click
    - _Requirements: 2.4.5, 2.6.1_

  - [ ] 14.5 Implement dashboard main view
    - Fetch and display summary metrics (total sales, recommendations, alerts)
    - Display sales trend chart (30 days)
    - Display inventory health donut chart
    - Display top recommendations list
    - Implement real-time updates when analysis completes
    - _Requirements: 2.7.1, 2.7.2, 2.7.3, 2.7.5_

