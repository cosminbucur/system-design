# Storing Large Files in Microservices with Spring

Storing large files (such as images, videos, PDFs, or archives) within a microservices architecture requires a different strategy than a traditional monolithic application. Storing files directly in a database or on a service's local disk quickly introduces bottlenecks, scaling issues, and data consistency headaches.

This guide outlines the standard, production-proven approach to handling large file storage using the Spring ecosystem.

---

## 1. The Core Strategy: Off-Store the Binary Data

Never store large binary files (BLOBs) directly inside your relational database or inside the microservice's local file system. 

* **The Database:** Stores only **metadata** (file name, file size, content type, upload timestamp, owner ID, and the storage path or URI).
* **Object Storage:** Stores the actual binary file. 

### Recommended Object Storage Solutions:
* **Cloud-native:** Amazon S3, Google Cloud Storage (GCS), or Azure Blob Storage.
* **On-Premise / Self-hosted:** MinIO (S3-compatible, open-source, and extremely popular for development and Kubernetes environments).

---

## 2. Implementation Patterns in Spring

Depending on your security and performance requirements, you can choose between two primary upload/download flows:

### Pattern A: Direct-to-Storage via Presigned URLs (Recommended)
Routing large files through your microservices wastes valuable network bandwidth and memory (heap/off-heap). Instead, use **Presigned URLs**.

1. **Client** requests an upload token/URL from the `File Service`.
2. **File Service** validates user permissions and generates a time-limited **Presigned Upload URL** using the cloud storage SDK.
3. **File Service** returns the URL to the client.
4. **Client** uploads the file **directly** to Object Storage (S3/MinIO) using that URL.
5. **Client** notifies the `File Service` that the upload is complete, and the service saves the file metadata to the database.

### Pattern B: Gateway / Service Proxy (For Smaller Files)
If files are relatively small or strict corporate firewalls prevent direct client-to-storage communication, you can stream through a Spring Boot service using Spring WebFlux or Spring MVC.

* **Spring Cloud Gateway** or a dedicated microservice accepts a `MultipartFile`.
* The service streams the file chunks directly to the object storage bucket rather than loading the entire file into memory.

---

## 3. Spring Boot Integration Example

Spring provides great abstractions for cloud storage through standard S3/MinIO SDKs. Below is a clean implementation using the official AWS S3 v2 SDK in a Spring Boot service.

### Maven Dependencies (`pom.xml`)
```xml
<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- AWS S3 SDK v2 -->
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>s3</artifactId>
        <version>2.25.50</version>
    </dependency>
</dependencies>
```

### Configuration (`application.yml`)
```yaml
aws:
  s3:
    region: us-east-1
    bucket-name: my-microservice-files-bucket
    endpoint: http://localhost:9000 # Useful if using MinIO locally
```

### Storage Service Component
```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;
import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.PutObjectRequest;

import java.io.IOException;
import java.util.UUID;

@Service
public class S3StorageService {

    private final S3Client s3Client;

    @Value("${aws.s3.bucket-name}")
    private String bucketName;

    public S3StorageService(S3Client s3Client) {
        this.s3Client = s3Client;
    }

    public String uploadFile(MultipartFile file) throws IOException {
        String fileName = UUID.randomUUID() + "_" + file.getOriginalFilename();

        PutObjectRequest putObjectRequest = PutObjectRequest.builder()
                .bucket(bucketName)
                .key(fileName)
                .contentType(file.getContentType())
                .build();

        s3Client.putObject(putObjectRequest, 
                RequestBody.fromInputStream(file.getInputStream(), file.getSize()));

        // Return the stored file key or public URI to be saved in your database metadata
        return fileName;
    }
}
```

---

## 4. Microservice Considerations

* **Asynchronous Processing:** If large files require post-processing (e.g., video transcoding, image resizing, virus scanning), have the upload service emit an event to a message broker like **Apache Kafka** or **RabbitMQ**. A separate worker microservice can consume the event, fetch the file from object storage, process it, and update its status.
* **Distributed Tracing:** Ensure you pass correlation IDs (via Micrometer Tracing) so you can track a file upload request across the API Gateway, File Service, and asynchronous workers.
* **Security & Access Control:** Never expose object storage buckets publicly by default. Use private buckets and serve downloads either via authenticated backend endpoints or short-lived presigned download URLs.