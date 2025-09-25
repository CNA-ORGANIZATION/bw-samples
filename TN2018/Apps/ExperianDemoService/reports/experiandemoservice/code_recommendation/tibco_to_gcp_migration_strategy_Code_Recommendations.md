## Migration Overview (from original report)
The report outlines a migration strategy for the `ExperianService` TIBCO BusinessWorks (BW) application to the Google Cloud Platform (GCP). The project consists of one TIBCO BW 6.5.0 module, which exposes a single REST endpoint to fetch credit score data from a PostgreSQL database. The migration complexity is **Low** due to the simple, stateless nature of the process. The recommended approach is to refactor the TIBCO process into a serverless Cloud Run service, fronted by an API Gateway, and migrate the PostgreSQL database to AlloyDB.

## Affected Source Files Analysis
- **ExperianService.module/Processes/experianservice/module/Process.bwp**: This file defines the entire business logic for the service. It orchestrates the flow of receiving an HTTP request, parsing the data, querying a database, formatting the response, and sending it back.
- **ExperianService.module/Resources/experianservice/module/JDBCConnectionResource.jdbcResource**: This file configures the database connection, pointing to a local PostgreSQL instance.
- **ExperianService.module/Service Descriptors/experianservice.module.Process-Creditscore.json**: This file defines the REST API contract for the `/creditscore` endpoint, including the `POST` method, request body structure, and response body structure.
- **ExperianService.module/Resources/experianservice/module/Creditscore.httpConnResource**: This file configures the service to listen on port 7080 on the host defined by the `BW.HOST.NAME` variable (which defaults to `localhost`).

## Specific Code Changes Required
1.  **TIBCO Process Migration**: The logic within `Process.bwp` (receive request, parse JSON, query DB, render JSON) will be rewritten as a lightweight Java web service (e.g., using Spring Boot) and deployed as a Cloud Run service.
2.  **Database Call Updates**: The JDBC connection logic will be updated to connect to AlloyDB. The existing SQL query (`SELECT * FROM public.creditscore where ssn like ?`) will be refactored to the more performant `SELECT ficoscore, rating, numofpulls FROM public.creditscore WHERE ssn = ?`.
3.  **Endpoint Configuration**: The current TIBCO HTTP Connector configuration will be replaced by GCP's API Gateway to manage the public-facing endpoint.
4.  **Security**: Implement API Key validation using GCP API Gateway.

## Step-by-Step Implementation Guide
1.  **Create a new Java Spring Boot application**.
2.  **Implement the REST endpoint**:
    ```java
    @RestController
    public class CreditScoreController {

        @Autowired
        private CreditScoreService creditScoreService;

        @PostMapping("/creditscore")
        public CreditScoreResponse getCreditScore(@RequestBody CreditScoreRequest request) {
            return creditScoreService.getCreditScore(request.getSsn());
        }
    }
    ```
3.  **Implement the CreditScoreService**:
    ```java
    @Service
    public class CreditScoreService {

        @Autowired
        private JdbcTemplate jdbcTemplate;

        public CreditScoreResponse getCreditScore(String ssn) {
            String sql = "SELECT ficoscore, rating, numofpulls FROM public.creditscore WHERE ssn = ?";
            return jdbcTemplate.queryForObject(sql, new Object[]{ssn}, (rs, rowNum) ->
                    new CreditScoreResponse(rs.getInt("ficoscore"), rs.getString("rating"), rs.getInt("numofpulls")));
        }
    }
    ```
4.  **Configure the JDBC connection to AlloyDB**:
    ```java
    spring.datasource.url=jdbc:postgresql://<alloydb-instance-address>/bookstore
    spring.datasource.username=bwuser
    spring.datasource.password=<alloydb-password>
    spring.datasource.driver-class-name=org.postgresql.Driver
    ```
5.  **Create a Dockerfile to containerize the application**:
    ```dockerfile
    FROM openjdk:17-jdk-slim
    COPY target/*.jar app.jar
    ENTRYPOINT ["java", "-jar", "app.jar"]
    ```
6.  **Deploy the application to Cloud Run**.
7.  **Configure GCP API Gateway** to expose the Cloud Run service and require an API Key.
8.  **Store the AlloyDB password in GCP Secret Manager** and grant the Cloud Run service account access to it.

## Before/After Code Examples
**Before (TIBCO Process.bwp)**:
(This is a visual representation in TIBCO Business Studio, so a direct code comparison isn't applicable. The logic involves activities for receiving an HTTP request, parsing JSON, querying a database, and rendering JSON.)

**After (Java Spring Boot)**:
See code snippets above.

## Dependencies and Configuration Updates
-   **Dependencies**:
    -   Java 17
    -   Spring Boot
    -   PostgreSQL JDBC driver
    -   GCP Cloud Run
    -   GCP API Gateway
    -   GCP Secret Manager
    -   AlloyDB for PostgreSQL
-   **Configuration Updates**:
    -   Update the JDBC connection string to point to the AlloyDB instance.
    -   Configure GCP API Gateway to require an API Key for all requests.
    -   Store the AlloyDB password in GCP Secret Manager.

## Testing and Validation Steps
1.  **Component Testing**: Test the individual components of the new service (REST endpoint, database query) in isolation.
2.  **Integration Testing**: Test the integration between the new service and AlloyDB.
3.  **End-to-End Testing**: Test the entire workflow from the client to the database and back.
4.  **Performance Testing**: Load test the new service to ensure it meets the performance requirements.
5.  **Security Testing**: Verify that the API Key validation is working correctly.

## Risk Assessment and Mitigation
-   **Risk**: Data loss during database migration.
    -   **Mitigation**: Use GCP's Database Migration Service (DMS) for a low-downtime migration.
-   **Risk**: Performance degradation.
    -   **Mitigation**: Optimize the SQL query and load test the new service.
-   **Risk**: Security vulnerability due to missing authentication.
    -   **Mitigation**: Implement API Key validation using GCP API Gateway.

## Additional Notes and Considerations
-   Consider using a more robust authentication mechanism than API Keys, such as OAuth 2.0.
-   Implement proper error handling and logging in the new service.
-   Monitor the performance of the new service in production.
