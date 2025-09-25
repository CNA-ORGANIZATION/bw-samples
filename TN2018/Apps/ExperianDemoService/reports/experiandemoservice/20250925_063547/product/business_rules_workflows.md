## Executive Summary

This analysis documents the business rules and workflows embedded in the `ExperianService` application. The system's sole function is to act as an internal service that retrieves a customer's credit score information based on their Social Security Number (SSN). The primary workflow involves receiving a request with customer PII, querying an internal PostgreSQL database, and returning key credit metrics such as FICO score, rating, and the number of inquiries. The governing business rules ensure that requests are properly formed, data is looked up accurately using the SSN, and access to the sensitive credit database is secured.

## Analysis

### Business Rules Catalog

The system operates under a set of implicit policies and rules that govern data handling, access, and processing. These rules are critical for maintaining data integrity, security, and operational consistency.

| Rule Name | Business Purpose | Policy Statement | Business Impact if Violated | Owner Department |
| :--- | :--- | :--- | :--- | :--- |
| **PII for Identification** | To ensure accurate lookup of an individual's credit history. | All credit score inquiries must be initiated with the individual's full name, date of birth (DOB), and Social Security Number (SSN). | Retrieving incorrect credit data, leading to flawed lending decisions, customer privacy violations, and non-compliance with fair credit reporting laws (FCRA). | Credit Risk / Underwriting |
| **SSN-Based Data Retrieval** | To use a unique identifier for fetching credit information from the internal database. | The system's policy is to retrieve credit profiles by performing an exact match on the customer's SSN within the credit score database. | The wrong person's sensitive credit data could be exposed or used, resulting in severe privacy breaches and incorrect financial assessments. | Data Management / IT |
| **Standardized Credit Response** | To provide a consistent data set for all downstream decision-making systems. | Every credit score response must contain the FICO score, a credit rating, and the total number of recent inquiries. | Automated systems (e.g., loan origination) may fail to process applications, causing operational delays and requiring costly manual reviews. | Product Management / IT |
| **Secure Database Access** | To protect sensitive customer credit information from unauthorized access. | Application access to the credit score database must be authenticated using predefined, securely managed credentials. | A data breach could expose the credit information of all customers, leading to significant regulatory fines, legal liability, and catastrophic brand damage. | Information Security |

### Workflow Documentation

The application executes a single, critical business workflow for retrieving customer credit information.

**Workflow Name**: Internal Credit Score Retrieval

**Business Purpose**:
This workflow provides a centralized, automated way for internal company systems to retrieve a customer's credit score and related metrics. It enables rapid, data-driven decisions for processes like loan applications, credit line increases, or identity verification without requiring a live, per-request call to an external credit bureau. This improves response times and reduces operational costs.

**Process Steps**:

1.  **Credit Inquiry Received**
    *   **Who**: An internal business system (e.g., Loan Origination System, Customer Onboarding App).
    *   **What**: The system sends a request containing a customer's First Name, Last Name, Date of Birth, and SSN.
    *   **Why**: To initiate the process of fetching the customer's credit profile.
    *   **When**: Triggered when a business process requires a credit check.
    *   **Evidence**: The process starts with an `HTTPReceiver` activity listening on the `/creditscore` endpoint, which expects a JSON payload defined by `ExperianRequestSchema.xsd`.

2.  **Customer Lookup in Credit Database**
    *   **Who**: The ExperianService application.
    *   **What**: The application uses the provided SSN to search for a matching record in the internal credit score database.
    *   **Why**: To locate the specific credit profile for the requested individual.
    *   **When**: Immediately after the incoming request is parsed and validated.
    *   **Evidence**: A `JDBCQuery` activity executes the SQL statement `SELECT * FROM public.creditscore where ssn like ?` against the PostgreSQL database defined in `JDBCConnectionResource.jdbcResource`.

3.  **Credit Data Formatted for Response**
    *   **Who**: The ExperianService application.
    *   **What**: The system extracts the FICO score, credit rating, and number of inquiries from the database record. It then formats this information into a standardized JSON structure.
    *   **Why**: To prepare a consistent, machine-readable response for the requesting system.
    *   **When**: After the database query successfully returns a result.
    *   **Evidence**: The `RenderJSON` activity uses the `ExperianResponseSchemaResource.xsd` to structure the output, mapping database columns like `ficoscore` and `rating` to the JSON response.

4.  **Credit Profile Returned**
    *   **Who**: The ExperianService application.
    *   **What**: The system sends the structured JSON response back to the originating system.
    *   **Why**: To deliver the requested credit information, allowing the business process to continue.
    *   **When**: As the final step of the workflow.
    *   **Evidence**: The `SendHTTPResponse` activity sends the generated JSON string back to the initial caller.

## Evidence Summary

*   **Scope Analyzed**: The analysis covered the TIBCO BusinessWorks project `ExperianService`, including one module (`ExperianService.module`) containing a single business process (`Process.bwp`), associated schemas, and resource configurations.
*   **Key Data Points**:
    *   **1** primary business workflow identified (Credit Score Retrieval).
    *   **4** key business rules extracted from the process logic and configuration.
    *   Input data requires **4** fields: `firstName`, `lastName`, `dob`, `ssn`.
    *   Output data provides **3** fields: `fiCOScore`, `rating`, `noOfInquiries`.
*   **References**: Findings are based on the analysis of `Process.bwp`, `ExperianRequestSchema.xsd`, `ExperianResponseSchemaResource.xsd`, and `JDBCConnectionResource.jdbcResource`.

## Assumptions Made

*   **Service Name vs. Functionality**: It is assumed that despite the name "ExperianService," the application's current implementation does **not** make a live call to Experian. The code clearly shows it queries an internal PostgreSQL database table named `creditscore`. This service likely retrieves data that was previously sourced from Experian.
*   **Database Name**: The JDBC connection string points to a database named `bookstore`. It is assumed this is a non-production or default name, and in a real environment, it would be a more appropriately named database (e.g., `credit_data`).
*   **Business Ownership**: The "Owner Department" for each rule was inferred based on standard corporate structures. For example, "Credit Risk" is assumed to own rules related to lending decisions, and "Information Security" owns rules related to data access.
*   **Error Handling**: The main process flow does not explicitly detail the business process for handling a "not found" scenario (i.e., when an SSN does not exist in the database). It is assumed that an empty or error response is returned, but the exact business rule for this path is not defined in the happy-path logic.

## Open Questions

*   **Data Sourcing**: How is the `public.creditscore` table populated and kept up-to-date? Is there a separate batch process that loads data from Experian?
*   **"Not Found" Business Rule**: What is the defined business process when a requested SSN is not found in the database? Should the system return an error, a null response, or trigger a different workflow?
*   **SSN Validation**: What are the specific validation rules for the SSN format? The schema defines it as a generic string.
*   **Security Context**: How are clients of this service authenticated and authorized? The process flow does not show any explicit authentication checks, which is a significant security concern.

## Confidence Level

**Overall Confidence**: High

**Rationale**: The provided codebase contains a single, well-defined TIBCO BusinessWorks process. The purpose of the service is unambiguous due to the clear naming in schemas (`ExperianRequestSchema`, `ExperianResponseSchema`), the SQL query (`SELECT * FROM public.creditscore`), and the process flow itself. The limited scope and clarity of the implementation allow for a high degree of confidence in the analysis of the existing workflow and rules.

**Evidence**:
*   **File**: `ExperianService.module/Processes/experianservice/module/Process.bwp` - This file contains the end-to-end workflow, from receiving the HTTP request to querying the database and sending the response.
*   **File**: `ExperianService.module/Schemas/ExperianRequestSchema.xsd` - This schema explicitly defines the required input PII (`ssn`, `firstName`, etc.), confirming the data needed to start the process.
*   **File**: `ExperianService.module/Resources/experianservice/module/JDBCConnectionResource.jdbcResource` - This configuration file confirms the direct database interaction with a PostgreSQL instance and the `bwuser` credentials.

## Action Items

**Immediate** (Next 1-2 weeks):
*   [ ] **Clarify "Not Found" Logic**: Business stakeholders (Credit Risk, Product Management) need to define the required system behavior when an SSN is not found in the database.
*   [ ] **Investigate Security Gaps**: IT and Information Security must investigate the lack of authentication and authorization on the `/creditscore` endpoint and prioritize securing it.

**Short-term** (Next 1-3 months):
*   [ ] **Document Data Lineage**: The Data Management team should document the process for how the `creditscore` table is populated to ensure data freshness and accuracy.
*   [ ] **Implement Input Validation**: Enhance the service to include strict format validation for the incoming SSN and other PII, as per business requirements.

**Long-term** (Next 3-6 months):
*   [ ] **Formalize Policies**: Officially document the identified business rules and workflows in a central policy repository, owned by the respective business departments.

## Risk Assessment

*   **High Risk**: The lack of authentication on an endpoint that processes and returns sensitive PII (SSN, FICO scores) is a critical security vulnerability. Unauthorized access could lead to a major data breach.
*   **Medium Risk**: The ambiguity around the "not found" scenario could lead to inconsistent error handling by client systems, potentially causing downstream process failures.
*   **Low Risk**: The use of a generic database name (`bookstore`) in the connection string could cause confusion in multi-environment setups, but does not pose a direct operational risk if managed correctly.