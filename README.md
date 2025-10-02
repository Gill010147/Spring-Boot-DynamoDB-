# Spring Boot & DynamoDB 연동 가이드

이 문서는 Spring Boot 애플리케이션과 AWS DynamoDB를 연동하는 과정과 주요 설정, 트러블슈팅 팁을 제공합니다.

## 1. 개요

이 프로젝트는 Spring Boot를 사용하여 간단한 리더보드 API를 제공하며, 데이터 저장소로 AWS DynamoDB를 사용합니다. 사용자가 로그인하고, 점수를 업데이트하며, 전체 순위를 확인할 수 있는 기능을 포함합니다.

- **DynamoDB**: AWS에서 제공하는 NoSQL 데이터베이스 서비스로, 빠르고 유연하며 확장성이 뛰어납니다.
- **Spring Boot**: 빠르고 간편하게 독립 실행형 애플리케이션을 만들 수 있는 Java 기반 프레임워크입니다.
- **AWS SDK for Java 2.x**: Java 애플리케이션에서 AWS 서비스를 쉽게 사용하도록 돕는 라이브러리입니다.

## 2. 연동 방법

### 단계 1: `build.gradle`에 의존성 추가

DynamoDB 연동을 위해 AWS SDK 관련 의존성을 추가해야 합니다. `dynamodb-enhanced` 클라이언트를 사용하면 Java 객체를 DynamoDB 테이블에 쉽게 매핑할 수 있습니다.

```groovy
// build.gradle

dependencies {
    // AWS SDK Bill of Materials (BOM) - SDK 버전 관리를 용이하게 함
	implementation(platform("software.amazon.awssdk:bom:2.34.5"))

    // Spring Boot 관련 기본 의존성
	implementation 'org.springframework.boot:spring-boot-starter-web'

    // AWS DynamoDB Core 및 Enhanced Client
	implementation("software.amazon.awssdk:dynamodb")
	implementation("software.amazon.awssdk:dynamodb-enhanced")

    // ... 기타 의존성
}
```

### 단계 2: AWS 자격 증명(Credentials) 설정

AWS 서비스에 접근하려면 자격 증명이 필요합니다. 이 프로젝트에서는 코드에 직접 키를 하드코딩하는 대신, AWS의 **기본 자격 증명 공급자 체인(Default Credential Provider Chain)** 을 사용합니다.

**권장하는 방법:**
1.  **환경 변수 사용 (로컬 개발 시)**:
    - `AWS_ACCESS_KEY_ID`: 발급받은 Access Key
    - `AWS_SECRET_ACCESS_KEY`: 발급받은 Secret Key
    - `AWS_REGION`: 사용하려는 AWS 리전 (예: `ap-northeast-2`)

2.  **IAM 역할 사용 (EC2/ECS 등 AWS 환경 배포 시)**:
    - EC2 인스턴스나 ECS 태스크에 DynamoDB 접근 권한이 있는 IAM 역할을 부여합니다. 이 방법이 가장 안전하고 권장됩니다.

> **경고**: 절대로 소스 코드나 `application.yml` 파일에 Access Key와 Secret Key를 직접 작성하지 마세요. Git에 커밋될 경우 보안 사고로 이어질 수 있습니다.

### 단계 3: `DynamoDbClient` Bean 설정 (`AwsConfig.java`)

Spring 애플리케이션 어디서든 `DynamoDbClient`를 주입받아 사용할 수 있도록 `@Configuration`을 사용하여 Bean으로 등록합니다.

```java
// src/main/java/com/example/demo/config/AwsConfig.java

package com.example.demo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.dynamodb.DynamoDbClient;

@Configuration
public class AwsConfig {

    @Bean
    public DynamoDbClient dynamoDbClient() {
        return DynamoDbClient.builder()
                .region(Region.AP_NORTHEAST_2) // 서울 리전으로 설정
                .build();
    }
}
```
- **`.region()`**: 사용할 AWS 리전을 명시적으로 지정합니다. 환경 변수(`AWS_REGION`)를 설정했다면 자동으로 인식되므로 이 코드를 생략할 수도 있습니다.
- **`.credentialsProvider()`**: 별도로 설정하지 않으면 위에서 설명한 기본 공급자 체인을 통해 자격 증명을 찾습니다.

### 단계 4: 데이터 모델(Entity) 생성 (`Person.java`)

DynamoDB 테이블과 매핑될 Java 객체를 생성합니다. `@DynamoDbBean`과 `@DynamoDbPartitionKey` 어노테이션을 사용합니다.

```java
// src/main/java/com/example/demo/model/Person.java

package com.example.demo.model;

import software.amazon.awssdk.enhanced.dynamodb.mapper.annotations.DynamoDbBean;
import software.amazon.awssdk.enhanced.dynamodb.mapper.annotations.DynamoDbPartitionKey;

@DynamoDbBean // 이 클래스가 DynamoDB 테이블과 매핑됨을 나타냄
public class Person {

    private String name;
    // ... other fields

    @DynamoDbPartitionKey // 'name' 속성을 파티션 키(기본 키)로 지정
    public String getName() {
        return name;
    }

    // ... getters and setters
}
```

### 단계 5: 서비스 로직 구현 (`DynamoDbService.java`)

실제로 DynamoDB와 상호작용하는 비즈니스 로직을 구현합니다.

```java
// src/main/java/com/example/demo/service/DynamoDbService.java

package com.example.demo.service;

import com.example.demo.model.Person;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.enhanced.dynamodb.DynamoDbEnhancedClient;
import software.amazon.awssdk.enhanced.dynamodb.DynamoDbTable;
import software.amazon.awssdk.enhanced.dynamodb.Key;
import software.amazon.awssdk.enhanced.dynamodb.TableSchema;
import software.amazon.awssdk.services.dynamodb.DynamoDbClient;

@Service
public class DynamoDbService {

    private final DynamoDbTable<Person> personTable;
    private static final String TABLE_NAME = "ce-28-people"; // DynamoDB 테이블 이름

    public DynamoDbService(DynamoDbClient dynamoDbClient) {
        // 주입받은 기본 클라이언트를 사용하여 Enhanced Client 생성
        DynamoDbEnhancedClient enhancedClient = DynamoDbEnhancedClient.builder()
                .dynamoDbClient(dynamoDbClient)
                .build();

        // 테이블 이름과 Java 클래스 스키마를 연결하여 테이블 객체 생성
        this.personTable = enhancedClient.table(TABLE_NAME, TableSchema.fromBean(Person.class));
    }

    // 특정 사용자 조회
    public Person getPerson(String name) {
        return personTable.getItem(Key.builder().partitionValue(name).build());
    }

    // 점수 업데이트
    public void updateScore(String name, int score) {
        Person person = getPerson(name);
        if (person != null) {
            // ... (logic)
            personTable.updateItem(person); // 아이템 업데이트
        }
    }

    // ... 기타 CRUD 및 비즈니스 로직
}
```
- **`DynamoDbEnhancedClient`**: `DynamoDbClient`를 감싸서 `Person` 객체를 직접 테이블에 저장하거나 조회하는 등 편리한 기능을 제공합니다.
- **`personTable.getItem(...)`**: 파티션 키를 사용하여 단일 아이템을 가져옵니다.
- **`personTable.updateItem(...)`**: 객체 내용을 기반으로 아이템을 업데이트합니다.
- **`personTable.scan()`**: 테이블의 모든 아이템을 스캔합니다 (주의: 프로덕션 환경의 대용량 테이블에서는 비용과 성능 문제가 발생할 수 있음).

## 3. 주의사항 및 트러블슈팅

- **테이블 이름이 하드코딩되어 있습니다.**
  - 현재 `DynamoDbService.java`에 테이블 이름(`ce-28-people`)이 직접 작성되어 있습니다.
  - **개선 방안**: `application.yml`에 테이블 이름을 변수로 빼고 `@Value` 어노테이션을 사용하여 주입받으면 유연성이 높아집니다.
    ```yaml
    # application.yml
    aws:
      dynamodb:
        table-name: "ce-28-people"
    ```
    ```java
    // DynamoDbService.java
    @Value("${aws.dynamodb.table-name}")
    private String tableName;
    ```

- **IAM 권한 문제**: `AccessDeniedException`이 발생할 경우
  - 애플리케이션이 사용하는 AWS 자격 증명(IAM 사용자 또는 역할)에 필요한 DynamoDB 권한이 있는지 확인하세요.
  - 최소 필요 권한: `dynamodb:GetItem`, `dynamodb:PutItem`, `dynamodb:UpdateItem`, `dynamodb:DeleteItem`, `dynamodb:Scan` 등

- **자격 증명 없음 오류**: `Unable to load credentials`
  - 위에서 설명한 **기본 자격 증명 공급자 체인**에 따라 자격 증명이 올바르게 설정되었는지 확인하세요. 로컬 환경이라면 환경 변수가 정확히 설정되었는지 다시 확인하는 것이 좋습니다.

- **리소스를 찾을 수 없음**: `ResourceNotFoundException`
  - `DynamoDbService`에 지정된 테이블 이름이 실제 AWS DynamoDB에 존재하는지, 그리고 리전이 올바른지 확인하세요.

- **로컬 테스트**:
  - **DynamoDB Local**을 사용하면 AWS에 직접 연결하지 않고도 로컬 환경에서 DynamoDB를 테스트할 수 있습니다.
  - 이 경우, `DynamoDbClient` Bean을 생성할 때 `.endpointOverride()`를 사용하여 로컬 엔드포인트(예: `http://localhost:8000`)를 지정해야 합니다.
