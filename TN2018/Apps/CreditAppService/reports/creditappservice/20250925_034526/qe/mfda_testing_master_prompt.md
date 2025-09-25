## Executive Summary

This report provides a comprehensive QE testing strategy for the `CreditApp` TIBCO BusinessWorks (BW) application, focusing on its mainframe integration aspects as defined by the MFDA scope. The analysis reveals that the application's core function is to act as an API facade, exposing a single REST endpoint (`/creditdetails`) that orchestrates calls to two downstream credit scoring services (mocked as `EquifaxScore` and `ExperianScore`).

The only MFDA integration pattern identified is **Web Services/API**. No evidence of MFT, MQ/Kafka, AlloyDB, or Oracle integrations was found in the codebase. Therefore, this testing strategy is exclusively focused on ensuring the quality, reliability, and security of these API interactions. The primary risks identified are the potential for downstream service failures, data transformation errors between the services, and a lack of defined security mechanisms. The recommended testing strategy prioritizes end-to-end validation of the business workflow, robust error handling scenarios, and performance benchmarking.

## MFDA Integration Matrix

| Integration Type | Upstream Component | Downstream Component | Data Flow Direction | Integration Method |
| :--- | :--- | :--- | :--- | :--- |
| Apigee/Web Service | External Client | `CreditApp.module` (`/creditdetails`) | Inbound | REST (POST) |
| Apigee/Web Service | `CreditApp.module` | `EquifaxScore` Service | Outbound | REST (POST) |
| Apigee/Web Service | `CreditApp.module` | `ExperianScore` Service | Outbound | REST (POST) |

## Integration Architecture Wire Diagram

This diagram illustrates the data flow for the credit scoring process. An external client sends a request to the `CreditApp`, which then queries both the Equifax and Experian services in parallel before aggregating the responses.

```mermaid
graph TD
    subgraph "Client"
        Client["External Client (e.g., Web App)"]
    end

    subgraph "MFDA Integration Layer (CreditApp)"
        MainProcess["MainProcess (/creditdetails)"]
        EquifaxSubProcess["EquifaxScore Sub-Process"]
        ExperianSubProcess["ExperianScore Sub-Process"]
    end

    subgraph "Downstream Services"
        EquifaxService["Equifax Service Endpoint"]
        ExperianService["Experian Service Endpoint"]
    end

    Client -->|1. POST Request (SSN, Name, DOB)| MainProcess
    MainProcess -->|2a. Call| EquifaxSubProcess
    MainProcess -->|2b. Call| ExperianSubProcess
    EquifaxSubProcess -->|3a. HTTP POST| EquifaxService
    ExperianSubProcess -->|3b. HTTP POST| ExperianService
    EquifaxService -->|4a. HTTP Response| EquifaxSubProcess
    ExperianService -->|4b. HTTP Response| ExperianSubProcess
    EquifaxSubProcess -->|5a. Return Score| MainProcess
    ExperianSubProcess -->|5b. Return Score| MainProcess
    MainProcess -->|6. Aggregated JSON Response| Client

    classDef da fill:#ccccff,stroke:#0066cc,stroke-width:2px
    classDef external fill:#f9f,stroke:#333,stroke-width:2px

    class MainProcess,EquifaxSubProcess,ExperianSubProcess da
    class Client,EquifaxService,ExperianService external
```

## Detailed Test Cases by Integration Type

### Apigee/Web Services Integration Test Cases

**Test Case ID:** MFDA-INT-API-001
**Test Case Name:** Happy Path - Successful Credit Score Retrieval
**Integration Type:** Apigee/Web Services
**Test Type:** Integration
**Description/Summary:** This test validates the end-to-end flow where a valid request is sent to the `/creditdetails` endpoint, and the service successfully calls both downstream services (Equifax, Experian) and returns an aggregated response.
**Pre-conditions:**
-   `CreditApp.module` is deployed and running.
-   Mock services for Equifax and Experian are deployed and configured to return successful responses.
**Test Data Requirements:**
-   **Request Payload:**
    ```json
    {
      "DOB": "01-01-1980",
      "FirstName": "John",
      "LastName": "Doe",
      "SSN": "[REDACTED_SSN]"
    }
    ```
-   **Mock Equifax Response (200 OK):**
    ```json
    {
      "FICOScore": 750,
      "NoOfInquiries": 2,
      "Rating": "Good"
    }
    ```
-   **Mock Experian Response (200 OK):**
    ```json
    {
      "fiCOScore": 760,
      "rating": "Good",
      "noOfInquiries": 3
    }
    ```
**Test Steps:**
1.  Construct a POST request to the `/creditdetails` endpoint with the valid request payload.
2.  Send the request.
3.  Receive the response from the service.
4.  Validate the HTTP status code is 200 (OK).
5.  Parse the JSON response body.
**Expected Results:**
-   The response body should be a valid JSON object.
-   The response should contain both `EquifaxResponse` and `ExperianResponse` objects.
-   `EquifaxResponse.FICOScore` should be `750`.
-   `ExperianResponse.FICOScore` should be `760`.
-   Response time should be less than 2000ms.
**Integration Environment Details:**
-   **Endpoint URL:** `http://{BW.HOST.NAME}:{BW.CLOUD.PORT}/creditdetails`
-   **Mock Equifax URL:** `http://{BWAppHostname}:13080/creditscore`
-   **Mock Experian URL:** `http://{ExperianAppHostname}:7080/creditscore`

---

**Test Case ID:** MFDA-INT-API-002
**Test Case Name:** Negative - Downstream Service Timeout
**Integration Type:** Apigee/Web Services
**Test Type:** Integration (Negative)
**Description/Summary:** This test validates that the main process handles a timeout from one of the downstream services gracefully and returns a partial or error response as designed.
**Pre-conditions:**
-   `CreditApp.module` is deployed and running.
-   The mock Experian service is configured to delay its response by 1500ms (longer than the 1000ms timeout configured in `HttpClientResource1.httpClientResource`).
-   The mock Equifax service is configured to respond normally.
**Test Data Requirements:**
-   **Request Payload:** Same as MFDA-INT-API-001.
**Test Steps:**
1.  Construct a POST request to the `/creditdetails` endpoint.
2.  Send the request.
3.  Receive the response from the service.
4.  Validate the HTTP status code.
**Expected Results:**
-   The system should not hang or crash.
-   The `ExperianScore` subprocess should encounter a timeout fault.
-   The main process should catch the fault. The expected behavior for a partial failure is not defined in the process; the test should verify the actual outcome (e.g., does it return a 500 error, or a 200 OK with only the Equifax data and a null/empty Experian object?).
-   Logs should contain an `ActivityTimedOutException` for the `SendHTTPRequest` activity in the `ExperianScore` process.
**Integration Environment Details:**
-   Same as MFDA-INT-API-001.

---

**Test Case ID:** MFDA-E2E-API-001
**Test Case Name:** End-to-End - Invalid Input Data Validation
**Integration Type:** Apigee/Web Services
**Test Type:** End-to-End
**Description/Summary:** Validates how the system handles requests with missing or malformed data. Since no explicit validation is built into the TIBCO processes, this test will verify the default behavior.
**Pre-conditions:**
-   `CreditApp.module` is deployed and running.
-   Downstream mock services are running.
**Test Data Requirements:**
-   **Request Payload (Missing SSN):**
    ```json
    {
      "DOB": "01-01-1980",
      "FirstName": "John",
      "LastName": "Doe"
    }
    ```
**Test Steps:**
1.  Construct a POST request to `/creditdetails` with the payload missing the `SSN` field.
2.  Send the request.
3.  Receive and validate the response.
**Expected Results:**
-   The service should accept the request, as there is no schema validation on the input.
-   The calls to downstream services will be made with a null/empty `SSN`.
-   The final response will depend on how the downstream services handle the missing data. The test should assert the expected behavior (e.g., downstream services return an error, which is then propagated back to the client).
**Integration Environment Details:**
-   Same as MFDA-INT-API-001.

## Integration-Specific Requirements

### Apigee/Web Services Integration Details

**Environment Details:**
-   **Endpoint URLs:**
    -   **Service:** `http://{BW.HOST.NAME}:{BW.CLOUD.PORT}/creditdetails`
    -   **Equifax Mock:** `http://{BWAppHostname}:13080/creditscore`
    -   **Experian Mock:** `http://{ExperianAppHostname}:7080/creditscore`
-   **Authentication:** No authentication is implemented in the current codebase. For testing, this is a critical gap. Test cases should be designed to validate the future implementation of API Key or OAuth2 security policies in an API Gateway like Apigee.
-   **Request/Response Payloads:** Payloads are defined in `creditapp.module.MainProcess-CreditDetails.json`. The input uses the `GiveNewSchemaNameHere` definition, and the output uses `CreditScoreSuccessSchema`.

**Test Data Requirements:**
-   **Request Payloads:** Generate JSON payloads based on `GiveNewSchemaNameHere` schema, including valid data, missing fields (e.g., no SSN), and malformed data (e.g., invalid date format for DOB).
-   **Response Validation:** Create expected JSON responses based on the `CreditScoreSuccessSchema` to validate against the actual output. This includes scenarios with full data, partial data (if one downstream service fails), and error responses.
-   **Performance Benchmarks:** Define expected response times under various loads (e.g., P95 latency < 1500ms at 100 TPS).

**Test Scenarios:**
1.  **Happy Path:** Valid request leading to successful responses from both downstream services.
2.  **Authentication Scenarios (Future State):**
    -   Request with valid API Key/Token.
    -   Request with invalid/expired API Key/Token (expect 401/403).
    -   Request with no API Key/Token (expect 401/403).
3.  **Error Handling:**
    -   One or both downstream services are unavailable (connection refused).
    -   One or both downstream services time out.
    -   One or both downstream services return a 4xx or 5xx error.
4.  **Data Mapping:**
    -   Request with all fields populated.
    -   Request with optional fields missing.
    -   Verify that the data from the initial request (`SSN`, `FirstName`, etc.) is correctly mapped and sent to the downstream services.
    -   Verify that the responses from downstream services are correctly aggregated into the final `CreditScoreSuccessSchema` structure.
5.  **Load Testing:**
    -   Simulate concurrent requests to test the parallel execution of the Equifax and Experian subprocesses.
    -   Measure throughput and latency to identify performance bottlenecks.

## Test Data Strategy

The test data strategy must cover the API contract defined in the service descriptors.

-   **Integration Testing Data:**
    -   **Apigee/API:**
        -   **Request Payloads:** A suite of JSON files representing different customer profiles.
            -   `valid-customer.json`: All fields correct.
            -   `missing-ssn.json`: `SSN` field omitted.
            -   `long-names.json`: `FirstName` and `LastName` at max character limit.
            -   `special-chars.json`: Names with special characters to test encoding.
        -   **Mock Responses:** A collection of responses for the mocked Equifax and Experian services.
            -   `equifax-success-high-score.json`
            -   `experian-success-low-score.json`
            -   `service-unavailable-503.json`
            -   `bad-request-400.json`
-   **Regression/E2E Testing Data:**
    -   **End-to-End Scenarios:** Create datasets that represent a full business process. For example, a customer `CUST001` with a specific SSN should have a corresponding mock response from both Equifax and Experian, allowing testers to validate the entire flow for that specific customer.
    -   **Volume Testing Data:** Generate a large set (10,000+) of unique request payloads to test the system under load and identify any data-dependent processing issues.

## Environment Configuration Details

### Lower Environment Setup

For effective testing, a dedicated test environment is required with the ability to mock the downstream dependencies.

**Environment URLs and Access:**
-   **DEV:**
    -   Service URL: `http://localhost:8080/creditdetails`
    -   Equifax Mock: `http://localhost:13080/creditscore`
    -   Experian Mock: `http://localhost:7080/creditscore`
-   **TEST/STAGE:**
    -   Service URL: `http://creditapp-test.example.com/creditdetails`
    -   Mock URLs would point to a shared, deployed mock service (e.g., WireMock, SoapUI).

**Configuration Files:**
-   The `default.substvar` and `docker.substvar` files should be used to manage environment-specific hostnames (`ExperianAppHostname`, `BWAppHostname`). QE will need to provide different versions of these files for each test environment.

**Setup Procedures:**
1.  **Deploy Mock Services:** Deploy mock services (e.g., WireMock) that can simulate the Equifax and Experian APIs.
2.  **Configure Mocks:** Before each test run, configure the mock services with the expected responses for the specific test case (e.g., success, timeout, error).
3.  **Deploy CreditApp:** Deploy the `CreditApp.ear` to a TIBCO AppNode.
4.  **Configure Environment:** Apply the correct `.substvar` file for the target environment to ensure the `CreditApp` calls the correct mock service endpoints.
5.  **Run Tests:** Execute automated tests (e.g., using Postman/Newman, REST-assured) against the `CreditApp` endpoint.
6.  **Verify:** Check application logs, mock service request logs, and test assertion results to validate behavior.

## Assumptions Made

-   The TIBCO processes `EquifaxScore.bwp` and `ExperianScore.bwp` are intended to call external, third-party credit bureaus. For testing, these will be treated as external dependencies that must be mocked.
-   The exposed REST service (`/creditdetails`) is the primary entry point and would be managed by an API Gateway (like Apigee) in a production environment, which would handle security aspects like API key validation.
-   The `default.substvar` file's use of `localhost` indicates a local development setup. The `docker.substvar` file indicates a containerized deployment context. Testing must account for both.
-   The project is a TIBCO BusinessWorks 6.x application, and testing tools should be compatible with this stack.

## Open Questions

-   What are the specific business rules for aggregating scores? If Equifax returns a score but Experian fails, what should the final response be? The current process does not define this.
-   What are the security requirements for the `/creditdetails` endpoint? (e.g., API Key, OAuth, mTLS). None are currently implemented.
-   What are the performance and throughput SLAs for this service?
-   What are the detailed error handling requirements? Should the service return a 200 OK with partial data or a 5xx error on downstream failure?

## Confidence Level

**Overall Confidence:** High

**Rationale:** The project is small, self-contained, and follows a common API facade pattern. The logic within the TIBCO processes is straightforward orchestration. The lack of database or messaging components simplifies the integration testing scope significantly. The primary complexity lies in thoroughly testing the parallel execution and error handling of the downstream API calls.

**Evidence:**
-   **File References:** `CreditApp.module/Processes/MainProcess.bwp` clearly shows the orchestration logic, calling the two subprocesses. `EquifaxScore.bwp` and `ExperianScore.bwp` show the outbound HTTP/REST calls.
-   **Configuration Files:** `Resources/creditapp/module/HttpClientResource1.httpClientResource` and `HttpClientResource2.httpClientResource` define the connection details and timeouts for the downstream services.
-   **Code Examples:** The `<tibex:extActivity>` blocks within the `.bwp` files show the direct invocation of the subprocesses and the XSLT mapping used for data transformation.

## Action Items

**Immediate:**
-   **[ ] Develop a Mock Service Strategy:** Define and implement mock services for `EquifaxScore` and `ExperianScore` that can simulate success, error, and timeout scenarios.
-   **[ ] Create Basic Happy Path Test:** Implement an automated test case for the successful retrieval of credit scores to establish a baseline.

**Short-term:**
-   **[ ] Implement Negative Test Cases:** Develop automated tests for downstream service failures (timeouts, 5xx errors) to validate the application's resilience.
-   **[ ] Clarify Error Handling Requirements:** Engage with business analysts/product owners to define the expected behavior when one or more downstream services fail.

**Long-term:**
-   **[ ] Integrate Security Testing:** Once security requirements are defined, add test cases for authentication and authorization failures.
-   **[ ] Develop Performance Test Suite:** Create a load testing suite to validate the application against performance SLAs.

## Risk Assessment

-   **High Risk:** A failure in a downstream service could cause the entire `CreditApp` to fail if not handled gracefully, potentially impacting any upstream business process that relies on credit scores. The lack of any security makes the endpoint vulnerable.
-   **Medium Risk:** Incorrect data mapping or transformation in the XSLT could lead to invalid data being sent to or returned from the service. Performance bottlenecks under concurrent load could violate SLAs.
-   **Low Risk:** The logic is simple, reducing the risk of complex business rule failures. The absence of a database or persistent state minimizes data corruption risks within this specific module.