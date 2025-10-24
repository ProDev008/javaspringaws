# javaspringaws
Java Spring AWS


Java 21 , Spring boot , microservices with AWS design

Focused, ready-to-use opinionated starter template and design checklist for building Java 21 + Spring Boot microservices with an AWS deployment architecture. It includes project layout, essential pom/build snippets, example code for one service (Employee), common cross-cutting concerns (API standards, error handling, metrics, resilience), Docker + docker-compose for local dev, and practical AWS design recommendations (ECS/Fargate, RDS, Secrets Manager, ALB, observability). Use it as a copy-pasteable starting point and adapt to your org conventions.

I kept this compact but complete — copy snippets into files and you’ll have a runnable base.



Multiple small services (e.g., employee-service, department-service, auth-service, gateway, config-server, discovery-server, monitoring).


API Gateway (Spring Cloud Gateway) + Centralized Config Server (Spring Cloud Config) + Service Discovery (Eureka).


Observability: Spring Boot Actuator + Micrometer (Prometheus) + Grafana.


Resilience: Resilience4j (circuit breaker, retry, rate-limiter).


Auth: OAuth2 / OIDC (Cognito or external IdP) or JWT-based.


Data: RDS (Postgres) per microservice; S3 for artifacts; Secrets Manager/Parameter Store for secrets.


Deployment: Build images -> push to ECR -> deploy with ECS Fargate behind ALB (or EKS if you prefer k8s).


CI/CD: GitHub Actions, CodeBuild/CodePipeline, or Jenkins; use infra-as-code (Terraform / CloudFormation).



Project layout (maven multi-module)



microservices-root/
├─ pom.xml                        # parent pom
├─ config/                         # spring-cloud-config server
├─ discovery/                      # eureka server
├─ gateway/                         # spring-cloud-gateway
├─ employee-service/               # example service
├─ department-service/
├─ common/                         # shared DTOs / libs (optional)
├─ monitoring/                      # prometheus rules, grafana dashboards
└─ infra/                           # terraform / cloudformation


Parent pom.xml (core deps & plugin management)

<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>microservices-parent</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>pom</packaging>
  <modules>
    <module>employee-service</module>
    <module>department-service</module>
    <module>gateway</module>
    <module>config</module>
    <module>discovery</module>
  </modules>

  <properties>
    <java.version>21</java.version>
    <spring.boot.version>3.2.0</spring.boot.version> <!-- adjust to latest -->
    <spring-cloud.version>2023.0.4</spring-cloud.version>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>${spring.boot.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>${spring-cloud.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <build>
    <pluginManagement>
      <plugins>
        <!-- maven-compiler, spring-boot-maven-plugin, dockerfile-maven-plugin etc -->
      </plugins>
    </pluginManagement>
  </build>
</project>



Employee entity (JPA)
package com.example.employee.entity;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "employees")
public class Employee {
  @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;
  private String firstName;
  private String lastName;
  private String email;
  private String departmentId;
  private Instant createdAt = Instant.now();
  // getters/setters/constructors...
}


package com.example.employee.repo;
import org.springframework.data.jpa.repository.JpaRepository;
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
}



package com.example.employee.dto;
public record EmployeeDTO(Long id, String firstName, String lastName, String email, String departmentId) {}


package com.example.employee.service;
import com.example.employee.repo.EmployeeRepository;
import com.example.employee.entity.Employee;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class EmployeeService {
  private final EmployeeRepository repo;
  public EmployeeService(EmployeeRepository repo){this.repo = repo;}

  public Employee create(Employee e){ return repo.save(e); }
  public List<Employee> list(){ return repo.findAll(); }
}




package com.example.employee.api;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import com.example.employee.service.EmployeeService;
import com.example.employee.entity.Employee;
import java.net.URI;
import java.util.List;

@RestController
@RequestMapping("/api/v1/employees")
public class EmployeeController {
  private final EmployeeService service;
  public EmployeeController(EmployeeService service){this.service = service;}

  @GetMapping
  public ResponseEntity<List<Employee>> list(){ return ResponseEntity.ok(service.list()); }

  @PostMapping
  public ResponseEntity<Employee> create(@RequestBody Employee e){
    Employee created = service.create(e);
    return ResponseEntity.created(URI.create("/api/v1/employees/"+created.getId())).body(created);
  }

  @GetMapping("/{id}")
  public ResponseEntity<Employee> get(@PathVariable Long id){
    return service.findById(id).map(ResponseEntity::ok)

           .orElse(ResponseEntity.notFound().build());
  }
}



Add findById to EmployeeService if used.


API standards & documentation

All APIs under /api/v1/... (versioning).


Use request/response DTOs; avoid exposing JPA entities directly (example uses simple mapping — adapt for production).


Errors: always return structured Problem JSON (RFC7807) or a simple structure:



{
  "timestamp":"2025-10-19T16:00:00Z",
  "status":404,
  "error":"Not Found",
  "message":"Employee not found",
  "path":"/api/v1/employees/123"
}

Add springdoc OpenAPI starter for auto-generated Swagger UI at /swagger-ui.html or /swagger-ui/index.html.




Global Error Handling (ControllerAdvice)

@RestControllerAdvice
public class ApiExceptionHandler {

  @ExceptionHandler(EntityNotFoundException.class)
  public ResponseEntity<Map<String,Object>> handleNotFound(EntityNotFoundException ex, HttpServletRequest req) {
    Map<String,Object> body = Map.of(
      "timestamp", Instant.now(),
      "status", 404,
      "error", "Not Found",
      "message", ex.getMessage(),
      "path", req.getRequestURI()
    );
    return ResponseEntity.status(404).body(body);
  }

  @ExceptionHandler(Exception.class)
  public ResponseEntity<Map<String,Object>> handleAny(Exception ex, HttpServletRequest req) {
    Map<String,Object> body = Map.of(
      "timestamp", Instant.now(),
      "status", 500,
      "error", "Internal Server Error",
      "message", "An unexpected error occurred",
      "path", req.getRequestURI()
    );
    return ResponseEntity.status(500).body(body);
  }
}





Observability & Metrics

Add Actuator endpoints in application.yml:




management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics,env,loggers,threaddump
  endpoint:
    health:
      show-details: always
management.metrics.export.prometheus:
  enabled: true



Micrometer + micrometer-registry-prometheus enabled -> Prometheus scrapes /actuator/prometheus.


Add custom metrics via MeterRegistry in services (timers, counters).




Resilience (Resilience4j) example

application.yml:


resilience4j:
  circuitbreaker:
    configs:
      default:
        slidingWindowType: TIME_BASED
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
  retry:
    configs:
      default:
        maxAttempts: 3
        waitDuration: 1s



Use annotation:

@CircuitBreaker(name = "employeeService", fallbackMethod = "fallbackEmployees")
public List<Employee> list() { ... }

public List<Employee> fallbackEmployees(Throwable t) {
  // degrade gracefully
  return List.of();
}



Security (JWT example high-level)

Validate incoming JWTs at gateway or service level.


Use Spring Security with OAuth2 Resource Server:


spring:

  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: https://cognito-idp.{region}.amazonaws.com/{pool-id}/.well-known/jwks.json



Dockerfile (service)

FROM eclipse-temurin:21-jdk-jammy
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
EXPOSE 8080
ENTRYPOINT ["java","-XX:+UseContainerSupport","-Xms512m","-Xmx512m","-jar","/app.jar"]



docker-compose.yml (local dev quick composition)

version: '3.8'
services:
  discovery:
    image: your-eureka-image
    ports: ["8761:8761"]
  config:
    image: your-config-image
    ports: ["8888:8888"]
  employee-service:
    build: ./employee-service
    environment:
      - SPRING_PROFILES_ACTIVE=local
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/employees
    depends_on: [db, discovery, config]
    ports: ["8081:8080"]
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: employees
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    ports: ["5432:5432"]
  prometheus:
    image: prom/prometheus
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    ports: ["9090:9090"]



Local run & build

Build a service:


mvn -pl employee-service -am clean package

docker build -t myorg/employee-service:0.0.1 ./employee-service
docker-compose up --build


WS deployment (practical checklist)

Container registry: push images to ECR.


Compute: deploy to ECS Fargate (or EKS if k8s).


Networking: ALB in front, private subnets for tasks; use target groups for services.


Discovery: use AWS Cloud Map if you prefer AWS-native discovery; otherwise keep Eureka (but ensure cluster placement and state).


DB: RDS (Postgres) with multi-AZ. Each microservice has its own schema/DB.


Secrets: AWS Secrets Manager for DB credentials and OAuth secrets.


Config: Use Spring Cloud Config backed by Git in CodeCommit or use AWS SSM Parameter Store.


Logging: Ship logs to CloudWatch Logs (sidecar or FireLens), use structured JSON logs.


Tracing: Use AWS X-Ray or OpenTelemetry -> X-Ray exporter.


Metrics: Continue Prometheus (hosted on EC2/EKS) or use CloudWatch metrics + Container Insights; for Prometheus on ECS use a Prometheus server scraping /actuator/prometheus (consider a Service Discovery relabel config).


IAM: Fine-grained IAM roles for tasks (task role) to grant S3/RDS/SecretsManager access.


CI/CD: Build image in CodeBuild or GitHub Actions -> push to ECR -> deploy via ECS task update or CodeDeploy Blue/Green.




Terraform snippet (very short)

resource "aws_ecr_repository" "employee" {
  name = "employee-service"
}

resource "aws_ecs_cluster" "main" {
  name = "microservices-cluster"
}
# add task definitions, iam roles, alb, ecs_service etc...



Testing & Quality

Unit tests: JUnit 5, Mockito.


Integration tests: Testcontainers (start postgres).


Contract tests: Pact or Spring Cloud Contract.


Code quality: SpotBugs, Checkstyle, JaCoCo for coverage.


End-to-end: run docker-compose with real DB and call endpoints.




Minimal Checklist before production

Secrets moved to Secrets Manager; not in config files.


Health endpoints secured for internal access only.


Circuit Breakers + bulkheads configured for remote calls.


DB connection pool tuned (HikariCP).


Resource limits set on container tasks.


Observability: traces + metrics + logs wired.


Automated backups for DB; alerts for unhealthy services.


Security: TLS enforced on ALB; validate tokens at gateway.




Quick example: application.yml (employee-service)

spring:
  application:
    name: employee-service
  datasource:
    url: ${JDBC_DATABASE_URL:jdbc:postgresql://localhost:5432/employees}
    username: ${JDBC_DATABASE_USERNAME:postgres}
    password: ${JDBC_DATABASE_PASSWORD:password}
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false

eureka:
  client:
    enabled: true
    service-url:
      defaultZone: http://localhost:8761/eureka
management:
  endpoints:
    web:
      exposure:
        include: "*"


Example of things to copy into your repo right now

Parent pom.xml (above)


employee-service code files (entity, repo, service, controller, ControllerAdvice)


Dockerfile + docker-compose


application.yml (profile local) and application-production.yml referencing secrets


Prometheus scrape config for actuator/prometheus





In connection Agentic AI to the Java 21 + Spring Boot + Microservices + AWS architecture outlined above, and define it in a precise, system-engineering way.


Agentic AI refers to AI systems that can act autonomously and proactively toward goals — not just respond passively — by perceiving environment data, reasoning about state and context, and executing actions (API calls, workflows, or service operations) to achieve objectives.



Agentic AI = Autonomous + Goal-Driven + Action-Capable + Observable


Derived from the above microservice stack

Using your described Java 21 + Spring Boot + AWS setup, here’s how an Agentic AI definition naturally emerges:



System Layer
How It Enables “Agency”

Spring Boot microservices
 Provide modular, observable action endpoints (e.g., /notify, /analyzeRisk, /triggerBackup). Each microservice represents a controllable capability.
 
API Gateway + Service Discovery
 Give the AI layer dynamic visibility and routing — the agent can discover and call any service as needed.
 
Event-Driven Messaging (Kafka/SQS)
 Allows asynchronous perception and reaction: the agent “listens” to domain events and acts autonomously (publish decisions, trigger workflows).
 
Resilience4j + Metrics + Prometheus/Grafana
 Provide real-time feedback loops — the agent can observe health and performance, learn which actions succeed, and self-correct.
 
AWS Cloud Integrations (Lambda, Step Functions, S3, RDS)
 Extend the agent’s ability to act in the cloud environment — invoking serverless tasks, storing or retrieving context.
 
AI Reasoning Layer (LLM / Rules Engine)
 Added on top of these microservices, this layer interprets goals and context, then decides which services to call and in what sequence.
 


Formal architectural definition

 > 
 > 
 > Agentic AI (within a microservices system) is a composition of cognitive and operational services that:
 > 1️⃣ perceive data from distributed sources (APIs, events, metrics),
 > 2️⃣ reason over this context using LLMs, rules, or reinforcement logic,
 > 3️⃣ autonomously plan and invoke other microservices to fulfill objectives, and
 > 4️⃣ learn from system feedback to improve future decisions.
 > 
 > 
 > This turns your infrastructure from reactive services → proactive, goal-oriented agents.


Example: Risk Reporting Automation (from your earlier context)

Let’s map your Risk Reporting Spring Boot system to Agentic AI:


Step
Traditional
Agentic AI Version

Scheduler triggers daily risk report
 Fixed cron job
 AI agent monitors metrics and decides when to run reports (e.g., when volatility > X %).
 
Risk data fetched and emailed
 Static template
 Agent personalizes recipients, picks Slack vs. email based on engagement history.
 
Human checks anomalies
 Manual dashboard scan
 AI correlates anomalies, raises incident proactively in Slack with explanation and suggested fix.
 

Outcome: system demonstrates initiative and judgment, not mere automation.


How to Implement in Java 21 + AWS

Perception services:

Subscribe to Kafka topics or Prometheus alert streams.


Use WebClient to fetch real-time metrics or logs.



Reasoning layer:

Integrate an LLM API or on-prem model (e.g., via AWS Bedrock, LangChain4j).


Encode domain rules with Drools or custom DSLs.



Action layer:

Use Spring Cloud OpenFeign or REST clients to invoke other microservices.


Orchestrate actions with AWS Step Functions or event chains.



Learning/Feedback loop:

Store outcomes in RDS or DynamoDB.


Retrain scoring models or update rule weights periodically.




Observability & Control

Agentic AI must be observable and governable:

Every decision → logged as structured event (agent_decision_log).


Each microservice exposes actuator endpoints → AI monitors its own effect.


Dashboards (Grafana) show agent actions vs. outcomes.


This maintains trust and auditability — crucial for production systems.


Agentic AI, in a Java 21 Spring Boot microservices + AWS context, is an autonomous orchestration layer that perceives system and business signals, reasons about them, and proactively invokes distributed microservices to achieve predefined goals while continuously learning from feedback.


