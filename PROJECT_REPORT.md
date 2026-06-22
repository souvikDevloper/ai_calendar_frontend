# CreditBridge AI - Complete Project Report

## Executive Summary

CreditBridge AI is a full-stack credit underwriting application designed for thin-file borrowers in India. It demonstrates an AI-powered lending platform that uses alternative data sources, behavioral assessment, and consented financial data to generate credit scores and recommendations for micro, small, and medium enterprises (MSMEs) and individual borrowers.

---

## 1. Problem Statement

### 1.1 The Challenge

Traditional credit scoring in India relies heavily on:
- Credit bureau data (CIBIL, Equifax, etc.)
- Formal banking history
- Collateral-based lending

### 1.2 The Gap

- **Thin-file borrowers**: Millions of MSMEs and individuals lack formal credit history
- **Informal economy**: Large portion of economic activity occurs in cash/ informal sectors
- **Data exclusion**: Traditional models exclude borrowers with no bureau scores
- **Language barriers**: Most underwriting tools are English-only, excluding vernacular speakers

### 1.3 The Solution

CreditBridge AI addresses these challenges by:
- Using **alternative data sources** (UPI transactions, phone bills, e-commerce behavior, utility payments)
- Implementing **behavioral assessment** in vernacular languages (Hindi/English)
- Enabling **consent-driven data fetching** through Account Aggregators
- Providing **explainable AI** with SHAP-like factor attribution
- Supporting **officer-in-the-loop** decision making with audit trails

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Frontend (React)                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Borrower PWA │  │ Application  │  │  UCO Officer Portal  │  │
│  │              │  │   Console    │  │                      │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ HTTP/REST API
                              │
┌─────────────────────────────────────────────────────────────────┐
│                      Backend (FastAPI)                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   API Layer  │  │  Scoring     │  │   Data Store (SQLite)│  │
│  │   (main.py)  │  │  Engine      │  │                      │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Technology Stack

#### Backend
- **Framework**: FastAPI 0.124.2
- **Server**: Uvicorn 0.38.0
- **Database**: SQLite (with PostgreSQL planned)
- **Language**: Python 3.11+
- **Testing**: Pytest 9.0.2, HTTPX 0.28.1

#### Frontend
- **Framework**: React (latest)
- **Build Tool**: Vite (latest)
- **UI Library**: Lucide React (icons)
- **Charts**: Recharts (latest)
- **Styling**: TailwindCSS 3.4.17
- **Internationalization**: i18next, react-i18next
- **Testing**: Vitest, Testing Library

### 2.3 Data Flow Architecture

```
Borrower Onboarding
       ↓
Profile Creation (Persona-based or Manual)
       ↓
Consent Management (8 data sources)
       ↓
Source Link Verification (AA, UPI, GSTIN, etc.)
       ↓
Behavioral Assessment (5 questions, Hindi/English)
       ↓
Signal Fetch (Simulated data retrieval)
       ↓
AI Scoring Engine (XGBoost-style + SHAP attribution)
       ↓
Officer Review (UCO Dashboard)
       ↓
Decision & Audit Trail
```

---

## 3. Backend Architecture & Implementation

### 3.1 Project Structure

```
backend/
├── app/
│   ├── __init__.py
│   ├── main.py           # FastAPI application & endpoints
│   ├── models.py         # Pydantic data models
│   ├── scoring.py        # AI scoring engine
│   ├── seed.py           # Demo data (personas, questions)
│   └── store.py          # SQLite database operations
├── requirements.txt
└── creditbridge_demo.db  # SQLite database
```

### 3.2 Database Schema (SQLite)

#### borrowers Table
```sql
CREATE TABLE borrowers (
    id TEXT PRIMARY KEY,
    persona_id TEXT,
    profile_json TEXT NOT NULL,
    consents_json TEXT NOT NULL,
    source_links_json TEXT NOT NULL DEFAULT '{}',
    behavioral_json TEXT NOT NULL,
    signals_json TEXT NOT NULL,
    score_json TEXT NOT NULL,
    status TEXT NOT NULL,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);
```

#### decisions Table
```sql
CREATE TABLE decisions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    borrower_id TEXT NOT NULL,
    action TEXT NOT NULL,
    notes TEXT NOT NULL,
    sanctioned_amount INTEGER,
    officer_name TEXT NOT NULL,
    created_at TEXT NOT NULL
);
```

#### audit Table
```sql
CREATE TABLE audit (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    borrower_id TEXT NOT NULL,
    actor TEXT NOT NULL,
    event TEXT NOT NULL,
    details_json TEXT NOT NULL,
    created_at TEXT NOT NULL
);
```

### 3.3 API Endpoints

#### Health & Demo Management
- `GET /api/health` - Health check
- `POST /api/demo/reset` - Reset database and load seed data
- `GET /api/demo/personas` - Get available personas and questions

#### Borrower Management
- `POST /api/borrowers` - Create new borrower
- `POST /api/borrowers/{borrower_id}/consents` - Update consent preferences
- `POST /api/borrowers/{borrower_id}/source-links` - Verify source links
- `POST /api/borrowers/{borrower_id}/behavioral` - Submit behavioral assessment
- `POST /api/borrowers/{borrower_id}/fetch-signals` - Fetch alternative data signals
- `POST /api/borrowers/{borrower_id}/score` - Generate credit score

#### Officer Portal
- `GET /api/officer/cases` - List all scored cases
- `GET /api/officer/cases/{borrower_id}` - Get case details
- `POST /api/officer/cases/{borrower_id}/decision` - Submit officer decision

#### Audit
- `GET /api/audit/{borrower_id}` - Get audit log for borrower

---

## 4. Scoring Engine (Core Logic)

### 4.1 Scoring Model Overview

The scoring engine uses a **deterministic XGBoost + neural scorer simulation** with SHAP-like factor attribution. It combines:

1. **Six Factor Scores** (weighted):
   - Cash Flow Strength (25%)
   - Payment Discipline (20%)
   - Business Health (18%)
   - Stability (14%)
   - Intent to Repay (13%)
   - Trust Flags (10%)

2. **Alternative Data Sources** (8 consented sources):
   - Bank account statements
   - Phone bill payment history
   - E-commerce purchase behavior
   - UPI transactions
   - GSTN / business data
   - Utility bill payments
   - Merchant ratings
   - Credit bureau-lite data

### 4.2 Factor Calculation Logic

#### Cash Flow Strength (25% weight)
```python
cash_flow_strength = (
    pct(inflow_regularity) * 0.35 +
    score_income(avg_monthly_inflow, stated_income) * 0.25 +
    pct(balance_stability) * 0.2 +
    affordability_score(income, existing_emi) * 0.2
)
```

**Components:**
- **Inflow Regularity**: Consistency of monthly income (target ≥75%)
- **Income Verification**: Ratio of bank inflow to stated income (target ≥100%)
- **Balance Stability**: Account balance consistency over time
- **Affordability**: EMI burden as percentage of income (target ≤25%)

#### Payment Discipline (20% weight)
```python
payment_discipline = (
    pct(phone_bill_on_time_ratio) * 0.24 +
    pct(utility_on_time_ratio) * 0.26 +
    pct(upi_consistency) * 0.27 +
    pct(recurring_payment_ratio) * 0.23
)
```

**Components:**
- Phone bill on-time payment ratio
- Utility bill payment discipline
- UPI transaction consistency
- Recurring payment regularity

#### Business Health (18% weight)
```python
business_health = (
    pct(sales_stability) * 0.26 +
    vintage_score(business_vintage_months) * 0.2 +
    pct(merchant_inflow_ratio) * 0.22 +
    merchant_rating_score(rating, reviews, complaints) * 0.2 +
    (92 if gst_available else 62) * 0.12
)
```

**Components:**
- Sales stability from GSTN/proxy data
- Business vintage (months in operation)
- Merchant inflow ratio
- Merchant ratings and reviews
- GST registration status

#### Stability (14% weight)
```python
stability = (
    clamp(geo_stability_months / 36 * 100) * 0.3 +
    pct(device_consistency) * 0.25 +
    pct(identity_match) * 0.25 +
    pct(address_stability) * 0.2
)
```

**Components:**
- Geographic stability (months at same location)
- Device consistency (same device usage)
- Identity match score (eKYC verification)
- Address stability

#### Intent to Repay (13% weight)
```python
intent_to_repay = behavioral_score(answers)
```

**Components:**
- Behavioral assessment answers (5 questions)
- Each answer scored 0.0-1.0
- Average score converted to 0-100 scale

**Behavioral Questions:**
1. Income irregularity management
2. EMI payment priority
3. Business record-keeping method
4. Loan usage purpose
5. Emergency buffer maintenance

#### Trust Flags (10% weight)
```python
trust_flags = (
    96 - fraud_flags * 38 - anomaly_count * 16
) + (consent_coverage - 0.6) * 12
trust_flags = trust_flags * 0.78 + ecommerce_trust * 0.22
```

**Components:**
- Fraud flags detected
- Transaction anomalies
- Consent coverage (percentage of sources consented)
- E-commerce trust signals

### 4.3 Final Score Calculation

```python
weighted_score = sum(subscore[key] * weight for key, weight in WEIGHTS.items())
missing_penalty = max(0, 4 - consented_sources_count) * 7
risk_penalty = anomaly_count * 12 + fraud_flags * 36
final_score = clamp(300 + weighted_score * 5.5 - missing_penalty - risk_penalty, 300, 900)
```

**Score Range:** 300-900
**Base Score:** 300
**Maximum Bonus:** ~600 points from weighted factors

### 4.4 Risk Band Classification

| Score Range | Risk Band | Action |
|-------------|----------|--------|
| 750-900 | Low Risk | Straight-through approval |
| 650-749 | Medium Risk | Approve with conditions |
| 580-649 | Manual Review | Mandatory officer review |
| <580 | High Risk | Decline or committee exception |

### 4.5 Confidence Calculation

```python
confidence = clamp(54 + consent_coverage * 26 + (weighted_score - 60) * 0.35, 35, 96)
```

**Range:** 35%-96%
**Factors:**
- Base confidence: 54%
- Consent coverage bonus: up to 26%
- Score strength bonus: up to 16%

### 4.6 Default Probability

```python
base_pd = clamp((900 - score) / 600 * 18 + 1.2, 1.0, 26.0)
high_flag_add = high_severity_flags * 3.5
medium_flag_add = medium_severity_flags * 1.4
confidence_discount = max(0, 80 - confidence) * 0.12
final_pd = clamp(base_pd + high_flag_add + medium_flag_add + confidence_discount, 1.0, 35.0)
```

**Range:** 1.0%-35.0%

### 4.7 Recommended Limit Calculation

```python
income_multiplier = {
    "Low Risk": 2.4,
    "Medium Risk": 1.8,
    "Manual Review": 1.15
}.get(risk_band, 0)
recommended_limit = ceil(min(income * income_multiplier, requested_amount) / 10000) * 10000
```

**Logic:**
- Low Risk: Up to 2.4x monthly income
- Medium Risk: Up to 1.8x monthly income
- Manual Review: Up to 1.15x monthly income
- High Risk: 0 (not recommended)

### 4.8 Evidence Metrics

The system generates 13 evidence metrics with:
- **Category**: Cash flow, Affordability, Payment behavior, etc.
- **Metric**: Specific measurement (e.g., "Average monthly inflow")
- **Value**: Actual value from data
- **Benchmark**: Target threshold for strong performance
- **Source**: Data source (e.g., "Bank account statements")
- **Score**: Calculated score (0-100)
- **Status**: strong, watch, risk, or missing
- **Detail**: Explanation of metric significance

### 4.9 Review Flags

Automated flags generated for officer attention:

**High Severity:**
- Low model confidence (<70%)
- Weak factor scores (<55)
- Transaction anomalies detected

**Medium Severity:**
- Weak factor scores (55-62)
- Missing consented data
- Affordability pressure (high EMI burden)

**Low Severity:**
- No hard review blockers

### 4.10 Officer Recommendations

Based on risk band, confidence, and flags:
- **Primary Action**: Route to manual review / Approve after verification / Approve within cap
- **Officer Guidance**: Specific instructions for the officer
- **Governance**: Reminder that final approval remains with UCO

---

## 5. Frontend Architecture & Implementation

### 5.1 Project Structure

```
frontend/
├── src/
│   ├── App.jsx           # Main React application
│   ├── api.js            # API client
│   ├── demoLogic.js      # Demo data and utilities
│   ├── i18n.js           # Internationalization config
│   ├── main.jsx          # React entry point
│   └── styles.css        # Global styles
├── index.html
├── package.json
├── tailwind.config.js
└── postcss.config.js
```

### 5.2 User Interfaces

#### 5.2.1 Application Console
- **Purpose**: Demo management and persona selection
- **Features**:
  - Persona cards (Ramesh, Lata, Aman, Meena)
  - Generate case button for each persona
  - Quick stats display
  - Reset demo functionality

#### 5.2.2 Borrower PWA
- **Purpose**: Borrower onboarding and application flow
- **Steps**:
  1. **Login**: Phone number entry with OTP simulation
  2. **Mobile Verification**: OTP entry
  3. **eKYC**: Identity verification simulation
  4. **Consent**: 8 data source toggles with explanations
  5. **Profile**: Business and personal information
  6. **Behavioral Assessment**: 5 vernacular questions
  7. **Source Linking**: Verify identifiers for consented sources
  8. **Data Fetch**: Simulated data retrieval with loading states
  9. **Scoring**: AI score generation
  10. **Result**: Score display with recommendations

#### 5.2.3 UCO Officer Portal
- **Purpose**: Loan officer dashboard for case review
- **Features**:
  - **Portfolio KPIs**: Total cases, approval rate, average score, etc.
  - **Case Queue**: Searchable list of scored cases
  - **Risk Filters**: Filter by risk band
  - **Policy Thresholds**: Score band reference table
  - **Case Workspace**: Detailed case view with:
    - Six factor scores with gauges
    - Evidence metrics table
    - Review flags
    - Reason codes
    - Default probability
    - Recommended limit
    - Pricing band
    - Consent ledger
    - Audit trail
  - **Action Center**: Officer decision controls
    - Approve
    - Approve with conditions
    - Request documents
    - Send field verification
    - Escalate to credit committee
    - Decline
  - **Load Sample Portfolio**: Quick demo data loading

### 5.3 Key Frontend Components

#### API Client (`api.js`)
```javascript
export const api = {
  health: () => fetchJson("/api/health"),
  reset: () => fetchJson("/api/demo/reset", { method: "POST" }),
  personas: () => fetchJson("/api/demo/personas"),
  createBorrower: (payload) => fetchJson("/api/borrowers", { method: "POST", body: JSON.stringify(payload) }),
  saveConsents: (borrowerId, consents) => fetchJson(`/api/borrowers/${borrowerId}/consents`, { method: "POST", body: JSON.stringify({ consents }) }),
  saveSourceLinks: (borrowerId, sourceLinks) => fetchJson(`/api/borrowers/${borrowerId}/source-links`, { method: "POST", body: JSON.stringify({ source_links: sourceLinks }) }),
  saveBehavioral: (borrowerId, answers) => fetchJson(`/api/borrowers/${borrowerId}/behavioral`, { method: "POST", body: JSON.stringify({ answers }) }),
  fetchSignals: (borrowerId) => fetchJson(`/api/borrowers/${borrowerId}/fetch-signals`, { method: "POST" }),
  score: (borrowerId) => fetchJson(`/api/borrowers/${borrowerId}/score`, { method: "POST" }),
  cases: () => fetchJson("/api/officer/cases"),
  caseDetail: (borrowerId) => fetchJson(`/api/officer/cases/${borrowerId}`),
  decide: (borrowerId, payload) => fetchJson(`/api/officer/cases/${borrowerId}/decision`, { method: "POST", body: JSON.stringify(payload) }),
  audit: (borrowerId) => fetchJson(`/api/audit/${borrowerId}`),
};
```

#### Demo Logic (`demoLogic.js`)
- Consent keys and labels
- Source link field definitions
- Default source links per persona
- Missing source link validation
- INR formatting
- Risk band styling
- Default answer generation
- Missing data consent handling

#### Internationalization (`i18n.js`)
- Hindi and English translations
- Step labels
- UI text
- Behavioral question translations

### 5.4 State Management

The React app uses:
- **useState**: For local component state
- **useEffect**: For API calls and side effects
- **useMemo**: For computed values
- **useRef**: For DOM references

No external state management library (Redux, etc.) is used - state is managed at component level.

### 5.5 Styling

- **Framework**: TailwindCSS 3.4.17
- **Theme**: Custom UCO-inspired colors (navy, saffron, green)
- **Dark Mode**: Supported with theme toggle
- **Responsive**: Mobile-first design
- **Custom Colors**:
  - `bridge-green`: UCO green
  - `bridge-amber`: Warning color
  - `bridge-red`: Error color
  - `bridge-navy`: UCO navy

---

## 6. Data Models

### 6.1 Borrower Profile

```python
{
    "full_name": str,
    "phone": str,
    "language": "en" | "hi",
    "business_type": str,
    "monthly_income_band": str,
    "monthly_income_mid": float,
    "business_vintage": str,
    "location": str,
    "gender": str,
    "requested_amount": int,
    "headline": str
}
```

### 6.2 Consent Data

```python
{
    "bank_account": bool,
    "phone_bills": bool,
    "ecommerce_behavior": bool,
    "upi_transactions": bool,
    "gstn_data": bool,
    "utility_bills": bool,
    "merchant_ratings": bool,
    "credit_bureau": bool
}
```

### 6.3 Source Links

```python
{
    "bank_account": {
        "identifier": str,
        "provider": str,
        "verified": bool,
        "auth_method": str
    },
    ... (for each consented source)
}
```

### 6.4 Behavioral Answers

```python
[
    {
        "question_id": str,
        "answer_id": str,
        "score": float  # 0.0-1.0
    },
    ... (5 answers)
]
```

### 6.5 Signals Data

```python
{
    "status": "fetched" | "not_started",
    "sources": [
        {"key": str, "status": "fetched" | "not_consented"}
    ],
    "features": {
        # Cash flow
        "inflow_regularity": float,
        "balance_stability": float,
        "avg_monthly_inflow": float,
        "existing_emi": float,
        
        # Payment discipline
        "phone_bill_on_time_ratio": float,
        "mobile_recharge_consistency": float,
        "utility_on_time_ratio": float,
        "recurring_payment_ratio": float,
        
        # E-commerce
        "ecommerce_order_regularity": float,
        "ecommerce_dispute_ratio": float,
        "ecommerce_essential_purchase_ratio": float,
        
        # UPI
        "upi_consistency": float,
        "merchant_inflow_ratio": float,
        
        # Business
        "sales_stability": float,
        "gst_available": int,
        "business_vintage_months": int,
        
        # Merchant
        "merchant_rating_avg": float,
        "merchant_review_count": int,
        "merchant_complaint_ratio": float,
        
        # Bureau
        "bureau_score": float | None,
        "credit_history_months": int,
        
        # Stability
        "geo_stability_months": int,
        "device_consistency": float,
        "identity_match": float,
        "address_stability": float,
        
        # Trust
        "fraud_flags": int,
        "anomaly_count": int
    }
}
```

### 6.6 Score Result

```python
{
    "score": int,  # 300-900
    "max_score": 900,
    "risk_band": "Low Risk" | "Medium Risk" | "Manual Review" | "High Risk",
    "confidence": int,  # 35-96
    "subscores": {
        "cash_flow_strength": float,
        "payment_discipline": float,
        "business_health": float,
        "stability": float,
        "intent_to_repay": float,
        "trust_flags": float
    },
    "factors": [
        {
            "key": str,
            "label": str,
            "score": float,
            "weight": float,
            "contribution": float
        }
    ],
    "reason_codes": [
        {
            "type": "positive" | "watch" | "data",
            "factor": str,
            "message": str
        }
    ],
    "evidence_metrics": [...],  # 13 metrics
    "review_flags": [
        {
            "severity": "high" | "medium" | "low",
            "title": str,
            "message": str,
            "suggestion": str
        }
    ],
    "officer_recommendations": [
        {
            "priority": str,
            "text": str
        }
    ],
    "privacy_controls": [str],
    "model_summary": {
        "consented_financial_data_weight": float,
        "behavioral_assessment_weight": float,
        "governance_trust_weight": float,
        "message": str
    },
    "recommended_limit": int,
    "default_probability": float,
    "pricing_band": str,
    "manual_review": bool,
    "model_version": str,
    "method": str,
    "questions": [...]  # Behavioral questions
}
```

### 6.7 Decision Payload

```python
{
    "action": "approve" | "conditional_approve" | "documents_requested" | 
              "field_verification" | "committee_escalation" | "review" | "decline",
    "officer_notes": str,
    "sanctioned_amount": int | None,
    "officer_name": str
}
```

---

## 7. Demo Personas

### 7.1 Ramesh Kumar (Ideal Borrower)
- **Profile**: Grocery store owner, 2-3 years vintage
- **Income**: INR 40,000-60,000/month
- **Location**: Barasat, West Bengal
- **Score**: ~782/900 (Low Risk)
- **Strengths**: Strong cash flow, excellent payment discipline, GST registered
- **Recommended Limit**: INR 1,20,000

### 7.2 Lata Devi (Strong Alternative Data)
- **Profile**: Tailoring unit owner, 3-5 years vintage
- **Income**: INR 25,000-40,000/month
- **Location**: Krishnanagar, West Bengal
- **Score**: ~745/900 (Low Risk)
- **Strengths**: No CIBIL history but strong UPI and bill discipline
- **Recommended Limit**: INR 78,000

### 7.3 Aman Ali (Borderline Case)
- **Profile**: Mobile repair kiosk, 6-12 months vintage
- **Income**: INR 20,000-35,000/month
- **Location**: Asansol, West Bengal
- **Score**: ~612/900 (Manual Review)
- **Challenges**: Unstable cash flow, new business, some anomalies
- **Recommended Limit**: INR 49,000 (with conditions)

### 7.4 Meena Shaw (Limited Data)
- **Profile**: Street food stall, 1-2 years vintage
- **Income**: INR 18,000-30,000/month
- **Location**: Howrah, West Bengal
- **Score**: ~595/900 (Manual Review)
- **Challenges**: Limited consented data (no e-commerce, UPI, GSTN)
- **Recommended Limit**: INR 27,600 (with conditions)

---

## 8. Complete User Flow

### 8.1 Borrower Flow

1. **Application Console**: Select persona (e.g., Ramesh Kumar)
2. **Generate Case**: Click "Generate Case" to create borrower
3. **Borrower PWA**:
   - Enter phone number
   - Simulate OTP verification
   - Complete eKYC simulation
   - Review and consent to 8 data sources
   - Confirm profile information
   - Answer 5 behavioral questions (Hindi/English)
   - Link and verify source identifiers (bank account, UPI ID, GSTIN, etc.)
   - Trigger data fetch (simulated)
   - View AI score and recommendations

### 8.2 Officer Flow

1. **UCO Officer Portal**: Open separate dashboard
2. **Load Sample Portfolio**: Click to populate with demo cases
3. **Review Portfolio**: View KPIs and case queue
4. **Select Case**: Click on a case to view details
5. **Analyze Case**:
   - Review six factor scores
   - Examine evidence metrics
   - Check review flags
   - Read reason codes
   - Verify consent ledger
   - Review audit trail
6. **Make Decision**:
   - Select action from action center
   - Enter officer notes
   - Specify sanctioned amount (if approving)
   - Confirm action
7. **Audit**: Decision is logged in audit trail

---

## 9. Security & Privacy Features

### 9.1 Consent Management
- **Explicit consent**: Each data source requires explicit borrower consent
- **Consent ledger**: All consent actions are audit logged
- **Revocation support**: Consent can be revoked (not implemented in demo)

### 9.2 Data Protection
- **AES-256 encryption**: Mentioned in reference architecture
- **Tokenized identifiers**: Sensitive data is tokenized
- **Role-based access**: Officer portal restricted to authorized users

### 9.3 Audit Trail
- **Complete logging**: All actions are logged with timestamp
- **Actor tracking**: Records who performed each action (borrower, system, officer)
- **Event details**: JSON details for each audit event
- **Immutable records**: Audit log cannot be modified

### 9.4 Governance
- **Officer-in-the-loop**: Final approval always requires human officer
- **Model versioning**: Model version is tracked for each score
- **Explainability**: SHAP-like factor attribution for transparency
- **Policy thresholds**: Clear score bands and actions defined

---

## 10. Key Features & Innovations

### 10.1 Vernacular Support
- Hindi and English interface
- Behavioral questions in both languages
- Localized UI elements

### 10.2 Alternative Data Integration
- 8 alternative data sources
- Account Aggregator (AA) simulation
- UPI transaction analysis
- Digital payment discipline tracking

### 10.3 Behavioral Assessment
- 5 questions designed for MSME context
- Scores intent and planning discipline
- Weighted appropriately (13%) - support signal only

### 10.4 Explainable AI
- Six factor scores with individual weights
- SHAP-like factor contributions
- Evidence metrics with source attribution
- Reason codes for decision explanation

### 10.5 Officer Empowerment
- Rich case workspace with all relevant data
- Action center with multiple decision options
- Review flags for risk identification
- Audit trail for compliance

### 10.6 Progressive Web App (PWA)
- Mobile-first design
- Responsive layout
- App-like experience

---

## 11. Testing

### 11.1 Backend Tests
- **Framework**: Pytest
- **Coverage**: API endpoints, scoring logic
- **Location**: `backend/tests/`

### 11.2 Frontend Tests
- **Framework**: Vitest + Testing Library
- **Coverage**: Component logic, demo utilities
- **Location**: `frontend/src/demoLogic.test.js`

### 11.3 Running Tests

**Backend:**
```powershell
cd backend
$env:PYTHONPATH = (Get-Location).Path
python -m pytest -q
```

**Frontend:**
```powershell
cd frontend
npm test
npm run build
```

---

## 12. Deployment & Running

### 12.1 Prerequisites
- Python 3.11+ or 3.12+
- Node.js 20+ with npm
- PowerShell on Windows

### 12.2 Quick Start

```powershell
.\start-dev.ps1
```

This script:
1. Installs Python dependencies
2. Starts FastAPI backend on port 8000
3. Installs npm dependencies
4. Starts Vite dev server on port 5173

### 12.3 Access Points

- **Borrower App**: http://127.0.0.1:5173
- **UCO Officer Portal**: http://127.0.0.1:5173/uco-dashboard
- **Backend API Docs**: http://127.0.0.1:8000/docs

### 12.4 Manual Start

**Backend:**
```powershell
cd backend
python -m pip install -r requirements.txt
$env:PYTHONPATH = (Get-Location).Path
python -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

**Frontend:**
```powershell
cd frontend
npm install
npm run dev
```

---

## 13. Future Enhancements

### 13.1 Planned Features
- PostgreSQL support for production deployment
- Real Account Aggregator integration
- Live bureau data integration
- Advanced fraud detection models
- Multi-language support (more Indian languages)
- Mobile app (React Native)
- Advanced analytics dashboard

### 13.2 Scalability Considerations
- Horizontal scaling with load balancers
- Caching layer for frequently accessed data
- Async processing for score generation
- Message queue for background jobs

---

## 14. Conclusion

CreditBridge AI demonstrates a modern, AI-powered credit underwriting platform that addresses the challenges of lending to thin-file borrowers in India. By combining alternative data sources, behavioral assessment, and explainable AI with officer-in-the-loop decision making, it provides a comprehensive solution for financial institutions to expand credit access while maintaining risk management and regulatory compliance.

The system's architecture is modular, scalable, and built with modern best practices, making it suitable for both demonstration and production deployment with appropriate enhancements.

---

## Appendix A: File Reference

### Backend Files
- `backend/app/main.py` - FastAPI application (261 lines)
- `backend/app/models.py` - Pydantic models (62 lines)
- `backend/app/scoring.py` - Scoring engine (558 lines)
- `backend/app/seed.py` - Demo data (252 lines)
- `backend/app/store.py` - Database operations (207 lines)
- `backend/requirements.txt` - Python dependencies

### Frontend Files
- `frontend/src/App.jsx` - Main React app (2603 lines)
- `frontend/src/api.js` - API client (62 lines)
- `frontend/src/demoLogic.js` - Demo utilities (187 lines)
- `frontend/src/i18n.js` - i18n config
- `frontend/src/main.jsx` - Entry point
- `frontend/src/styles.css` - Global styles
- `frontend/package.json` - Node dependencies

### Configuration
- `start-dev.ps1` - Development startup script
- `frontend/tailwind.config.js` - Tailwind config
- `frontend/postcss.config.js` - PostCSS config

---

## Appendix B: API Reference

### POST /api/borrowers
Create a new borrower

**Request:**
```json
{
  "persona_id": "ramesh",
  "phone": "9876543210",
  "language": "hi",
  "full_name": "Ramesh Kumar",
  "business_type": "Grocery Store",
  "monthly_income_band": "INR 40,000 - INR 60,000",
  "business_vintage": "2 - 3 Years",
  "location": "West Bengal"
}
```

**Response:** Borrower object with ID

### POST /api/borrowers/{id}/consents
Update consent preferences

**Request:**
```json
{
  "consents": {
    "bank_account": true,
    "phone_bills": true,
    ...
  }
}
```

**Response:** Updated borrower object

### POST /api/borrowers/{id}/score
Generate credit score

**Response:** Complete score result with factors, evidence, flags, recommendations

---

*Report generated for CreditBridge AI v0.3.0*
