## Executive Summary

This report outlines a comprehensive test data strategy for the `ExperianService` module, focusing on its mainframe integration points as part of the MFDA (Mainframe and Distributed Application) initiative. The analysis identified two primary integration types: a REST API for credit score inquiries (classified as Apigee/API) and a JDBC connection to a PostgreSQL database (classified as AlloyDB). The strategy provides detailed requirements for generating, managing, and validating test data across Integration, Regression, and End-to-End testing phases to ensure a high-quality, low-risk migration and operation.

## Analysis

The `ExperianService` application provides a single core function: retrieving a person's credit score information based on their Social Security Number (SSN). This is exposed via a REST API and backed by a PostgreSQL database. The following test data strategy is designed to cover all aspects of this integration.

### 1. Integration Testing Data

This data is focused on validating the individual API and Database integration points.

#### Apigee/API Integration Test Data

The service exposes a `/creditscore` endpoint via a POST request. Test data must cover valid inputs, error conditions, and boundary cases for the request and response payloads.

**Valid Request Payload (`/creditscore`)**:
Based on `ExperianRequestSchema.xsd`.

```json
{
  "dob": "1985-03-15",
  "firstName": "John",
  "lastName": "Doe",
  "ssn": "[REDACTED_SSN]"
}
```

**Valid Response Payload**:
Based on `ExperianResponseSchemaResource.xsd`.

```json
{
  "fiCOScore": 780,
  "rating": "Excellent",
  "noOfInquiries": 2
}
```

**Error Response Payload** (e.g., User Not Found):

```json
{
  "errorCode": "USR_NOT_FOUND",
  "errorMessage": "User with the specified SSN was not found.",
  "errorDetails": "SSN [REDACTED_SSN] does not exist in the credit score database."
}
```

**API Test Data Scenarios**:

| Scenario | `ssn` | `firstName` | `lastName` | `dob` | Expected Outcome |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Happy Path** | Valid, existing SSN | Valid | Valid | Valid | `200 OK` with credit data |
| **User Not Found** | Valid format, non-existent | Valid | Valid | Valid | `404 Not Found` error |
| **Invalid SSN Format** | `123-45-678` (invalid) | Valid | Valid | Valid | `400 Bad Request` error |
| **Missing Required Field**| Valid, existing SSN | Valid | `null` | Valid | `400 Bad Request` error |
| **Boundary - Min Length**| `1` | `J` | `D` | `2000-01-01` | `400 Bad Request` |
| **Boundary - Max Length**| SSN with 500 chars | Name with 500 chars | Name with 500 chars | Date with 500 chars | `400 Bad Request` |

#### AlloyDB Integration Test Data

The application uses a PostgreSQL database, which maps to AlloyDB in the MFDA context. The `Process.bwp` file indicates a query against a `creditscore` table.

**Database Test Datasets (`public.creditscore` table)**:

```sql
-- Schema derived from the JDBCQuery activity in Process.bwp
CREATE TABLE public.creditscore (
    firstname VARCHAR(255),
    lastname VARCHAR(255),
    ssn VARCHAR(11) PRIMARY KEY,
    dateofBirth VARCHAR(10),
    ficoscore INT,
    rating VARCHAR(50),
    numofpulls INT
);

-- Sample Records for Testing
-- Happy Path (Excellent Score)
INSERT INTO public.creditscore (firstname, lastname, ssn, dateofBirth, ficoscore, rating, numofpulls) VALUES
('John', 'Doe', '[REDACTED_SSN_1]', '1985-03-15', 820, 'Excellent', 1);

-- Good Score
INSERT INTO public.creditscore (firstname, lastname, ssn, dateofBirth, ficoscore, rating, numofpulls) VALUES
('Jane', 'Smith', '[REDACTED_SSN_2]', '1990-07-22', 710, 'Good', 3);

-- Fair Score
INSERT INTO public.creditscore (firstname, lastname, ssn, dateofBirth, ficoscore, rating, numofpulls) VALUES
('Jim', 'Brown', '[REDACTED_SSN_3]', '1978-11-08', 650, 'Fair', 8);

-- Poor Score
INSERT INTO public.creditscore (firstname, lastname, ssn, dateofBirth, ficoscore, rating, numofpulls) VALUES
('Emily', 'Jones', '[REDACTED_SSN_4]', '1995-05-19', 550, 'Poor', 15);

-- Boundary Case (No Inquiries)
INSERT INTO public.creditscore (firstname, lastname, ssn, dateofBirth, ficoscore, rating, numofpulls) VALUES
('Sam', 'Wilson', '[REDACTED_SSN_5]', '2001-01-20', 750, 'Good', 0);
```

**Database Performance Test Data**:
A large volume of data is required to test query performance. The following procedure can be used to generate this data.

```sql
-- Procedure to generate 1 million records for performance testing
INSERT INTO public.creditscore (firstname, lastname, ssn, dateofBirth, ficoscore, rating, numofpulls)
SELECT
    'FirstName' || s.id,
    'LastName' || s.id,
    LPAD((s.id)::text, 9, '0'),
    (CURRENT_DATE - (random() * 20000 + 6570) * '1 day'::interval)::date::text,
    (random() * 550 + 300)::int,
    CASE WHEN (random() * 550 + 300)::int > 800 THEN 'Excellent'
         WHEN (random() * 550 + 300)::int > 670 THEN 'Good'
         WHEN (random() * 550 + 300)::int > 580 THEN 'Fair'
         ELSE 'Poor'
    END,
    (random() * 20)::int
FROM generate_series(1, 1000000) AS s(id);
```

### 2. Regression Testing Data

Regression testing requires a stable, version-controlled dataset that covers all critical business scenarios to ensure new changes do not break existing functionality.

**Business Process Data Set: Credit Score Inquiry**

| Test Case ID | Description | Input SSN | Expected FICO Score | Expected Rating |
| :--- | :--- | :--- | :--- | :--- |
| REG-CS-001 | Excellent Credit | `[REDACTED_SSN_1]` | 820 | `Excellent` |
| REG-CS-002 | Good Credit | `[REDACTED_SSN_2]` | 710 | `Good` |
| REG-CS-003 | Fair Credit | `[REDACTED_SSN_3]` | 650 | `Fair` |
| REG-CS-004 | Poor Credit | `[REDACTED_SSN_4]` | 550 | `Poor` |
| REG-CS-005 | Non-Existent User | `[NON_EXISTENT_SSN]` | N/A | N/A (Error) |

This dataset should be loaded into the test database before each regression run.

### 3. End-to-End Testing Data

This data simulates a full day of operations for the credit inquiry workflow.

**Daily Operations Workflow: Real-time Credit Score Inquiry**

| Data Requirement | Volume | Description |
| :--- | :--- | :--- |
| **API Calls** | 100,000 calls/day | A mix of valid and invalid SSNs to simulate real-world traffic. |
| **Database Reads** | 100,000 queries/day | Corresponding read operations on the `creditscore` table. |
| **New Customers** | 1,000 records/day | Data representing new credit profiles added to the database daily. |

### 4. Environment-Specific Test Data Management

| Environment | Purpose | Data Volume | Data Refresh | Data Masking |
| :--- | :--- | :--- | :--- | :--- |
| **Development** | Unit & Feature Testing | 1,000 records | Weekly | Full PII Masking |
| **Test** | Integration & E2E Testing | 100,000 records | Daily | Partial PII Masking |
| **Staging** | Performance & UAT | 1,000,000+ records | On-demand | Minimal (non-PII only) |

### 5. Test Data Generation and Management Procedures

#### Automated Test Data Generation
A script should be created to populate environments with realistic, anonymized data.

```python
# Example Python script using Faker for data generation
import csv
from faker import Faker
import random

fake = Faker()

def generate_credit_data(num_records):
    """Generates a list of credit score records."""
    data = []
    for _ in range(num_records):
        score = random.randint(300, 850)
        if score > 800:
            rating = 'Excellent'
        elif score > 670:
            rating = 'Good'
        elif score > 580:
            rating = 'Fair'
        else:
            rating = 'Poor'
        
        record = {
            'firstname': fake.first_name(),
            'lastname': fake.last_name(),
            'ssn': fake.ssn(),
            'dateofBirth': fake.date_of_birth(minimum_age=18, maximum_age=90).strftime('%Y-%m-%d'),
            'ficoscore': score,
            'rating': rating,
            'numofpulls': random.randint(0, 20)
        }
        data.append(record)
    return data

# Generate data and write to a CSV file for DB import
credit_records = generate_credit_data(1000)
with open('credit_score_data.csv', 'w', newline='') as f:
    writer = csv.DictWriter(f, fieldnames=credit_records[0].keys())
    writer.writeheader()
    writer.writerows(credit_records)
```

#### Data Masking and Privacy Protection
Sensitive data like SSN must be masked in non-production environments.

```sql
-- Example SQL for masking SSN in a TEST environment
UPDATE public.creditscore
SET
    ssn = '***-**-' || SUBSTRING(ssn, 8, 4),
    firstname = 'TestFirstName' || (random()*1000)::int,
    lastname = 'TestLastName' || (random()*1000)::int
WHERE ssn IS NOT NULL;
```

### 6. Test Data Quality Assurance

#### Data Validation Procedures
A quality checklist must be used to validate all test datasets before use.

**Test Data Quality Checklist**:

-   **Completeness**: Are all required fields (`ssn`, `firstName`, etc.) populated?
-   **Accuracy**: Do `ficoscore` values fall within the 300-850 range? Do ratings correspond to scores?
-   **Consistency**: Is the `ssn` format consistent across all records?
-   **Uniqueness**: Is the `ssn` field unique for every record?
-   **Security**: Is all PII (SSN, Name, DOB) properly masked in non-production environments?

## Evidence Summary

-   **Scope Analyzed**: The analysis covered the `ExperianService` TIBCO BusinessWorks project, including one module and its associated processes, resources, and schemas.
-   **Key Data Points**:
    -   1 REST API endpoint (`/creditscore`) was identified.
    -   1 JDBC database connection to a PostgreSQL database was found.
    -   2 primary data schemas (`ExperianRequestSchema`, `ExperianResponseSchemaResource`) define the data contracts.
-   **References**:
    -   `ExperianService.module/Processes/experianservice/module/Process.bwp` (Defines the integration logic).
    -   `ExperianService.module/Resources/experianservice/module/JDBCConnectionResource.jdbcResource` (Defines the database connection).
    -   `ExperianService.module/Service Descriptors/experianservice.module.Process-Creditscore.json` (Defines the API contract).

## Assumptions Made

-   The PostgreSQL database connection is the designated "AlloyDB" integration point within the MFDA context.
-   The REST service is intended to be exposed via an "Apigee" gateway, as per the MFDA pattern.
-   The `public.creditscore` table exists in the target database with a schema matching the fields returned by the `JDBCQuery` activity.
-   Authentication (e.g., API Keys) and rate limiting are managed at a gateway level (Apigee) and are required for testing, even if not explicitly defined in the BW process.

## Open Questions

-   What are the specific performance SLAs for the `/creditscore` API endpoint (e.g., p99 latency, max requests per second)?
-   What are the defined valid value ranges and business rules for `ficoscore`, `rating`, and `numofpulls`?
-   Are there other downstream systems that consume data from the `creditscore` table, which might be affected by data changes?
-   What is the official data retention policy for test data in each environment?

## Confidence Level

**Overall Confidence**: High

**Rationale**: The provided codebase represents a small, self-contained service with a clear and simple integration pattern. The data entities and flows are well-defined in the process and schema files, allowing for a confident and detailed test data strategy to be formulated. The scope is limited, reducing the chance of unknown dependencies.

**Evidence**:
-   The entire logic is contained within a single process file: `ExperianService.module/Processes/experianservice/module/Process.bwp`.
-   The database connection is explicitly defined in `JDBCConnectionResource.jdbcResource`, pointing to a PostgreSQL driver.
-   The API contract is clearly documented in the Swagger JSON file `experianservice.module.Process-Creditscore.json`.

## Action Items

**Immediate** (Next 1-2 days):
-   [ ] Generate and load the initial 1,000-record masked dataset into the DEV environment.
-   [ ] Manually create test data for the 5 primary regression scenarios (REG-CS-001 to REG-CS-005).

**Short-term** (Next Sprint):
-   [ ] Automate the test data generation script (Python/Faker) and integrate it with the DEV environment setup process.
-   [ ] Develop and automate the SQL masking scripts for use in TEST environment data refreshes.

**Long-term** (Next Quarter):
-   [ ] Integrate the automated data generation and masking scripts into the CI/CD pipeline to ensure environments are refreshed automatically.
-   [ ] Build a monitoring dashboard to track test data quality metrics (freshness, volume, coverage).

## Risk Assessment

-   **High Risk**: Insufficient variety in the `creditscore` test data (e.g., only "Good" ratings) could lead to uncaught bugs in business logic for other rating types.
-   **Medium Risk**: Using stale or inconsistent data across test environments may result in tests that pass locally but fail in integrated environments.
-   **Low Risk**: Manual data generation is time-consuming and prone to human error, potentially slowing down testing cycles.