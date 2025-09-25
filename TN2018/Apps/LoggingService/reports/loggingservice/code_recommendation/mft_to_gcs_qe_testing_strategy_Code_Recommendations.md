## Testing and Validation Steps for GCS Migration

This document outlines the testing and validation steps required for migrating the `LoggingService` TIBCO application's file-writing functionality from a local file system to Google Cloud Storage (GCS).

### 1. Component & Integration Testing

This phase focuses on validating the refactored TIBCO process and its direct integration with GCS.

*   **GCS Connectivity & Authentication**:
    *   Test with a valid service account key/Workload Identity configuration.
    *   Test with invalid or expired credentials to verify authentication failure is handled gracefully.
    *   Test with a service account that has insufficient IAM permissions (e.g., no `storage.objectCreator` role) to ensure permission errors are caught and logged.
*   **File Operation Validation**:
    *   Trigger the process with `handler=file` and `formatter=text`. Verify a correctly named `.txt` object is created in GCS.
    *   Trigger the process with `handler=file` and `formatter=xml`. Verify a correctly named `.xml` object is created in GCS.
    *   Validate that the object names are correctly constructed based on the `loggerName` input, as per the logic in `LogProcess.bwp`.
*   **Error Handling & Retry Logic**:
    *   Simulate GCS API transient errors (e.g., 503 Service Unavailable) using a mock or fault injection proxy.
    *   Verify that the application handles GCS exceptions (e.g., `StorageException`) without crashing.
    *   Confirm that a failure to write to GCS is logged to a fallback mechanism (e.g., console logs) to prevent silent data loss.
*   **Configuration Validation**:
    *   Test that the process correctly reads the new GCS bucket name and project ID from module properties, replacing the legacy `fileDir` property.
    *   Verify the application fails gracefully if the GCS bucket configuration is missing or incorrect.

### 2. End-to-End & Performance Testing

This phase validates the business workflow and non-functional requirements.

*   **Business Workflow Validation**:
    *   Simulate an upstream application sending a log request to the TIBCO process.
    *   Verify the log object appears in GCS with the correct content and metadata.
    *   Validate the final `result` message ("Logging Done") is returned to the caller.
*   **Performance & Scalability Testing**:
    *   Establish a baseline by measuring the throughput and latency of the existing local file-writing implementation.
    *   Execute load tests against the GCS-integrated version, simulating realistic log volumes (e.g., 1000 logs/minute).
    *   Compare GCS latency and throughput against the baseline. Analyze any degradation and optimize if necessary (e.g., by adjusting GCS client settings).
*   **Data Integrity & Consistency Testing**:
    *   During load tests, programmatically download a sample of created GCS objects.
    *   Perform a byte-for-byte comparison of the downloaded object content against the original log message sent.
    *   Verify that object metadata (e.g., `Content-Type`) is set correctly.
*   **Security Validation**:
    *   Attempt to access the GCS bucket with an unauthorized service account to confirm access is denied.
    *   Review GCS bucket policies and IAM roles to ensure no public access is granted.
    *   Verify that Cloud Audit Logs are capturing all write operations for security auditing.

### 3. Production Validation & Monitoring

This phase ensures the migrated service is observable and supportable in production.

*   **Production Performance Monitoring**:
    *   After deployment, monitor GCS metrics in Cloud Monitoring (e.g., API error rates, latency for `storage.objects.create`).
    *   Trigger a test alert to confirm the notification channel is working.
*   **Data Quality Monitoring**:
    *   Implement and test a scheduled Cloud Function that periodically samples new log objects in GCS and validates their format and content.
*   **Rollback Procedure Validation**:
    *   In a pre-production environment, execute the rollback procedure, which involves redeploying the previous version of the TIBCO application that writes to the local file system.
    *   Verify that the application reverts to writing files locally without data loss for in-flight requests.

### Code Modifications for Testing

1.  **Update Test Cases:** Modify existing test cases to validate the new GCS integration. This includes updating assertions to check for the existence and content of log files in GCS instead of the local file system.
2.  **Mock GCS Service:** Create a mock GCS service to simulate GCS API errors and test the application's error handling logic.
3.  **Implement Performance Tests:** Implement performance tests to measure the throughput and latency of the GCS integration.
4.  **Implement Security Tests:** Implement security tests to verify that the GCS bucket is secured according to the principle of least privilege.

### Before/After Code Examples (Illustrative)

**Before (writing to local file system):**

```java
// Java code (example)
File logFile = new File(fileDir + "/" + loggerName + ".txt");
FileWriter fileWriter = new FileWriter(logFile, true);
fileWriter.write(message);
fileWriter.close();
```

**After (writing to GCS):**

```java
// Java code (example using Google Cloud Storage client library)
Storage storage = StorageOptions.getDefaultInstance().getService();
String bucketName = "your-gcs-bucket-name";
String objectName = loggerName + ".txt";
String content = message;
BlobId blobId = BlobId.of(bucketName, objectName);
BlobInfo blobInfo = BlobInfo.newBuilder(blobId).setContentType("text/plain").build();
Blob blob = storage.create(blobInfo, content.getBytes(UTF_8));
```

**Note:** The above code examples are illustrative and may need to be adapted to your specific environment and requirements.

### Dependencies and Configuration Updates

*   **Google Cloud Storage Client Library:** Add the Google Cloud Storage client library to the project dependencies.
*   **IAM Permissions:** Grant the TIBCO BusinessWorks application the necessary IAM permissions to write to the GCS bucket.
*   **Module Properties:** Update the `META-INF/default.substvar` file with the GCS bucket name and project ID.

### Risk Assessment and Mitigation

*   **Risk:** Data loss due to GCS write failures.
    *   **Mitigation:** Implement error handling logic to catch GCS-specific exceptions and log the errors to a fallback mechanism.
*   **Risk:** Performance degradation due to network latency.
    *   **Mitigation:** Benchmark the GCS implementation against the current file system implementation and optimize the GCS connector configuration.
*   **Risk:** Security vulnerabilities due to misconfigured IAM permissions.
    *   **Mitigation:** Grant the TIBCO BusinessWorks application only the necessary IAM permissions to write to the GCS bucket.

### Additional Notes and Considerations

*   Consider using a more unique naming scheme for the log files to prevent object overwrites.
*   Consider using a GCS lifecycle policy to automatically move the log files to Coldline storage after a certain period of time.

This document provides a high-level overview of the testing and validation steps required to migrate the `LoggingService` TIBCO application's file-writing functionality to GCS. You may need to adjust the specific steps based on your environment and requirements.
