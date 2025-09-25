## Executive Summary

This report outlines a comprehensive test data strategy for the `CreditCheckService` application, focusing on its Mainframe and Distributed Application (MFDA) integration points. The analysis identified two primary integration types: an Apigee/API endpoint for receiving credit check requests and an AlloyDB (PostgreSQL) database integration for retrieving and updating credit score information. No MFT, Kafka, or Oracle integrations were found. The strategy provides detailed requirements for test data generation, management, and quality assurance across Integration, Regression, and End-to-End testing phases to ensure a robust and reliable service.

## Analysis

### 1. MFDA Integration Test Data Strategy

This section details the test data required for validating the identified integration points.

#### MFT Integration Test Data

No MFT (Managed File Transfer) components, processes, or configurations were identified in the codebase. Therefore, no MFT-specific test data is required for this application.

#### Apigee/API Integration Test Data

The application exposes a REST API endpoint at `/creditscore` for performing credit checks. Test data must cover valid payloads, error conditions, authentication, and performance scenarios.

**Request/Response Payloads:**

```json
### Account Inquiry API Test Data
**API Endpoint**: /creditscore
**Authentication**: API Key + OAuth 2.0 (Assumed for strategy)

**Valid Request Payload**:
{
  "SSN": "[REDACTED_SSN]",
  "FirstName": "John",
  "LastName": "Doe",
  "DOB": "12/29/1984"
}

**Valid Response Payload (200 OK)**:
{
  "FICOScore": 780,
  "Rating": "Good",
  "NoOfInquiries": 3
}

**Error Response Payload (404 Not Found)**:
{
  "errorCode": "ACC_NOT_FOUND",
  "errorMessage": "Account not found for the given SSN."
}
```

**API Test Data Scenarios**:

```markdown
### Authentication Test Data
**Valid API Keys**:
- DEV Environment: [REDACTED_DEV_API_KEY]
- TEST Environment: [REDACTED_TEST_API_KEY]

**Invalid API Keys** (for negative testing):
- Expired Key: [REDACTED_EXPIRED_API_KEY]
- Invalid Format Key: "invalid-key-format"
- Null/Empty Key

**OAuth Tokens**:
- Valid Token: [REDACTED_VALID_JWT_TOKEN]
- Expired Token: [REDACTED_EXPIRED_JWT_TOKEN]
- Token with insufficient scope

### Rate Limiting Test Data
**Normal Load**: 50 requests/minute
**High Load**: 500 requests/minute
**Burst Load**: 1000 requests/minute (to trigger rate limiting)
```

#### Kafka Integration Test Data

No Kafka or TIBCO EMS components were identified in the codebase. Therefore, no Kafka-specific test data is required.

#### AlloyDB Integration Test Data

The application connects to a PostgreSQL database (assumed to be AlloyDB in the MFDA context) to query and update a `creditscore` table.

**Database Test Datasets:**

```sql
### Credit Score Test Data (AlloyDB/PostgreSQL)
-- Datasets for various load scenarios
-- Small: 1,000 records
-- Medium: 50,000 records
-- Large: 500,000 records

-- Table structure inferred from `LookupDatabase.bwp`
CREATE TABLE public.creditscore (
    firstname VARCHAR(255),
    lastname VARCHAR(255),
    ssn VARCHAR(20) PRIMARY KEY,
    dateofBirth VARCHAR(20),
    ficoscore INT,
    rating VARCHAR(50),
    numofpulls INT
);

-- Sample records for functional testing
INSERT INTO public.creditscore (firstname, lastname, ssn, dateofBirth, ficoscore, rating, numofpulls) VALUES
('John', 'Doe', '[REDACTED_SSN_1]', '1984-12-29', 810, 'Excellent', 1),
('Jane', 'Smith', '[REDACTED_SSN_2]', '1990-05-15', 720, 'Good', 5),
('Peter', 'Jones', '[REDACTED_SSN_3]', '1975-02-10', 650, 'Fair', 12),
('Mary', 'Williams', '[REDACTED_SSN_4]', '1988-11-20', 550, 'Poor', 25),
('Sam', 'Brown', '[REDACTED_SSN_5]', '2000-01-01', 0, 'No Record', 0); -- Edge case
```

**Database Performance Test Data Generation**:

```sql
### Performance Testing Data Generation Procedure

-- This procedure generates a large volume of realistic test data for performance validation.
-- It should be run in a dedicated performance testing environment.

-- Generate 1 million credit score records
INSERT INTO public.creditscore (firstname, lastname, ssn, dateofBirth, ficoscore, rating, numofpulls)
SELECT
    'FirstName' || s.id,
    'LastName' || s.id,
    '999-' || LPAD((s.id % 100)::text, 2, '0') || '-' || LPAD((s.id % 10000)::text, 4, '0'),
    (CURRENT_DATE - (random() * 20000 + 6570) * '1 day'::interval)::date,
    (random() * 550 + 300)::int,
    CASE
        WHEN (random() * 550 + 300)::int > 800 THEN 'Excellent'
        WHEN (random() * 550 + 300)::int > 740 THEN 'Very Good'
        WHEN (random() * 550 + 300)::int > 670 THEN 'Good'
        WHEN (random() * 550 + 300)::int > 580 THEN 'Fair'
        ELSE 'Poor'
    END,
    (random() * 20)::int
FROM generate_series(1, 1000000) AS s(id);
```

#### Oracle Database Integration Test Data

No Oracle database components were identified in the codebase. The only detected database is PostgreSQL.

### 2. Regression and End-to-End Testing Data

**Business Process**: Credit Score Inquiry
This single workflow is the core of the application and serves for both regression and E2E testing.

**Required Test Data Components**:
1.  **API Request Data**: A set of JSON payloads with SSNs corresponding to various profiles in the test database.
2.  **Database State**: The `creditscore` table must be pre-populated with records matching the API request data.
3.  **Expected Outcome Data**: A mapping of input SSNs to expected API responses (`FICOScore`, `Rating`, `NoOfInquiries`) and expected database state changes (`numofpulls` incremented by 1).

**Test Data Variations**:
*   **Happy Path**: SSN exists, has a valid score, and a low number of inquiries.
*   **Non-Existent Record**: SSN does not exist in the database.
*   **High Inquiry Count**: SSN with a high `numofpulls` value to test business logic thresholds (if any).
*   **Boundary Score**: SSN with a FICO score on the boundary between two ratings (e.g., 669 for 'Fair' and 670 for 'Good').
*   **Data Volume for E2E Performance**: Simulate 100,000 credit check requests over a 1-hour period to validate system performance under load.

### 3. Environment-Specific Test Data Management

| Environment | Purpose | Data Volume | Data Refresh | Data Masking |
| :--- | :--- | :--- | :--- | :--- |
| **Development** | Unit & component testing | ~1,000 records | On-demand | Full PII Masking |
| **Test** | Integration & E2E testing | ~50,000 records | Weekly | Full PII Masking |
| **Staging** | UAT, Performance, Pre-prod | ~500,000 records | Bi-weekly | Full PII Masking |

**Environment-Specific Locations**:
*   **API Endpoint**:
    *   DEV: `http://creditcheck-dev.internal/creditscore`
    *   TEST: `https://api-test.company.com/creditcheck/v1/creditscore`
*   **Database Connection (BWCE.DB.URL)**:
    *   DEV: `jdbc:postgresql://dev-pg.internal:5432/credit_dev`
    *   TEST: `jdbc:postgresql://test-alloydb.internal:5432/credit_test`

### 4. Test Data Generation and Management Procedures

#### Automated Test Data Generation
A data generation utility should be created to populate environments with realistic and varied data.

```python
### Python script example for generating credit score data
import random
from faker import Faker

fake = Faker()

def generate_credit_data(num_records):
    """Generates a list of synthetic credit score records."""
    records = []
    for i in range(num_records):
        ssn = fake.ssn()
        ficoscore = random.randint(300, 850)
        rating = "Poor"
        if ficoscore > 800: rating = "Excellent"
        elif ficoscore > 740: rating = "Very Good"
        elif ficoscore > 670: rating = "Good"
        elif ficoscore > 580: rating = "Fair"

        record = {
            'firstname': fake.first_name(),
            'lastname': fake.last_name(),
            'ssn': ssn,
            'dateofBirth': fake.date_of_birth(minimum_age=18, maximum_age=90).strftime('%Y-%m-%d'),
            'ficoscore': ficoscore,
            'rating': rating,
            'numofpulls': random.randint(0, 30)
        }
        records.append(record)
    return records

# Usage:
# test_data = generate_credit_data(1000)
# Then, use a DB library to insert `test_data` into the target database.
```

#### Data Masking and Privacy Protection
Sensitive data like SSN, Name, and DOB must be masked in all non-production environments.

```sql
### Data Masking SQL Procedure
-- This procedure should be part of the data refresh process for non-production environments.

UPDATE public.creditscore
SET
    firstname = 'FirstName-' || ssn,
    lastname = 'LastName-' || ssn,
    ssn = '999-XX-' || RIGHT(ssn, 4), -- Mask SSN, keeping last 4 digits
    dateofBirth = '1900-01-01'
WHERE
    -- Condition to ensure this only runs in non-production environments
    current_database() NOT LIKE '%prod%';
```

### 5. Test Data Quality Assurance

#### Data Validation Procedures
A quality checklist must be used to validate test data before each major test cycle.

**Test Data Quality Checklist**:
*   **Completeness**: [✔] All required fields (`ssn`, `ficoscore`, etc.) are populated.
*   **Accuracy**: [✔] `ficoscore` is within the 300-850 range. `rating` corresponds to the score.
*   **Consistency**: [✔] The same `ssn` is not duplicated across records.
*   **Coverage**: [✔] Data exists for all defined test scenarios (high/low scores, all ratings).
*   **Security**: [✔] No real PII (SSN, Name, DOB) exists in non-production environments.

## Evidence Summary

*   **Scope Analyzed**: The analysis covered all files within the `CreditCheckService` TIBCO BusinessWorks project.
*   **Key Data Points**:
    *   1 REST API endpoint (`/creditscore`) was identified.
    *   1 JDBC database connection to a PostgreSQL-compatible database was identified.
    *   2 core database operations were found: a `SELECT` query and an `UPDATE` statement on the `public.creditscore` table.
*   **References**:
    *   `CreditCheckService/Processes/creditcheckservice/Process.bwp`: Defines the main process flow, including the API call and subprocess invocation.
    *   `CreditCheckService/Processes/creditcheckservice/LookupDatabase.bwp`: Contains the JDBC Query and Update activities.
    *   `CreditCheckService/Resources/creditcheckservice/JDBCConnectionResource.jdbcResource`: Defines the PostgreSQL database connection.
    *   `CreditCheckService/Service Descriptors/creditcheckservice.Process-CreditScore.json`: Defines the Swagger/OpenAPI contract for the REST service.

## Assumptions Made

*   The PostgreSQL database connection (`org.postgresql.Driver`) is intended for an **AlloyDB** instance, as per the MFDA modernization context.
*   The business requires standard API security (e.g., API Keys, OAuth) and rate limiting, which should be included in the test data strategy even if not explicitly defined in the current TIBCO configuration.
*   Personally Identifiable Information (PII) includes SSN, FirstName, LastName, and DOB, and must be masked in all non-production environments.
*   The `ssn like ?` SQL syntax is used for an exact match and not a pattern match.

## Open Questions

*   What are the specific FICO score ranges that map to each `rating` (e.g., 'Excellent', 'Good')? This is needed for boundary value test data.
*   Are there any business rules that trigger based on the `numofpulls` (number of inquiries)?
*   What are the production-level performance and volume expectations (e.g., API calls per second, total records in the database)?
*   What are the definitive security requirements for the `/creditscore` API endpoint?

## Confidence Level

**Overall Confidence**: High

**Rationale**: The project is small, self-contained, and its purpose is clearly defined by the file names and contents. The integration points (one REST API, one database) are explicit and well-defined within the TIBCO process files, leaving little room for ambiguity. The data model is simple and directly referenced in the SQL queries.

**Evidence**:
*   The database schema and operations are explicitly defined in `CreditCheckService/Processes/creditcheckservice/LookupDatabase.bwp`.
*   The API contract is clearly documented in `CreditCheckService/Service Descriptors/creditcheckservice.Process-CreditScore.json`.
*   The database connection technology is specified in `CreditCheckService/Resources/creditcheckservice/JDBCConnectionResource.jdbcResource`.

## Action Items

**Immediate** (Next 1-2 weeks):
*   [ ] Develop and validate data masking scripts (SQL or Python) to ensure no PII leaks into DEV/TEST environments.
*   [ ] Create a baseline dataset of at least 1,000 masked records covering all defined functional scenarios (high/low scores, all ratings).

**Short-term** (Next 1-2 sprints):
*   [ ] Implement automated data generation scripts (e.g., Python with Faker) to populate DEV and TEST environments on demand.
*   [ ] Integrate the data generation process into the CI/CD pipeline to ensure fresh data for every test run.

**Long-term** (Next Quarter):
*   [ ] Establish a formal Test Data Management (TDM) framework, including automated data refresh cycles for the STAGING environment.
*   [ ] Develop a self-service portal for QE and developers to request specific, scenario-based test datasets.

## Risk Assessment

*   **High Risk**: PII Exposure. Without a robust data masking strategy, sensitive customer information (SSN, DOB) could be exposed in lower environments.
*   **Medium Risk**: Inadequate Performance Data. Testing with small data volumes may fail to uncover performance bottlenecks in database queries or API responses that will only appear under production load.
*   **Low Risk**: Stale Test Data. If not refreshed regularly, test data may become unrepresentative of production data, leading to regression tests that pass but miss new edge cases.