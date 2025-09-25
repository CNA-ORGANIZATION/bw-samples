## Executive Summary

Based on the codebase, this application is a TIBCO BusinessWorks microservice named `ExperianService`. Its sole function is to provide credit score information. It exposes a single REST API endpoint that accepts an individual's Personally Identifiable Information (PII), queries a PostgreSQL database using the provided Social Security Number (SSN), and returns key credit metrics such as FICO score, a credit rating, and the number of recent inquiries.

## Analysis

### Application Purpose

**Evidence**:
-   **Service Naming**: The project and module are named `ExperianService`, and the core process involves a resource named `Creditscore`. This strongly indicates the application's purpose is related to credit reporting.
-   **Process Flow**: The file `ExperianService.module\Processes\experianservice\module\Process.bwp` defines the end-to-end business logic. It outlines a sequence of receiving an HTTP request, parsing JSON data, querying a database, rendering a JSON response, and sending it back.
-   **API Contract**: The REST service contract is defined in `ExperianService.module\Service Descriptors\experianservice.module.Process-Creditscore.json`. It specifies a `POST /creditscore` endpoint.

**Core Functionality**:
The application functions as a backend service for on-demand credit score retrieval. It takes personal details as input and returns a summary of an individual's credit profile.

**Business Domain**:
The business domain is **Financial Services**, specifically **Credit Reporting**. This is evident from:
-   The service name `ExperianService`.
-   The data fields handled, such as `ssn`, `fiCOScore`, `rating`, and `noOfInquiries` found in `ExperianRequestSchema.xsd` and `ExperianResponseSchemaResource.xsd`.
-   The database query in `Process.bwp` which selects from a table named `creditscore`.

**User Types**:
The code does not define any human user roles. The consumer of this service is another application or system that makes API calls to the `/creditscore` endpoint. This is typical of a backend microservice architecture where this service provides a specific business capability to other parts of an enterprise system (e.g., a loan origination system, a customer onboarding portal).

**Key Operations**:
The single key business operation implemented is **Credit Score Inquiry**. This operation takes an individual's PII to identify them and returns their credit data.

### Business Capabilities

**Features**:
-   **On-demand Credit Score Retrieval**: The system provides a real-time credit score lookup capability.
    -   **Evidence**: The `POST /creditscore` endpoint defined in `ExperianService.module\Service Descriptors\experianservice.module.Process-Creditscore.json` exposes this feature.

**Workflows**:
-   **PII-based Credit Data Lookup**: The application implements a straight-through workflow for retrieving credit data.
    1.  Receives a request with PII.
    2.  Parses the PII.
    3.  Uses the SSN to query a database for credit information.
    4.  Formats the retrieved data into a standardized JSON response.
    5.  Returns the credit data to the requesting system.
    -   **Evidence**: The sequence of activities (`HTTPReceiver` -> `ParseJSON` -> `JDBCQuery` -> `RenderJSON` -> `SendHTTPResponse`) in `ExperianService.module\Processes\experianservice\module\Process.bwp` defines this workflow.

**Integrations**:
-   **Credit Data Repository (PostgreSQL)**: The service integrates with a PostgreSQL database to fetch credit score data.
    -   **Evidence**: The file `ExperianService.module\Resources\experianservice\module\JDBCConnectionResource.jdbcResource` defines the connection to a PostgreSQL database (`jdbc:postgresql://localhost:5432/bookstore`). The `JDBCQuery` activity in `Process.bwp` executes the query `SELECT * FROM public.creditscore where ssn like ?`.

**Data Management**:
-   **Credit Profile Data**: The system processes and returns data related to an individual's credit profile.
    -   **Input Data (PII)**: The system handles an individual's `firstName`, `lastName`, `dob` (date of birth), and `ssn`.
        -   **Evidence**: `ExperianService.module\Schemas\ExperianRequestSchema.xsd`.
    -   **Output Data (Credit Metrics)**: The system provides `fiCOScore`, `rating`, and `noOfInquiries`.
        -   **Evidence**: `ExperianService.module\Schemas\ExperianResponseSchemaResource.xsd`.

### Business Rules & Constraints

**Validation Rules**:
-   **Mandatory PII for Lookup**: To perform a credit score inquiry, the `firstName`, `lastName`, `dob`, and `ssn` of an individual are all required.
    -   **Evidence**: The `required` array `[ "firstName", "lastName", "dob", "ssn" ]` in the API definition file `ExperianService.module\Service Descriptors\experianservice.module.Process-Creditscore.json`.

**Process Rules**:
-   **SSN as Primary Identifier**: The system uses the Social Security Number (SSN) as the key to look up an individual's credit record in the database.
    -   **Evidence**: The SQL statement `SELECT * FROM public.creditscore where ssn like ?` in the `JDBCQuery` activity within `Process.bwp`.

**Access Rules**:
-   The functionality is accessible via an unauthenticated HTTP POST request to the `/creditscore` endpoint. The code shows no implementation of authentication or authorization checks (e.g., API keys, OAuth tokens).
    -   **Evidence**: The process flow in `Process.bwp` begins directly with an `HTTPReceiver` activity and contains no subsequent security-related validation steps.

## Evidence Summary

-   **Scope Analyzed**: The analysis covered all files within the `ExperianService` and `ExperianService.module` projects, including TIBCO process files, resource configurations, and data schemas.
-   **Key Data Points**:
    -   1 REST endpoint (`POST /creditscore`) was identified.
    -   1 business workflow (Credit Score Inquiry) was documented.
    -   1 database integration (PostgreSQL) was found.
    -   4 required input data fields and 3 output data fields were cataloged.
-   **References**:
    -   `ExperianService.module/Processes/experianservice/module/Process.bwp`: Provided the core business workflow.
    -   `ExperianService.module/Service Descriptors/experianservice.module.Process-Creditscore.json`: Defined the API contract.
    -   `ExperianService.module/Schemas/*.xsd`: Defined the input and output data models.
    -   `ExperianService.module/Resources/experianservice/module/JDBCConnectionResource.jdbcResource`: Confirmed the database technology and connection details.

## Assumptions Made

-   It is assumed that the service name `ExperianService` indicates that the data originates from or is related to the Experian credit bureau, even though the direct integration shown is with a PostgreSQL database.
-   The database name `bookstore` found in the JDBC connection string is assumed to be a placeholder from development, and the actual database contains credit-related data as evidenced by the query against the `public.creditscore` table.
-   The lack of authentication logic within the TIBCO module suggests that security may be handled by an upstream component, such as an API Gateway, which is outside the scope of this codebase.

## Open Questions

-   What is the source of the data in the `public.creditscore` table? Is it populated via a batch process from an external provider like Experian?
-   How is access to the `/creditscore` endpoint secured in production? Are there security layers (e.g., an API Gateway) not visible in this module's code?
-   What are the specific business definitions for the `rating` field (e.g., what FICO score ranges correspond to "Good", "Fair", "Excellent")?

## Confidence Level

**Overall Confidence**: High

**Rationale**: The codebase is small, self-contained, and has a very specific purpose. The naming of files, processes, and data schemas is highly descriptive, leaving little ambiguity about the application's core function. The entire business workflow is contained within a single process file (`Process.bwp`), making it easy to trace from start to finish.

**Evidence**:
-   **File References**: `Process.bwp` clearly shows the sequence of operations.
-   **Configuration Files**: `JDBCConnectionResource.jdbcResource` and `Creditscore.httpConnResource` explicitly define the external integration points.
-   **Code Examples**: The SQL query `SELECT * FROM public.creditscore where ssn like ?` is direct evidence of the system's primary lookup logic.
-   **Metrics**: The API contract in `Process-Creditscore.json` is unambiguous about the service's inputs and outputs.