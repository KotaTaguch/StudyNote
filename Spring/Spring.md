0からspringbootを使ったWebアプリケーションの開発手順を以下に示します。
# Spring Bootを使ったWebアプリケーションの開発手順
## 1. 開発環境の準備
Spring Bootのローカル開発環境構築手順については、[こちらのドキュメント](https://az.sd.cicdgw.nekonet.ne.jp/springer-projects/springer-docs/-/blob/master/03.%E3%82%A2%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E9%96%8B%E7%99%BA/01.%E3%83%AD%E3%83%BC%E3%82%AB%E3%83%AB%E9%96%8B%E7%99%BA%E7%92%B0%E5%A2%83%E6%A7%8B%E7%AF%89/README.md)を参照してください。
もしくは、プロジェクトごとの環境構築資料に従ってください。

## 2. プロジェクト構成の全体像
`us`（User Service）は外部向け API を公開し、入力バリデーションとレスポンス整形を担当します。`bs`（Backend Service）はデータベースから社員情報を取得し、OpenAPI（Swagger UI）で契約を公開します。`<product>` は実際のプロダクト識別子に置き換えてください。

### `us` のディレクトリ構成
```
src/main/java/jp/co/nekonet/<product>/us/api/employee
├── controller
│   ├── EmployeeController.java
│   └── form
│       └── EmployeeSearchForm.java
├── dto
│   └── EmployeeResponseDto.java
├── mapper
│   └── EmployeeMapper.java
├── model
│   ├── Employee.java
│   └── EmployeeSearchCondition.java
├── repository
│   ├── EmployeeRepository.java
│   └── impl
│       └── EmployeeRepositoryImpl.java
└── service
    ├── EmployeeService.java
    └── impl
        └── EmployeeServiceImpl.java
```

### `bs` のディレクトリ構成
```
src/main/java/jp/co/nekonet/<product>/bs/api/employee
├── controller
│   ├── EmployeeController.java
│   └── schema
│       └── EmployeeResponse.java
├── mapper
│   └── EmployeeMapper.java
├── repository
│   ├── EmployeeRepository.java
│   └── entity
│       └── EmployeeEntity.java
└── service
    ├── EmployeeQueryService.java
    └── impl
        └── EmployeeQueryServiceImpl.java
```

## 3. `us` 層の作成
### 3.1 コントローラー
#### EmployeeSearchForm.java
```java
package jp.co.nekonet.<product>.us.api.employee.controller.form;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Pattern;

public class EmployeeSearchForm {
    @NotBlank(message = "社員番号は必須です。")
    @Pattern(regexp = "[A-Z][0-9]{4}", message = "社員番号は先頭1文字が英大文字、続く4桁が数字の形式で指定してください。")
    private String employeeNumber;

    public String getEmployeeNumber() {
        return employeeNumber;
    }

    public void setEmployeeNumber(String employeeNumber) {
        this.employeeNumber = employeeNumber;
    }
}
```

#### EmployeeController.java
```java
package jp.co.nekonet.<product>.us.api.employee.controller;

import jp.co.nekonet.<product>.us.api.employee.controller.form.EmployeeSearchForm;
import jp.co.nekonet.<product>.us.api.employee.dto.EmployeeResponseDto;
import jp.co.nekonet.<product>.us.api.employee.model.EmployeeSearchCondition;
import jp.co.nekonet.<product>.us.api.employee.service.EmployeeService;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/employees")
@Validated
public class EmployeeController {

    private final EmployeeService employeeService;

    public EmployeeController(EmployeeService employeeService) {
        this.employeeService = employeeService;
    }

    @GetMapping
    public ResponseEntity<EmployeeResponseDto> getEmployee(@Valid EmployeeSearchForm form) {
        EmployeeSearchCondition condition = new EmployeeSearchCondition(form.getEmployeeNumber());
        EmployeeResponseDto response = employeeService.findByEmployeeNumber(condition);
        return ResponseEntity.ok(response);
    }
}
```

### 3.2 サービス
#### EmployeeService.java
```java
package jp.co.nekonet.<product>.us.api.employee.service;

import jp.co.nekonet.<product>.us.api.employee.dto.EmployeeResponseDto;
import jp.co.nekonet.<product>.us.api.employee.model.EmployeeSearchCondition;

public interface EmployeeService {
    EmployeeResponseDto findByEmployeeNumber(EmployeeSearchCondition condition);
}
```

#### EmployeeServiceImpl.java
```java
package jp.co.nekonet.<product>.us.api.employee.service.impl;

import jp.co.nekonet.<product>.us.api.employee.dto.EmployeeResponseDto;
import jp.co.nekonet.<product>.us.api.employee.mapper.EmployeeMapper;
import jp.co.nekonet.<product>.us.api.employee.model.EmployeeSearchCondition;
import jp.co.nekonet.<product>.us.api.employee.repository.EmployeeRepository;
import jp.co.nekonet.<product>.us.api.employee.service.EmployeeService;
import org.springframework.stereotype.Service;

@Service
public class EmployeeServiceImpl implements EmployeeService {

    private final EmployeeRepository repository;
    private final EmployeeMapper mapper;

    public EmployeeServiceImpl(EmployeeRepository repository, EmployeeMapper mapper) {
        this.repository = repository;
        this.mapper = mapper;
    }

    @Override
    public EmployeeResponseDto findByEmployeeNumber(EmployeeSearchCondition condition) {
        return repository.findByEmployeeNumber(condition)
                .map(mapper::toDto)
                .orElseThrow(() -> new IllegalArgumentException(
                        "指定された社員番号の情報が見つかりませんでした: " + condition.getEmployeeNumber()
                ));
    }
}
```

#### EmployeeMapper.java
```java
package jp.co.nekonet.<product>.us.api.employee.mapper;

import jp.co.nekonet.<product>.us.api.employee.dto.EmployeeResponseDto;
import jp.co.nekonet.<product>.us.api.employee.model.Employee;
import org.springframework.stereotype.Component;

@Component
public class EmployeeMapper {

    public EmployeeResponseDto toDto(Employee employee) {
        if (employee == null) {
            return null;
        }
        return new EmployeeResponseDto(
                employee.getEmployeeNumber(),
                employee.getName(),
                employee.getEmail()
        );
    }
}
```

### 3.3 リポジトリ
`us` のリポジトリ実装では、`WebClient` などの HTTP クライアントを通じて `bs` の社員参照 API を呼び出し、取得したレスポンスをドメインモデルに変換します。

#### EmployeeRepository.java
```java
package jp.co.nekonet.<product>.us.api.employee.repository;

import jp.co.nekonet.<product>.us.api.employee.model.Employee;
import jp.co.nekonet.<product>.us.api.employee.model.EmployeeSearchCondition;

import java.util.Optional;

public interface EmployeeRepository {
    Optional<Employee> findByEmployeeNumber(EmployeeSearchCondition condition);
}
```

#### EmployeeRepositoryImpl.java
```java
package jp.co.nekonet.<product>.us.api.employee.repository.impl;

import jp.co.nekonet.<product>.us.api.employee.model.Employee;
import jp.co.nekonet.<product>.us.api.employee.model.EmployeeSearchCondition;
import jp.co.nekonet.<product>.us.api.employee.repository.EmployeeRepository;
import jakarta.annotation.PostConstruct;
import org.springframework.stereotype.Repository;

import java.util.HashMap;
import java.util.Map;
import java.util.Optional;

@Repository
public class EmployeeRepositoryImpl implements EmployeeRepository {

    private final Map<String, Employee> dataStore = new HashMap<>();

    @PostConstruct
    void loadFixtures() {
        dataStore.put("A1001", new Employee("A1001", "田中 太郎", "taro.tanaka@example.com"));
        dataStore.put("A1002", new Employee("A1002", "佐藤 花子", "hanako.sato@example.com"));
        dataStore.put("A2001", new Employee("A2001", "鈴木 次郎", "jiro.suzuki@example.com"));
    }

    @Override
    public Optional<Employee> findByEmployeeNumber(EmployeeSearchCondition condition) {
        if (condition == null || condition.getEmployeeNumber() == null) {
            return Optional.empty();
        }
        return Optional.ofNullable(dataStore.get(condition.getEmployeeNumber()));
    }
}
```

### 3.4 DTO とモデル
#### EmployeeResponseDto.java
```java
package jp.co.nekonet.<product>.us.api.employee.dto;

public class EmployeeResponseDto {
    private String employeeNumber;
    private String name;
    private String email;

    public EmployeeResponseDto() {
    }

    public EmployeeResponseDto(String employeeNumber, String name, String email) {
        this.employeeNumber = employeeNumber;
        this.name = name;
        this.email = email;
    }

    public String getEmployeeNumber() {
        return employeeNumber;
    }

    public void setEmployeeNumber(String employeeNumber) {
        this.employeeNumber = employeeNumber;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }
}
```

#### Employee.java
```java
        import org.springframework.beans.factory.annotation.Value;
package jp.co.nekonet.<product>.us.api.employee.model;
        import org.springframework.web.reactive.function.client.WebClient;
        import org.springframework.web.reactive.function.client.WebClientResponseException;

    private final String email;

    public Employee(String employeeNumber, String name, String email) {
        this.employeeNumber = employeeNumber;
        this.name = name;
            private final WebClient webClient;
        return name;
            public EmployeeRepositoryImpl(WebClient.Builder builder,
                                          @Value("${bs.employee-service.base-url}") String baseUrl) {
                this.webClient = builder.baseUrl(baseUrl).build();
            }

            @Override
            public Optional<Employee> findByEmployeeNumber(EmployeeSearchCondition condition) {
                if (condition == null || condition.getEmployeeNumber() == null) {
                    return Optional.empty();
                }

                try {
                    EmployeeResponseDto response = webClient
                            .get()
                            .uri(uriBuilder -> uriBuilder
                                    .path("/api/employees")
                                    .queryParam("employeeNumber", condition.getEmployeeNumber())
                                    .build())
                            .retrieve()
                            .bodyToMono(EmployeeResponseDto.class)
                            .block();

                    if (response == null) {
                        return Optional.empty();
                    }

                    return Optional.of(new Employee(
                            response.getEmployeeNumber(),
                            response.getName(),
                            response.getEmail()
                    ));
                } catch (WebClientResponseException.NotFound ex) {
                    return Optional.empty();
                }
```

#### EmployeeSearchCondition.java
```java
package jp.co.nekonet.<product>.us.api.employee.model;

public class EmployeeSearchCondition {
    private final String employeeNumber;

    public EmployeeSearchCondition(String employeeNumber) {
        this.employeeNumber = employeeNumber;
    }

    public String getEmployeeNumber() {
        return employeeNumber;
    }
}
```

## 4. `bs` 層の作成
`bs` は OpenAPI を公開しつつ DB にアクセスするバックエンドサービスです。Springdoc を通じて Swagger UI を提供し、API 契約を可視化します。

### 4.1 コントローラー
```java
package jp.co.nekonet.<product>.bs.api.employee.controller;

import jp.co.nekonet.<product>.bs.api.employee.controller.schema.EmployeeResponse;
import jp.co.nekonet.<product>.bs.api.employee.service.EmployeeQueryService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/employees")
@Validated
@Tag(name = "Employee", description = "社員参照 API")
public class EmployeeController {

    private final EmployeeQueryService employeeQueryService;

    public EmployeeController(EmployeeQueryService employeeQueryService) {
        this.employeeQueryService = employeeQueryService;
    }

    @GetMapping
    @Operation(summary = "社員情報の取得", description = "社員番号をキーに社員情報を返却します。")
    public ResponseEntity<EmployeeResponse> findByEmployeeNumber(@RequestParam("employeeNumber") String employeeNumber) {
        EmployeeResponse response = employeeQueryService.findByEmployeeNumber(employeeNumber);
        return ResponseEntity.ok(response);
    }
}
```

#### EmployeeResponse.java
```java
package jp.co.nekonet.<product>.bs.api.employee.controller.schema;

import io.swagger.v3.oas.annotations.media.Schema;

@Schema(description = "社員検索レスポンス")
public class EmployeeResponse {

    @Schema(description = "社員番号", example = "A1001")
    private String employeeNumber;

    @Schema(description = "氏名", example = "田中 太郎")
    private String name;

    @Schema(description = "メールアドレス", example = "taro.tanaka@example.com")
    private String email;

    public EmployeeResponse() {
    }

    public EmployeeResponse(String employeeNumber, String name, String email) {
        this.employeeNumber = employeeNumber;
        this.name = name;
        this.email = email;
    }

    public String getEmployeeNumber() {
        return employeeNumber;
    }

    public void setEmployeeNumber(String employeeNumber) {
        this.employeeNumber = employeeNumber;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }
}
```

### 4.2 サービス
```java
package jp.co.nekonet.<product>.bs.api.employee.service;

import jp.co.nekonet.<product>.bs.api.employee.controller.schema.EmployeeResponse;

public interface EmployeeQueryService {
    EmployeeResponse findByEmployeeNumber(String employeeNumber);
}
```

#### EmployeeQueryServiceImpl.java
```java
package jp.co.nekonet.<product>.bs.api.employee.service.impl;

import jp.co.nekonet.<product>.bs.api.employee.controller.schema.EmployeeResponse;
import jp.co.nekonet.<product>.bs.api.employee.mapper.EmployeeMapper;
import jp.co.nekonet.<product>.bs.api.employee.repository.EmployeeRepository;
import jp.co.nekonet.<product>.bs.api.employee.service.EmployeeQueryService;
import org.springframework.stereotype.Service;

@Service
public class EmployeeQueryServiceImpl implements EmployeeQueryService {

    private final EmployeeRepository repository;
    private final EmployeeMapper mapper;

    public EmployeeQueryServiceImpl(EmployeeRepository repository, EmployeeMapper mapper) {
        this.repository = repository;
        this.mapper = mapper;
    }

    @Override
    public EmployeeResponse findByEmployeeNumber(String employeeNumber) {
        return repository.findByEmployeeNumber(employeeNumber)
                .map(mapper::toResponse)
                .orElseThrow(() -> new IllegalArgumentException(
                        "指定された社員番号の情報が見つかりませんでした: " + employeeNumber
                ));
    }
}
```

#### EmployeeMapper.java
```java
package jp.co.nekonet.<product>.bs.api.employee.mapper;

import jp.co.nekonet.<product>.bs.api.employee.controller.schema.EmployeeResponse;
import jp.co.nekonet.<product>.bs.api.employee.repository.entity.EmployeeEntity;
import org.springframework.stereotype.Component;

@Component
public class EmployeeMapper {

    public EmployeeResponse toResponse(EmployeeEntity entity) {
        if (entity == null) {
            return null;
        }
        return new EmployeeResponse(
                entity.getEmployeeNumber(),
                entity.getName(),
                entity.getEmail()
        );
    }
}
```

### 4.3 リポジトリ
```java
package jp.co.nekonet.<product>.bs.api.employee.repository;

import jp.co.nekonet.<product>.bs.api.employee.repository.entity.EmployeeEntity;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

public interface EmployeeRepository extends JpaRepository<EmployeeEntity, String> {
    Optional<EmployeeEntity> findByEmployeeNumber(String employeeNumber);
}
```

#### EmployeeEntity.java
```java
package jp.co.nekonet.<product>.bs.api.employee.repository.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;

@Entity
@Table(name = "employees")
public class EmployeeEntity {

    @Id
    @Column(name = "employee_number", length = 8, nullable = false)
    private String employeeNumber;

    @Column(name = "name", nullable = false)
    private String name;

    @Column(name = "email", nullable = false)
    private String email;

    protected EmployeeEntity() {
    }

    public EmployeeEntity(String employeeNumber, String name, String email) {
        this.employeeNumber = employeeNumber;
        this.name = name;
        this.email = email;
    }

    public String getEmployeeNumber() {
        return employeeNumber;
    }

    public String getName() {
        return name;
    }

    public String getEmail() {
        return email;
    }
}
```

### 4.4 OpenAPI と DB 設定
- `pom.xml` または `build.gradle` に `org.springdoc:springdoc-openapi-starter-webmvc-ui` を追加し、Swagger UI を有効化します。
- `application.yml` 等でデータベース接続設定（JDBC URL、ユーザ、パスワード）と JPA の方言・DDL モードを定義します。
- 起動後、`http://<bs-host>:<port>/swagger-ui.html`（もしくは `/swagger-ui/index.html`）で API ドキュメントを確認します。

## 5. `us` から `bs` への連携設定
- `application.yml`（または `application.properties`）に `bs.employee-service.base-url` を定義し、`us` が接続すべき `bs` のホスト・ポートを指定します。
- `WebClient.Builder` は Spring Boot が自動登録するため、そのまま DI 可能です。必要に応じて `WebClientCustomizer` などでタイムアウトやヘッダーを共通設定します。
- リポジトリで 404 を `Optional.empty()` に変換しているため、`EmployeeServiceImpl` は従来通り存在チェックの責務を果たせます。

この構成により、社員番号をキーに社員名とメールアドレスを返却する参照 API を `us` と `bs` の分離した責務で構築できます。
