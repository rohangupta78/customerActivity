**Real-Time Customer Activity Stream Processor**
Project Overview
This project demonstrates a foundational Real-Time Customer Data Platform (RTCDP) pipeline. It simulates the ingestion, real-time processing, and analytical querying of customer activity events (e.g., page views, add-to-carts, purchases). Built with modern distributed technologies, it showcases robust backend development, event-driven architecture, stream processing, and data persistence, making it highly relevant to roles in data-intensive product organizations.

The system is designed to handle a continuous stream of events, providing insights into customer behavior with minimal latency.

**Features**
Event Generation: Simulates continuous customer activity events (e.g., page views, purchases) via a Spring Boot application.

Real-time Ingestion: Uses Apache Kafka as a high-throughput, fault-tolerant message broker for event ingestion.

Stream Processing: Leverages Apache Spark Structured Streaming to consume raw events, perform windowed aggregations (e.g., page view counts per user per time window), and transform data in real-time.

Data Persistence: Stores processed and aggregated customer insights in a MongoDB database (easily swappable with PostgreSQL).

Analytics API: Provides a Spring Boot REST API for querying and retrieving processed customer activity data, enabling real-time analytics.

Containerization: Uses Docker Compose for easy setup and management of all infrastructure components (Kafka, Zookeeper, MongoDB).

**Tech Stack**
Backend: Java 11+, Spring Boot

Messaging: Apache Kafka

Stream Processing: Apache Spark Structured Streaming (with Java API)

Database: MongoDB (or PostgreSQL)

Build Tools: Maven

Containerization: Docker, Docker Compose

Development Tools: IntelliJ IDEA / VS Code

**High-Level Design / Architecture**
The system comprises three main applications interacting via Apache Kafka and persisting data to a database.

+-------------------+       +-----------------+       +-----------------+
| Event Generator   |------>| Apache Kafka    |------>| Spark Stream    |
| (Spring Boot)     |       | (raw_events)    |       | Processor       |
+-------------------+       +-----------------+       | (Java/Spark SS) |
        (Simulates Customer Events)                     +--------+--------+
                                                                 |
                                                                 | Writes Processed Data
                                                                 v
                                                        +--------+--------+
                                                        | MongoDB         |
                                                        | (user_page_views)|
                                                        +--------+--------+
                                                                 ^
                                                                 | Queries Processed Data
                                                                 |
                                                        +-------------------+
                                                        | Analytics Service |
                                                        | (Spring Boot REST)|
                                                        +-------------------+

**Component Roles:**

Event Generator (Spring Boot): Acts as the source of synthetic customer activity. It generates JSON-formatted events (e.g., PAGE_VIEW, PURCHASE) at regular intervals and publishes them to a Kafka topic. This simulates web/mobile client activity.

Apache Kafka: Serves as the central nervous system for data flow. Raw customer events are buffered in Kafka, providing decoupling and fault tolerance between the event producer and the stream processor.

Spark Stream Processor (Java/Spark Structured Streaming): This is the core real-time processing engine. It continuously consumes raw events from Kafka, applies transformations (like windowed counts of page views per user), and writes the aggregated results to MongoDB. It uses Spark's Structured Streaming API for fault-tolerant, exactly-once processing.

MongoDB: The persistence layer for the processed and aggregated customer activity data. It stores the output from the Spark processor in a flexible document format.

Analytics Service (Spring Boot REST): Exposes a RESTful API to query the aggregated customer data stored in MongoDB. This service allows other applications (or a UI) to retrieve real-time insights like "recent page views" or "page views by a specific user."

**Low-Level Design / Component Details**
1. Event Generator (Spring Boot)
CustomerActivityEvent POJO: Defines the structure of the customer event (e.g., userId, sessionId, timestamp, eventType, productId, pageUrl, value).

KafkaProducerService: A Spring @Service that encapsulates the logic for sending CustomerActivityEvent objects to a pre-configured Kafka topic (raw_customer_events) using KafkaTemplate.

EventGeneratorScheduler: A Spring @Component with @Scheduled annotation. It periodically generates random CustomerActivityEvent instances and uses KafkaProducerService to send them, simulating continuous traffic. Configurable via application.properties.

2. Spark Stream Processor (Java)
Input (Kafka): Reads raw event messages from the raw_customer_events Kafka topic. The value field (containing the JSON event) is cast to STRING.

Deserialization & Mapping: Uses a MapFunction with Jackson's ObjectMapper to parse the incoming JSON strings into CustomerActivityEvent Java objects (POJOs), making them strongly typed for Spark's operations.

Transformations:

Filtering: Filters events based on eventType (e.g., only "PAGE_VIEW" events for initial aggregation).

Watermarking: Implemented on the timestamp field to handle late-arriving data and ensure correct windowing.

Windowed Aggregation: Groups events into tumbling or sliding time windows (e.g., 1-minute windows, updating every 30 seconds) and aggregates metrics (e.g., count of events) per userId within each window.

Output (MongoDB): The processed DataFrame (containing userId, window_start, window_end, page_view_count) is written to a MongoDB collection (user_page_view_counts) using the mongo format sink. outputMode("update") is used for aggregations that update continuously.

Checkpointing: Crucial for fault tolerance in Structured Streaming. A checkpointLocation is configured to store offsets and metadata, allowing the stream to resume from where it left off after failures.

3. Analytics Service (Spring Boot REST)
Data Model (UserPageViewCount Document): Represents the structure of the processed data stored in MongoDB, mapping directly to the documents written by Spark. Uses Spring Data MongoDB annotations (@Document, @Id).

MongoRepository: A Spring Data MongoDB interface (UserPageViewCountRepository) extends MongoRepository, providing out-of-the-box CRUD operations and query methods (e.g., findByUserId, findByWindowEndAfter) for easy data access.

AnalyticsController: A Spring @RestController exposing REST endpoints.

/api/analytics/user/{userId}/page-views: Retrieves all page view counts for a specific user.

/api/analytics/recent-page-views: Fetches aggregated page view counts within a recent time window (e.g., last 5 minutes), demonstrating near real-time insights.

**Setup and Run Instructions**
To get this project up and running locally, follow these steps:

**Prerequisites**
Java Development Kit (JDK) 11 or higher

Apache Maven

Docker Desktop (for Kafka, Zookeeper, MongoDB)

Apache Spark (Optional for local execution): While the project runs spark-submit --master local[*], having a local Spark installation can be helpful for debugging if not using an IDE's direct run.

1. Clone the Repository
Bash

git clone https://github.com/rohangupta78/customerActivity.git
cd customerActivity
2. Start Infrastructure with Docker Compose
This will spin up Kafka, Zookeeper, and MongoDB.

Bash

docker-compose up -d
Verify all containers are running: docker-compose ps.

3. Build and Run Applications
Open three separate terminal windows (or tabs) for each application.

a) Event Generator

Bash

cd event-generator
mvn spring-boot:run
(You should see "Sent event: ..." logs indicating events being sent to Kafka.)

b) Spark Stream Processor

Bash

cd stream-processor
mvn clean package # Builds the runnable JAR
spark-submit --class com.example.rtcdp.streamprocessor.CustomerActivityStreamProcessor \
             --master local[*] \
             target/stream-processor-1.0-SNAPSHOT-jar-with-dependencies.jar
(You'll see Spark logs, and messages like "Writing batch X to DB" as it processes and saves data to MongoDB.)

c) Analytics Service

Bash

cd analytics-service
mvn spring-boot:run
(This service will start on port 8081.)

4. Test Analytics API Endpoints
Once all services are running, you can query the analytics API:

Get recent page view counts:

Bash

curl http://localhost:8081/api/analytics/recent-page-views
Get page view counts for a specific user (e.g., user_A):

Bash

curl http://localhost:8081/api/analytics/user/user_A/page-views
(Note: User IDs are user_A through user_E as per generator logic.)

**Challenges & Learnings**
Environment Setup: Managing multiple interconnected services (Kafka, Spark, Spring Boot, DB) locally via Docker Compose.

Spark Structured Streaming: Understanding event-time processing, watermarking, windowed aggregations, and writing to external databases.

Data Serialization/Deserialization: Handling JSON data across Spring Boot, Kafka, and Spark.

Fault Tolerance & Checkpointing: Implementing Spark's checkpointing for reliable stream processing.

Distributed Systems Thinking: Designing components for independent scaling and communication via message queues.

Database Integration: Integrating Spring Data MongoDB with Spark's write capabilities.

**Future Enhancements**
More Complex Spark Analytics:

Implement sessionization (grouping events into user sessions).

Calculate user funnels (e.g., view -> add-to-cart -> purchase).

Real-time anomaly detection for unusual activity.

Stream-to-stream joins for enriching event data.

Personalized Recommendations: Develop a simple real-time recommendation engine based on user activity.

User Interface (UI): Build a simple dashboard (e.g., with React, Angular, or even a simple Thymeleaf template) to visualize the analytics.

Error Handling & Monitoring: Add more robust error handling, metrics (Prometheus/Grafana), and tracing.

Containerization of Spark: Dockerize the Spark application for easier deployment.

Security: Implement basic authentication/authorization for the Analytics API.

Advanced Kafka Features: Explore Kafka Streams API for simpler stream processing needs.

**Contributing**
Feel free to fork this repository, submit issues, or create pull requests. All contributions are welcome!
