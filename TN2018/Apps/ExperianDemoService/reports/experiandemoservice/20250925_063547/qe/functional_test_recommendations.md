## Executive Summary

This report provides a comprehensive functional testing strategy for the `ExperianService` TIBCO BusinessWorks (BW) application. The application exposes a single REST API endpoint (`/creditscore`) to retrieve credit score information based on a provided Social Security Number (SSN). The core logic involves parsing an incoming JSON request, querying a PostgreSQL database, and returning a JSON response.

The highest-risk area identified is the use of a `LIKE` clause in the SQL query for SSN lookup, which could lead to incorrect data being returned. The testing strategy prioritizes validation of this end-to-end data flow, including data accuracy, error handling, and input validation. We recommend a risk-based approach, focusing 70% of the effort on P0 (Critical) scenarios like correct data retrieval and handling of "not found" cases.

## Analysis

### Functional Testing Scope Analysis

Based on the codebase, the functional scope is a single, well-defined business process: retrieving a credit score.

*   **Business Function:** The application provides credit score details (`fiCOScore`, `rating`, `noOfInquiries`) for a person identified by their SSN.
*   **User Workflow:** This is a system-to-system workflow. A client system sends a POST request with a person's details, and the `ExperianService` returns their credit information.
*   **Business Rules:**
    *   **Input Validation:** The incoming request must be a JSON object containing `dob`, `firstName`, `lastName`, and `ssn`. This is defined in `ExperianService.module/Schemas/ExperianRequestSchema.xsd`.
    *   **Data Retrieval:** The system retrieves data from the `public.creditscore` table by matching the provided `ssn`.
    *   **Data Mapping:** The response is constructed by mapping database columns (`ficoscore`, `rating`, `numofpulls`) to JSON fields (`fiCOScore`, `rating`, `noOfInquiries`), as defined in `ExperianService.module/Processes/experianservice/module/Process.bwp`.
*   **API Functionality:** A single `POST /creditscore` endpoint is defined in `ExperianService.module/Service Descriptors/experianservice.module.Process-Creditscore.json`.
*   **Integration Points:**
    *   **Inbound REST:** An HTTP receiver listens for POST requests.
    *   **Outbound JDBC:** A JDBC connection is made to a PostgreSQL database, as configured in `ExperianService.module/Resources/experianservice/module/JDBCConnectionResource.jdbcResource`.

### Risk-Based Test Scenario Prioritization

Testing efforts should be prioritized based on business impact and technical risk.

*   **P0 Critical Business Workflows (70% of testing effort):**
    *   Validating the accuracy of credit data returned for a correct SSN.
    *   Ensuring no data is returned for an SSN that does not exist.
    *   Verifying that required input fields are enforced.
    *   Testing the high-risk `LIKE` query behavior to ensure it doesn't return data for a partial SSN match.

*   **P1 High-Impact User Workflows (25% of testing effort):**
    *   Validating system behavior when the database is unavailable.
    *   Testing the handling of malformed or incomplete JSON requests.
    *   Verifying correct mapping of all fields from the database to the JSON response.

*   **P2 Medium-Impact Features (5% of testing effort):**
    *   Testing the handling of database records with missing optional fields (e.g., `rating` is null).

*   **P3 Low-Impact Features (0% of testing effort):**
    *   The service is too simple to have any low-impact features that warrant dedicated functional testing cycles at this stage.

### Functional Test Case Development (Given/When/Then)

The following test cases are recommended, structured using the Gherkin syntax.

#### P0 Critical Business Workflows

```gherkin
# P0 Critical Path - Successful Credit Score Retrieval
Feature: Successful credit score retrieval

  Scenario: Retrieve credit score for an existing user
    Given the 'creditscore' table contains a record for SSN '***-**-1111' with FICO score 800, rating 'Excellent', and 2 inquiries
    When a POST request is made to '/creditscore' with the following valid JSON body:
      """
      {
        "dob": "1980-01-01",
        "firstName": "John",
        "lastName": "Doe",
        "ssn": "***-**-1111"
      }
      """
    Then the HTTP response status should be 200 (OK)
    And the response body should be a JSON object containing:
      """
      {
        "fiCOScore": 800,
        "rating": "Excellent",
        "noOfInquiries": 2
      }
      """
```

```gherkin
# P0 Failure Scenario - SSN Not Found
Feature: Handling of non-existent user

  Scenario: Attempt to retrieve score for an SSN that does not exist
    Given the 'creditscore' table does not contain any record for SSN '***-**-0000'
    When a POST request is made to '/creditscore' with SSN '***-**-0000'
    Then the HTTP response status should be 200 (OK)
    And the response body should be an empty JSON object '{}'
    # Note: This behavior is based on the current implementation. A business decision should be made whether a 404 or a more descriptive error is preferable.
```

```gherkin
# P0 Security/Functional Risk - Incorrect Data Retrieval via LIKE clause
Feature: SQL Query Specificity

  Scenario: Ensure partial SSN match does not return incorrect data
    Given the 'creditscore' table contains a record for SSN '***-**-1111' but not for '***-**-111'
    When a POST request is made to '/creditscore' with SSN '***-**-111'
    Then the HTTP response status should be 200 (OK)
    And the response body should be an empty JSON object '{}'
    # This test is critical to validate that the 'ssn like ?' query doesn't incorrectly match a longer SSN.
```

#### P1 High-Impact User Workflows

```gherkin
# P1 Failure Scenario - Invalid JSON Input
Feature: Robustness against malformed requests

  Scenario: Submit a request with a missing required field
    Given the service is running
    When a POST request is made to '/creditscore' with a JSON body missing the 'ssn' field
    Then the HTTP response status should be 400 (Bad Request) or another appropriate 4xx client error
    And the response body should contain an error message indicating 'ssn' is required.

  Scenario: Submit a request with malformed JSON
    Given the service is running
    When a POST request is made to '/creditscore' with the body '{"ssn": "***-**-1111"' (missing closing brace)
    Then the HTTP response status should be 400 (Bad Request)
    And the response body should contain an error message about JSON parsing failure.
```

```gherkin
# P1 Failure Scenario - Database Unavailability
Feature: System resilience during integration failure

  Scenario: Service handles database connection failure
    Given the PostgreSQL database is not accessible
    When a POST request is made to '/creditscore' with any valid JSON body
    Then the HTTP response status should be 500 (Internal Server Error) or 503 (Service Unavailable)
    And the response body should contain a generic error message indicating a system issue.
```

#### P2 Medium-Impact Features

```gherkin
# P2 Data Mapping - Handling of Null/Empty DB Fields
Feature: Graceful handling of incomplete data

  Scenario: Retrieve a record with a null 'rating' field
    Given the 'creditscore' table contains a record for SSN '***-**-2222' where the 'rating' column is NULL
    When a POST request is made to '/creditscore' with SSN '***-**-2222'
    Then the HTTP response status should be 200 (OK)
    And the response JSON should contain 'fiCOScore' and 'noOfInquiries' but NOT the 'rating' field.
    # This validates the 'xsl:if' logic in the RenderJSON activity in Process.bwp.
```

### Functional Test Implementation Recommendations

*   **API Testing Framework:** Use a tool like **Postman**, **REST Assured** (for Java), or **Karate** to automate the API test cases. This allows for easy assertion of HTTP status codes and JSON response bodies.
*   **Test Data Management:**
    *   A dedicated test database schema should be created.
    *   Use database seeding scripts (e.g., SQL scripts) to populate the `creditscore` table with a known set of data before each test run. This ensures tests are repeatable and not dependent on existing state.
    *   The test data should include the scenarios outlined above: existing users, users with partial data, and boundary conditions.
*   **Test Environment:**
    *   Testing requires a running TIBCO BW application instance connected to a PostgreSQL database instance.
    *   For the "Database Unavailability" test, the test environment should allow for controllably stopping/blocking access to the database.

## Evidence Summary

*   **Scope Analyzed**: The analysis covered the `ExperianService` and `ExperianService.module` TIBCO projects.
*   **Key Files Analyzed**:
    *   `ExperianService.module/Processes/experianservice/module/Process.bwp`: Defined the core business logic and process flow.
    *   `ExperianService.module/Service Descriptors/experianservice.module.Process-Creditscore.json`: Defined the REST API contract.
    *   `ExperianService.module/Schemas/*.xsd`: Defined the request and response data schemas.
    *   `ExperianService.module/Resources/experianservice/module/JDBCConnectionResource.jdbcResource`: Provided database connection details.
*   **Key Data Points**:
    *   1 primary business workflow identified.
    *   1 inbound REST API endpoint (`POST /creditscore`).
    *   1 outbound JDBC integration.

## Assumptions Made

*   The TIBCO application is deployed in an environment where it can access the PostgreSQL database configured at `localhost:5432`.
*   The `public.creditscore` table exists in the `bookstore` database with the columns `firstname`, `lastname`, `ssn`, `dateofBirth`, `ficoscore`, `rating`, and `numofpulls`.
*   The use of `ssn like ?` in the JDBC query is intentional and not a typo for `ssn = ?`. This assumption is the source of a major identified risk.

## Open Questions

1.  **Incorrect Data Risk:** The SQL query `SELECT * FROM public.creditscore where ssn like ?` is a significant risk. If one person's SSN is a substring of another's, the query could return the wrong credit score. Is this intended behavior? We strongly recommend changing this to an exact match (`ssn = ?`).
2.  **"Not Found" Response:** The current implementation returns an HTTP 200 with an empty JSON object `{}` when an SSN is not found. Is this the desired behavior for client applications? A `404 Not Found` status or a `200 OK` with a descriptive message might be more explicit.
3.  **Multiple Matches:** What is the expected behavior if the `LIKE` query returns multiple records? The current implementation will only process the first record (`$JDBCQuery/Record[1]`), which is non-deterministic and likely incorrect.

## Confidence Level

**Overall Confidence**: High

**Rationale**: The project is small, self-contained, and follows a standard TIBCO process pattern. The logic is linear and easy to trace from the HTTP request to the JDBC query and back to the HTTP response. The primary risks and functional gaps are clearly identifiable from the implementation in `Process.bwp`.

## Action Items

**Immediate** (Next 1-3 days):
*   **[ ] Clarify Open Questions:** Discuss the "LIKE query" and "Not Found response" issues with the development team and product owner to confirm expected behavior.
*   **[ ] Implement P0 Test Cases:** Automate the happy path and "not found" scenarios using an API testing tool.

**Short-term** (Next Sprint):
*   **[ ] Implement P1 Test Cases:** Automate tests for invalid JSON input and database unavailability scenarios.
*   **[ ] Set Up Test Data:** Create and version-control SQL scripts to set up and tear down test data for repeatable test runs.

**Long-term** (Next 1-2 Sprints):
*   **[ ] Integrate Tests into CI/CD:** Add the automated API test suite to the continuous integration pipeline to run on every build.
*   **[ ] Address Design Flaws:** Based on the outcome of the "Open Questions", work with development to fix the potential data correctness issues.

## Risk Assessment

*   **High Risk**:
    *   **Incorrect Data Retrieval:** The use of `LIKE` on an SSN field could cause the service to return credit information for the wrong person. This is a critical data privacy and functional correctness failure.
*   **Medium Risk**:
    *   **Ambiguous "Not Found" Response:** Returning a 200 OK with an empty body for a non-existent user can lead to misinterpretation by client systems, which may not distinguish it from a successful call that simply had no data to return.
*   **Low Risk**:
    *   **Unhandled Database Errors:** While a generic 5xx error will likely be returned, the lack of specific error handling for different database exceptions (e.g., timeout vs. constraint violation) could make troubleshooting difficult.