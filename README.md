# sample
# GUÍA PC2 — Sensibo Climate Control (DAOP 1ASI0729)
**Objetivo: Sobresaliente en todos los criterios**

> Reemplaza `<NRC>` por tu NRC y `<CODE>` por tu código en minúsculas.
> Ejemplo: NRC=7377, CODE=u20241a195 → proyecto `pc27377u20241a195`
> Package raíz: `com.sensibo.platform.u20241a195`

---

## QUÉ TIENE EL ZIP DE REFERENCIA Y QUÉ LE FALTA

El zip tiene implementado (y está bien):
- Value objects: `DeviceTypes`, `MacAddress`, `SensiboUserId`, `SerialNumber`, `Version`
- Aggregate: `Device`
- Commands y Queries
- Application services (interfaces + impl)
- Infrastructure: converters, assembler persistence, adapter

**Lo que le falta (y es obligatorio para Sobresaliente):**

| # | Qué falta | Por qué importa |
|---|-----------|-----------------|
| 1 | Package `shared` completo | El zip lo referencia con imports pero los archivos no están incluidos |
| 2 | `DevicePersistenceEntity` | El archivo existe en el zip pero está **vacío** (sin `@Entity`, sin columnas) |
| 3 | `DevicePersistenceRepository` | El Spring Data JPA repo no existe en el zip |
| 4 | `interfaces/rest/` completo | No hay Controller, Resources ni Transform Assemblers |
| 5 | `application.properties` | No hay carpeta resources en el zip |
| 6 | `messages.properties` + `messages_es.properties` | Sin esto el i18n no funciona (requisito técnico #7) |
| 7 | `README.md` correcto | El del zip dice "ACME Learning Center" — debe decir Sensibo con tu nombre |
| 8 | JavaDoc + `@author` | Requisito técnico #11 |
| 9 | Swagger `@Tag`, `@Operation`, `@ApiResponse` | Requisito técnico #12 |

**Error en el zip que debes corregir:**
El `CreateDeviceCommand` del zip incluye `serialNumber` como parámetro. El enunciado dice explícitamente: *"Los valores de id y serialNumber no se ingresan al momento de agregar, pues son autogenerados"*. El `serialNumber` no va en el Command ni en el request body.

---

## PASO 1 — Crear el proyecto en Spring Initializr

Usa este enlace (ajusta `Artifact` y `Package name` con tus datos antes de generar):

```
https://start.spring.io/#!type=maven-project&language=java&platformVersion=3.5.0&packaging=jar&jvmVersion=25&groupId=com.sensibo.platform&artifactId=pc2<NRC><CODE>&packageName=com.sensibo.platform.<CODE>&dependencies=data-jpa,validation,web,devtools,postgresql,lombok
```

> El enunciado pide **Java 25** y **Spring Boot 3.5.0**. Úsalos aunque el zip de referencia use versiones distintas.

Guarda en `IdeaProjects\1ASI0729\` y ábrelo en IntelliJ IDEA.

---

## PASO 2 — pom.xml

Confirma en `<properties>`:
```xml
<properties>
    <java.version>25</java.version>
</properties>
```

Agrega dentro de `<dependencies>`:
```xml
<!-- Swagger UI -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.6.0</version>
</dependency>

<!-- Necesario para la estrategia de nombres de tabla en plural -->
<dependency>
    <groupId>io.github.encryptorcode</groupId>
    <artifactId>pluralize</artifactId>
    <version>1.0.0</version>
</dependency>
```

---

## PASO 3 — Base de datos

En pgAdmin:
```sql
CREATE DATABASE sensibo;
```
Luego conéctate a la base `sensibo` y ejecuta:
```sql
CREATE SCHEMA IF NOT EXISTS sensibo;
```
Hibernate crea las tablas automáticamente, pero el esquema debes crearlo tú una vez.

---

## PASO 4 — application.properties

Archivo: `src/main/resources/application.properties`

```properties
spring.application.name=pc2<NRC><CODE>

spring.datasource.url=jdbc:postgresql://localhost:5432/sensibo
spring.datasource.username=postgres
spring.datasource.password=12345678
spring.datasource.driver-class-name=org.postgresql.Driver

spring.jpa.database=postgresql
spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=update
spring.jpa.open-in-view=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.properties.hibernate.default_schema=sensibo
spring.jpa.hibernate.naming.physical-strategy=com.sensibo.platform.<CODE>.shared.infrastructure.persistence.jpa.configuration.strategy.SnakeCaseWithPluralizedTablePhysicalNamingStrategy

server.port=8096

documentation.application.description=@project.description@
documentation.application.version=@project.version@

spring.messages.basename=messages
spring.messages.encoding=UTF-8
```

---

## PASO 5 — Estructura de paquetes completa

Crea estos paquetes dentro de `src/main/java/com/sensibo/platform/<CODE>/`:

```
com.sensibo.platform.<CODE>
├── registry
│   ├── application
│   │   ├── commandservices          ← interfaces públicas
│   │   ├── internal
│   │   │   ├── commandservices      ← implementaciones
│   │   │   └── queryservices
│   │   └── queryservices            ← interfaces públicas
│   ├── domain
│   │   ├── model
│   │   │   ├── aggregates
│   │   │   ├── commands
│   │   │   ├── queries
│   │   │   └── valueobjects
│   │   └── repositories
│   ├── infrastructure
│   │   └── persistence
│   │       └── jpa
│   │           ├── adapters
│   │           ├── assemblers
│   │           ├── converters
│   │           ├── entities
│   │           └── repositories     ← Spring Data JPA repo (falta en el zip)
│   └── interfaces
│       └── rest
│           ├── resources             ← DTOs request/response (falta en el zip)
│           └── transform             ← Assemblers REST (falta en el zip)
└── shared
    ├── application
    │   └── result                    ← Result<T,E> + ApplicationError
    ├── domain
    │   └── model
    │       └── aggregates            ← AbstractDomainAggregateRoot
    └── infrastructure
        ├── documentation
        │   └── openapi
        │       └── configuration     ← OpenApiConfiguration
        ├── i18n
        │   └── configuration         ← WebConfig (locale resolver)
        └── persistence
            └── jpa
                └── configuration
                    └── strategy      ← SnakeCaseWithPluralizedTablePhysicalNamingStrategy
```

---

## PASO 6 — Clase principal

```java
package com.sensibo.platform.<CODE>;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;

/**
 * Main entry point for Sensibo Climate Control Platform.
 *
 * @author Tu Nombre y Apellido
 */
@SpringBootApplication
@EnableJpaAuditing
public class Pc2<NRC><CODE>Application {
    public static void main(String[] args) {
        SpringApplication.run(Pc2<NRC><CODE>Application.class, args);
    }
}
```

---

## PASO 7 — Bounded Context SHARED

### 7.1 AbstractDomainAggregateRoot

```java
package com.sensibo.platform.<CODE>.shared.domain.model.aggregates;

import jakarta.persistence.*;
import lombok.Getter;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;
import java.util.Date;

/**
 * Base class for all domain aggregate roots.
 * Provides JPA auditing fields createdAt and updatedAt.
 *
 * @author Tu Nombre y Apellido
 */
@Getter
@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass
public abstract class AbstractDomainAggregateRoot {

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Date createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private Date updatedAt;
}
```

### 7.2 Result

```java
package com.sensibo.platform.<CODE>.shared.application.result;

/**
 * Generic result wrapper for CQRS operations.
 *
 * @param <T> success value type
 * @param <E> error value type
 * @author Tu Nombre y Apellido
 */
public class Result<T, E> {
    private final T value;
    private final E error;
    private final boolean isSuccess;

    private Result(T value, E error, boolean isSuccess) {
        this.value = value;
        this.error = error;
        this.isSuccess = isSuccess;
    }

    public static <T, E> Result<T, E> success(T value) {
        return new Result<>(value, null, true);
    }

    public static <T, E> Result<T, E> failure(E error) {
        return new Result<>(null, error, false);
    }

    public boolean isSuccess() { return isSuccess; }
    public boolean isFailure() { return !isSuccess; }
    public T getValue() { return value; }
    public E getError() { return error; }
}
```

### 7.3 ApplicationError

```java
package com.sensibo.platform.<CODE>.shared.application.result;

/**
 * Represents an application-level error.
 *
 * @author Tu Nombre y Apellido
 */
public record ApplicationError(ErrorType type, String message) {

    public enum ErrorType { NOT_FOUND, CONFLICT, VALIDATION, UNEXPECTED }

    public static ApplicationError conflict(String entity, String message) {
        return new ApplicationError(ErrorType.CONFLICT, entity + ": " + message);
    }

    public static ApplicationError notFound(String entity, String message) {
        return new ApplicationError(ErrorType.NOT_FOUND, entity + ": " + message);
    }

    public static ApplicationError unexpected(String context, String message) {
        return new ApplicationError(ErrorType.UNEXPECTED, context + ": " + message);
    }
}
```

### 7.4 SnakeCaseWithPluralizedTablePhysicalNamingStrategy

```java
package com.sensibo.platform.<CODE>.shared.infrastructure.persistence.jpa.configuration.strategy;

import org.hibernate.boot.model.naming.Identifier;
import org.hibernate.boot.model.naming.PhysicalNamingStrategy;
import org.hibernate.engine.jdbc.env.spi.JdbcEnvironment;
import static io.github.encryptorcode.pluralize.Pluralize.pluralize;

/**
 * Custom naming strategy: converts class names to snake_case and pluralizes table names.
 *
 * @author Tu Nombre y Apellido
 */
public class SnakeCaseWithPluralizedTablePhysicalNamingStrategy implements PhysicalNamingStrategy {

    private String toSnakeCase(String name) {
        if (name == null) return null;
        return name.replaceAll("([a-z])([A-Z])", "$1_$2").toLowerCase();
    }

    @Override
    public Identifier toPhysicalCatalogName(Identifier logicalName, JdbcEnvironment env) {
        return logicalName;
    }

    @Override
    public Identifier toPhysicalSchemaName(Identifier logicalName, JdbcEnvironment env) {
        return logicalName;
    }

    @Override
    public Identifier toPhysicalTableName(Identifier logicalName, JdbcEnvironment env) {
        if (logicalName == null) return null;
        return Identifier.toIdentifier(pluralize(toSnakeCase(logicalName.getText())));
    }

    @Override
    public Identifier toPhysicalSequenceName(Identifier logicalName, JdbcEnvironment env) {
        return logicalName;
    }

    @Override
    public Identifier toPhysicalColumnName(Identifier logicalName, JdbcEnvironment env) {
        if (logicalName == null) return null;
        return Identifier.toIdentifier(toSnakeCase(logicalName.getText()));
    }
}
```

### 7.5 OpenApiConfiguration

```java
package com.sensibo.platform.<CODE>.shared.infrastructure.documentation.openapi.configuration;

import io.swagger.v3.oas.models.*;
import io.swagger.v3.oas.models.info.*;
import io.swagger.v3.oas.models.servers.Server;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.List;

/**
 * OpenAPI / Swagger UI configuration.
 *
 * @author Tu Nombre y Apellido
 */
@Configuration
public class OpenApiConfiguration {

    @Value("${spring.application.name}")
    String applicationName;

    @Value("${documentation.application.description}")
    String applicationDescription;

    @Value("${documentation.application.version}")
    String applicationVersion;

    /**
     * Builds the OpenAPI descriptor for Swagger UI.
     *
     * @return configured OpenAPI instance
     */
    @Bean
    public OpenAPI sensiboOpenApi() {
        return new OpenAPI()
                .info(new Info()
                        .title(this.applicationName)
                        .description(this.applicationDescription)
                        .version(this.applicationVersion)
                        .contact(new Contact()
                                .name("Sensibo Climate Control")
                                .email("support@sensibo.com")
                                .url("https://sensibo.com/support"))
                        .license(new License()
                                .name("Apache 2.0")
                                .url("https://www.apache.org/licenses/LICENSE-2.0.html")))
                .externalDocs(new ExternalDocumentation()
                        .description("Sensibo Platform Documentation")
                        .url("https://sensibo.wiki.github.io/docs"))
                .servers(List.of(
                        new Server().url("http://localhost:8096").description("Local Development"),
                        new Server().url("https://api.sensibo.com").description("Production")
                ));
    }
}
```

### 7.6 WebConfig — i18n por Accept-Language

```java
package com.sensibo.platform.<CODE>.shared.infrastructure.i18n.configuration;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.LocaleResolver;
import org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver;
import java.util.List;
import java.util.Locale;

/**
 * Web MVC configuration for i18n locale resolution via Accept-Language header.
 *
 * @author Tu Nombre y Apellido
 */
@Configuration
public class WebConfig {

    /**
     * Resolves locale from the Accept-Language request header.
     * Supports EN, EN-US, ES, ES-PE as required by the technical constraints.
     *
     * @return configured locale resolver
     */
    @Bean
    public LocaleResolver localeResolver() {
        var resolver = new AcceptHeaderLocaleResolver();
        resolver.setDefaultLocale(Locale.ENGLISH);
        resolver.setSupportedLocales(List.of(
                Locale.ENGLISH,
                Locale.forLanguageTag("en-US"),
                new Locale("es"),
                Locale.forLanguageTag("es-PE")
        ));
        return resolver;
    }
}
```

---

## PASO 8 — Archivos i18n

### `src/main/resources/messages.properties` (English — default)

```properties
device.duplicate=A device with the same serial number and MAC address already exists.
device.invalid.mac=Invalid MAC address format. Expected format: AA:BB:CC:DD:EE:FF
device.invalid.version=Invalid firmware version format. Expected format: X.Y.Z
device.invalid.device.type=Invalid device type. Valid values: SMART_AC_CONTROLLER, ROOM_SENSOR, AIR_QUALITY_MONITOR, DOOR_WINDOW_SENSOR
device.invalid.installation.date=Installation date cannot be in the future.
device.invalid.user.id=User ID must be a positive number greater than zero.
device.not.found=Device not found.
validation.error=Validation failed.
error.unexpected=An unexpected error occurred.
```

### `src/main/resources/messages_es.properties` (Español)

```properties
device.duplicate=Ya existe un dispositivo con el mismo número de serie y dirección MAC.
device.invalid.mac=Formato de dirección MAC inválido. Formato esperado: AA:BB:CC:DD:EE:FF
device.invalid.version=Formato de versión de firmware inválido. Formato esperado: X.Y.Z
device.invalid.device.type=Tipo de dispositivo inválido. Valores válidos: SMART_AC_CONTROLLER, ROOM_SENSOR, AIR_QUALITY_MONITOR, DOOR_WINDOW_SENSOR
device.invalid.installation.date=La fecha de instalación no puede ser posterior a la fecha actual.
device.invalid.user.id=El ID de usuario debe ser un número positivo mayor a cero.
device.not.found=Dispositivo no encontrado.
validation.error=La validación falló.
error.unexpected=Ocurrió un error inesperado.
```

---

## PASO 9 — Domain Layer: registry

### 9.1 Value Objects

Copia los 5 value objects del zip. Están correctos. Solo cambia el package.
Agrega JavaDoc a cada uno con `@author Tu Nombre y Apellido`.

**Ejemplo para MacAddress:**
```java
/**
 * Value object representing a physical MAC address.
 * Validates the standard hexadecimal format AA:BB:CC:DD:EE:FF.
 *
 * @param value the MAC address string
 * @author Tu Nombre y Apellido
 */
public record MacAddress(String value) { ... }
```

Haz lo mismo para `Version`, `SerialNumber`, `SensiboUserId` y `DeviceTypes`.

### 9.2 Aggregate: Device

```java
package com.sensibo.platform.<CODE>.registry.domain.model.aggregates;

import com.sensibo.platform.<CODE>.registry.domain.model.commands.CreateDeviceCommand;
import com.sensibo.platform.<CODE>.registry.domain.model.valueobjects.*;
import com.sensibo.platform.<CODE>.shared.domain.model.aggregates.AbstractDomainAggregateRoot;
import lombok.Getter;
import lombok.Setter;
import java.time.LocalDate;

/**
 * Device aggregate root for the registry bounded context.
 * Represents a Sensibo IoT device registered in the platform.
 *
 * @author Tu Nombre y Apellido
 */
@Getter
public class Device extends AbstractDomainAggregateRoot {

    @Setter
    private Long id;

    private SensiboUserId userId;
    private DeviceTypes deviceType;

    @Setter
    private String modelName;

    private SerialNumber serialNumber;
    private MacAddress macAddress;
    private Version firmwareVersion;

    @Setter
    private LocalDate installationDate;

    @Setter
    private String roomLocation;

    // Needed by the persistence assembler
    public void setUserId(SensiboUserId userId) { this.userId = userId; }
    public void setDeviceType(DeviceTypes deviceType) { this.deviceType = deviceType; }
    public void setSerialNumber(SerialNumber serialNumber) { this.serialNumber = serialNumber; }
    public void setMacAddress(MacAddress macAddress) { this.macAddress = macAddress; }
    public void setFirmwareVersion(Version firmwareVersion) { this.firmwareVersion = firmwareVersion; }

    /** Default constructor required by the persistence assembler. */
    public Device() {}

    /**
     * Creates a new Device from a CreateDeviceCommand.
     * The serialNumber is auto-generated as a UUID.
     *
     * @param command the command containing all required device data
     */
    public Device(CreateDeviceCommand command) {
        this.userId = new SensiboUserId(command.userId());
        this.deviceType = DeviceTypes.fromString(command.deviceType());
        this.modelName = command.modelName();
        this.serialNumber = new SerialNumber();  // autogenerado: UUID aleatorio
        this.macAddress = new MacAddress(command.macAddress());
        this.firmwareVersion = new Version(command.firmwareVersion());
        this.installationDate = command.installationDate();
        this.roomLocation = command.roomLocation();
    }
}
```

> El enunciado dice que `serialNumber` es **autogenerado**. Por eso se usa `new SerialNumber()` (sin argumentos), que internamente hace `UUID.randomUUID().toString()`. El zip lo recibe por parámetro — eso es incorrecto según el enunciado.

### 9.3 Command

El command **no lleva serialNumber** porque es autogenerado. El enunciado lo confirma: "Los valores de id y serialNumber no se ingresan al momento de agregar".

```java
package com.sensibo.platform.<CODE>.registry.domain.model.commands;

import java.time.LocalDate;

/**
 * Command to register a new Sensibo device.
 * Note: serialNumber is auto-generated and must NOT be provided.
 *
 * @param userId            Sensibo user ID (mandatory, positive)
 * @param deviceType        device type name matching DeviceTypes enum
 * @param modelName         model name (max 50 characters)
 * @param macAddress        MAC address in AA:BB:CC:DD:EE:FF format
 * @param firmwareVersion   firmware version in X.Y.Z format
 * @param installationDate  installation date (not in the future)
 * @param roomLocation      room location description
 * @author Tu Nombre y Apellido
 */
public record CreateDeviceCommand(
        Long userId,
        String deviceType,
        String modelName,
        String macAddress,
        String firmwareVersion,
        LocalDate installationDate,
        String roomLocation
) {}
```

### 9.4 Query

```java
package com.sensibo.platform.<CODE>.registry.domain.model.queries;

/**
 * Query to retrieve a device by its identifier.
 *
 * @param deviceId the device unique identifier
 * @author Tu Nombre y Apellido
 */
public record GetDeviceById(Long deviceId) {}
```

### 9.5 Domain Repository interface

```java
package com.sensibo.platform.<CODE>.registry.domain.repositories;

import com.sensibo.platform.<CODE>.registry.domain.model.aggregates.Device;
import com.sensibo.platform.<CODE>.registry.domain.model.valueobjects.MacAddress;
import com.sensibo.platform.<CODE>.registry.domain.model.valueobjects.SerialNumber;
import java.util.Optional;

/**
 * Domain repository interface for the Device aggregate.
 *
 * @author Tu Nombre y Apellido
 */
public interface DeviceRepository {

    /**
     * Saves a device to the repository.
     *
     * @param device the device to persist
     * @return the saved device with generated id
     */
    Device save(Device device);

    /**
     * Finds a device by its unique identifier.
     *
     * @param deviceId the device id
     * @return optional containing the device if found
     */
    Optional<Device> findById(Long deviceId);

    /**
     * Checks whether a device with the given serialNumber and macAddress already exists.
     * Used to enforce the business rule: no two devices with same serialNumber + macAddress.
     *
     * @param serialNumber the serial number value object
     * @param macAddress   the MAC address value object
     * @return true if a device with those values already exists
     */
    boolean existsBySerialNumberAndMacAddress(SerialNumber serialNumber, MacAddress macAddress);
}
```

---

## PASO 10 — Application Layer

### 10.1 DeviceCommandService (interface pública)

```java
package com.sensibo.platform.<CODE>.registry.application.commandservices;

import com.sensibo.platform.<CODE>.registry.domain.model.commands.CreateDeviceCommand;
import com.sensibo.platform.<CODE>.shared.application.result.ApplicationError;
import com.sensibo.platform.<CODE>.shared.application.result.Result;

/**
 * Command service interface for Device operations (CQRS write side).
 *
 * @author Tu Nombre y Apellido
 */
public interface DeviceCommandService {

    /**
     * Handles the creation and registration of a new Device.
     *
     * @param command the CreateDeviceCommand with device registration data
     * @return Result with the generated device id on success, or ApplicationError on failure
     */
    Result<Long, ApplicationError> handle(CreateDeviceCommand command);
}
```

### 10.2 DeviceCommandServiceImpl

```java
package com.sensibo.platform.<CODE>.registry.application.internal.commandservices;

import com.sensibo.platform.<CODE>.registry.application.commandservices.DeviceCommandService;
import com.sensibo.platform.<CODE>.registry.domain.model.aggregates.Device;
import com.sensibo.platform.<CODE>.registry.domain.model.commands.CreateDeviceCommand;
import com.sensibo.platform.<CODE>.registry.domain.repositories.DeviceRepository;
import com.sensibo.platform.<CODE>.shared.application.result.ApplicationError;
import com.sensibo.platform.<CODE>.shared.application.result.Result;
import org.springframework.stereotype.Service;

/**
 * Implementation of DeviceCommandService.
 * Enforces all business rules before persisting a device.
 *
 * @author Tu Nombre y Apellido
 */
@Service
public class DeviceCommandServiceImpl implements DeviceCommandService {

    private final DeviceRepository deviceRepository;

    public DeviceCommandServiceImpl(DeviceRepository deviceRepository) {
        this.deviceRepository = deviceRepository;
    }

    /**
     * {@inheritDoc}
     *
     * Business rules enforced:
     * - No two devices with the same serialNumber and macAddress.
     * - Domain value objects validate mac format, version format, userId range,
     *   deviceType enum, and installationDate not in the future.
     */
    @Override
    public Result<Long, ApplicationError> handle(CreateDeviceCommand command) {
        // Crear el aggregate: esto genera el serialNumber automáticamente
        Device device;
        try {
            device = new Device(command);
        } catch (IllegalArgumentException e) {
            return Result.failure(ApplicationError.unexpected("Device", e.getMessage()));
        }

        // Regla de negocio: no mismo serialNumber + macAddress
        if (deviceRepository.existsBySerialNumberAndMacAddress(
                device.getSerialNumber(), device.getMacAddress())) {
            return Result.failure(
                ApplicationError.conflict("Device",
                    "device.duplicate")
            );
        }

        try {
            device = deviceRepository.save(device);
        } catch (Exception e) {
            return Result.failure(ApplicationError.unexpected("create-device", e.getMessage()));
        }

        return Result.success(device.getId());
    }
}
```

### 10.3 DeviceQueryService (interface + impl)

```java
// application/queryservices/DeviceQueryService.java
package com.sensibo.platform.<CODE>.registry.application.queryservices;

import com.sensibo.platform.<CODE>.registry.domain.model.aggregates.Device;
import com.sensibo.platform.<CODE>.registry.domain.model.queries.GetDeviceById;
import java.util.Optional;

/**
 * Query service interface for Device queries (CQRS read side).
 *
 * @author Tu Nombre y Apellido
 */
public interface DeviceQueryService {

    /**
     * Retrieves a device by its identifier.
     *
     * @param query the GetDeviceById query
     * @return optional containing the device if found
     */
    Optional<Device> handle(GetDeviceById query);
}
```

```java
// application/internal/queryservices/DeviceQueryServiceImpl.java
package com.sensibo.platform.<CODE>.registry.application.internal.queryservices;

import com.sensibo.platform.<CODE>.registry.application.queryservices.DeviceQueryService;
import com.sensibo.platform.<CODE>.registry.domain.model.aggregates.Device;
import com.sensibo.platform.<CODE>.registry.domain.model.queries.GetDeviceById;
import com.sensibo.platform.<CODE>.registry.domain.repositories.DeviceRepository;
import org.springframework.stereotype.Service;
import java.util.Optional;

/**
 * Implementation of DeviceQueryService.
 *
 * @author Tu Nombre y Apellido
 */
@Service
public class DeviceQueryServiceImpl implements DeviceQueryService {

    private final DeviceRepository deviceRepository;

    public DeviceQueryServiceImpl(DeviceRepository deviceRepository) {
        this.deviceRepository = deviceRepository;
    }

    /** {@inheritDoc} */
    @Override
    public Optional<Device> handle(GetDeviceById query) {
        return deviceRepository.findById(query.deviceId());
    }
}
```

---

## PASO 11 — Infrastructure Layer

### 11.1 DevicePersistenceEntity (estaba vacía en el zip)

```java
package com.sensibo.platform.<CODE>.registry.infrastructure.persistence.jpa.entities;

import com.sensibo.platform.<CODE>.registry.domain.model.valueobjects.*;
import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;
import java.time.LocalDate;
import java.util.Date;

/**
 * JPA persistence entity for the Device aggregate.
 * Maps to the devices table in the sensibo schema.
 *
 * @author Tu Nombre y Apellido
 */
@Getter
@Setter
@Entity
@EntityListeners(AuditingEntityListener.class)
public class DevicePersistenceEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // userId: Long almacenado via converter
    @Column(name = "user_id", nullable = false)
    private SensiboUserId userId;

    // deviceType: almacenado como String (nombre del enum)
    @Enumerated(EnumType.STRING)
    @Column(name = "device_type", nullable = false)
    private DeviceTypes deviceType;

    @Column(name = "model_name", nullable = false, length = 50)
    private String modelName;

    // serialNumber: UUID almacenado como String via converter
    @Column(name = "serial_number", nullable = false, unique = true)
    private SerialNumber serialNumber;

    // macAddress: String via converter
    @Column(name = "mac_address", nullable = false)
    private MacAddress macAddress;

    // firmwareVersion: String via converter
    @Column(name = "firmware_version", nullable = false)
    private Version firmwareVersion;

    @Column(name = "installation_date", nullable = false)
    private LocalDate installationDate;

    @Column(name = "room_location", nullable = false)
    private String roomLocation;

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private Date createdAt;

    @LastModifiedDate
    @Column(name = "updated_at", nullable = false)
    private Date updatedAt;
}
```

### 11.2 DevicePersistenceRepository (no estaba en el zip)

```java
package com.sensibo.platform.<CODE>.registry.infrastructure.persistence.jpa.repositories;

import com.sensibo.platform.<CODE>.registry.domain.model.valueobjects.MacAddress;
import com.sensibo.platform.<CODE>.registry.domain.model.valueobjects.SerialNumber;
import com.sensibo.platform.<CODE>.registry.infrastructure.persistence.jpa.entities.DevicePersistenceEntity;
import org.springframework.data.jpa.repository.JpaRepository;

/**
 * Spring Data JPA repository for DevicePersistenceEntity.
 *
 * @author Tu Nombre y Apellido
 */
public interface DevicePersistenceRepository extends JpaRepository<DevicePersistenceEntity, Long> {

    /**
     * Checks if a device with the given serialNumber and macAddress already exists.
     *
     * @param serialNumber the serial number
     * @param macAddress   the MAC address
     * @return true if an existing device matches both values
     */
    boolean existsBySerialNumberAndMacAddress(SerialNumber serialNumber, MacAddress macAddress);
}
```

### 11.3 Converters (copiar del zip, están bien)

Copia los 4 converters del zip cambiando solo el package. Son correctos:
- `MacAddressPersistenceConverter`
- `SerialNumberPersistenceConverter`
- `VersionPersistenceConverter`
- `SensiboUserIdPersistenceConverter` — verifica que use `attribute.userId()` si tu VO usa ese campo, o `attribute.value()` si lo renombraste

### 11.4 DevicePersistenceAssembler (copiar del zip, está bien)

Copia el assembler del zip cambiando el package. Está correctamente implementado.

### 11.5 DeviceRepositoryImpl (copiar del zip, está bien)

Copia el adapter del zip cambiando el package. Está correctamente implementado.

---

## PASO 12 — Interfaces Layer: REST (no está en el zip)

### 12.1 CreateDeviceResource — request body

El enunciado dice que el request recibe: `userId, deviceType, modelName, macAddress, firmwareVersion, installationDate, roomLocation`. **No recibe serialNumber** (autogenerado) ni `id`.

```java
package com.sensibo.platform.<CODE>.registry.interfaces.rest.resources;

import com.fasterxml.jackson.annotation.JsonFormat;
import jakarta.validation.constraints.*;
import java.time.LocalDate;

/**
 * Resource for registering a new Sensibo device.
 * serialNumber is excluded — it is auto-generated.
 *
 * @param userId            Sensibo user ID (mandatory, positive)
 * @param deviceType        device type name (must match DeviceTypes enum)
 * @param modelName         model name (max 50 characters)
 * @param macAddress        MAC address in AA:BB:CC:DD:EE:FF format
 * @param firmwareVersion   firmware version in X.Y.Z format
 * @param installationDate  installation date, not in the future
 * @param roomLocation      room location description
 * @author Tu Nombre y Apellido
 */
public record CreateDeviceResource(

        @NotNull
        @Positive
        Long userId,

        @NotBlank
        String deviceType,

        @NotBlank
        @Size(max = 50)
        String modelName,

        @NotBlank
        @Pattern(
            regexp = "^([0-9A-Fa-f]{2}:){5}[0-9A-Fa-f]{2}$",
            message = "device.invalid.mac"
        )
        String macAddress,

        @NotBlank
        @Pattern(
            regexp = "^\\d+\\.\\d+\\.\\d+$",
            message = "device.invalid.version"
        )
        String firmwareVersion,

        @NotNull
        @PastOrPresent(message = "device.invalid.installation.date")
        LocalDate installationDate,

        @NotBlank
        String roomLocation
) {}
```

### 12.2 DeviceResource — response body

El enunciado especifica exactamente qué incluir: `id, serialNumber (como String), deviceType (como String), modelName, macAddress (como String), firmwareVersion (como String), installationDate, roomLocation`.

```java
package com.sensibo.platform.<CODE>.registry.interfaces.rest.resources;

import java.time.LocalDate;

/**
 * Resource representing a registered Device in API responses.
 * All value objects are returned as plain Strings as specified.
 *
 * @param id               auto-generated device identifier
 * @param serialNumber     auto-generated UUID as String
 * @param deviceType       device type name as String
 * @param modelName        model name
 * @param macAddress       MAC address as String
 * @param firmwareVersion  firmware version as String
 * @param installationDate installation date
 * @param roomLocation     room location
 * @author Tu Nombre y Apellido
 */
public record DeviceResource(
        Long id,
        String serialNumber,
        String deviceType,
        String modelName,
        String macAddress,
        String firmwareVersion,
        LocalDate installationDate,
        String roomLocation
) {}
```

### 12.3 Transform Assemblers

```java
// interfaces/rest/transform/CreateDeviceCommandFromResourceAssembler.java
package com.sensibo.platform.<CODE>.registry.interfaces.rest.transform;

import com.sensibo.platform.<CODE>.registry.domain.model.commands.CreateDeviceCommand;
import com.sensibo.platform.<CODE>.registry.interfaces.rest.resources.CreateDeviceResource;

/**
 * Assembler that converts a CreateDeviceResource into a CreateDeviceCommand.
 *
 * @author Tu Nombre y Apellido
 */
public class CreateDeviceCommandFromResourceAssembler {

    /**
     * Converts the REST request resource to a domain command.
     *
     * @param resource the incoming REST request body
     * @return CreateDeviceCommand ready for the command service
     */
    public static CreateDeviceCommand toCommandFromResource(CreateDeviceResource resource) {
        return new CreateDeviceCommand(
                resource.userId(),
                resource.deviceType(),
                resource.modelName(),
                resource.macAddress(),
                resource.firmwareVersion(),
                resource.installationDate(),
                resource.roomLocation()
        );
    }
}
```

```java
// interfaces/rest/transform/DeviceResourceFromEntityAssembler.java
package com.sensibo.platform.<CODE>.registry.interfaces.rest.transform;

import com.sensibo.platform.<CODE>.registry.domain.model.aggregates.Device;
import com.sensibo.platform.<CODE>.registry.interfaces.rest.resources.DeviceResource;

/**
 * Assembler that converts a Device domain aggregate to a DeviceResource for API responses.
 *
 * @author Tu Nombre y Apellido
 */
public class DeviceResourceFromEntityAssembler {

    /**
     * Converts a Device to its corresponding REST response resource.
     * All value objects are unwrapped to their plain String or primitive values
     * as required by the API specification.
     *
     * @param device the Device domain aggregate
     * @return DeviceResource for the HTTP response body
     */
    public static DeviceResource toResourceFromEntity(Device device) {
        return new DeviceResource(
                device.getId(),
                device.getSerialNumber().value(),        // serialNumber como String
                device.getDeviceType().name(),           // deviceType como String
                device.getModelName(),
                device.getMacAddress().value(),          // macAddress como String
                device.getFirmwareVersion().value(),     // firmwareVersion como String
                device.getInstallationDate(),
                device.getRoomLocation()
        );
    }
}
```

### 12.4 DevicesController

```java
package com.sensibo.platform.<CODE>.registry.interfaces.rest;

import com.sensibo.platform.<CODE>.registry.application.commandservices.DeviceCommandService;
import com.sensibo.platform.<CODE>.registry.application.queryservices.DeviceQueryService;
import com.sensibo.platform.<CODE>.registry.domain.model.queries.GetDeviceById;
import com.sensibo.platform.<CODE>.registry.interfaces.rest.resources.CreateDeviceResource;
import com.sensibo.platform.<CODE>.registry.interfaces.rest.resources.DeviceResource;
import com.sensibo.platform.<CODE>.registry.interfaces.rest.transform.CreateDeviceCommandFromResourceAssembler;
import com.sensibo.platform.<CODE>.registry.interfaces.rest.transform.DeviceResourceFromEntityAssembler;
import com.sensibo.platform.<CODE>.shared.application.result.ApplicationError;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.media.Content;
import io.swagger.v3.oas.annotations.media.Schema;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import org.springframework.context.MessageSource;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.*;

import java.util.Locale;
import java.util.Map;
import java.util.stream.Collectors;

/**
 * REST controller exposing the /api/v1/devices endpoint.
 * Handles device registration for the Sensibo platform.
 *
 * @author Tu Nombre y Apellido
 */
@RestController
@RequestMapping(value = "/api/v1/devices", produces = MediaType.APPLICATION_JSON_VALUE)
@Tag(name = "Devices", description = "Sensibo device registration and management")
public class DevicesController {

    private final DeviceCommandService deviceCommandService;
    private final DeviceQueryService deviceQueryService;
    private final MessageSource messageSource;

    public DevicesController(DeviceCommandService deviceCommandService,
                             DeviceQueryService deviceQueryService,
                             MessageSource messageSource) {
        this.deviceCommandService = deviceCommandService;
        this.deviceQueryService = deviceQueryService;
        this.messageSource = messageSource;
    }

    /**
     * Registers a new Sensibo device.
     * The serialNumber and id are auto-generated.
     *
     * @param resource the device registration request body
     * @param locale   resolved from the Accept-Language header (EN, EN-US, ES, ES-PE)
     * @return 201 Created with DeviceResource on success, or error response
     */
    @PostMapping
    @Operation(
        summary = "Register a new device",
        description = "Registers a new Sensibo device. The fields 'id' and 'serialNumber' are auto-generated and must not be included in the request."
    )
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "Device registered successfully",
                     content = @Content(schema = @Schema(implementation = DeviceResource.class))),
        @ApiResponse(responseCode = "400", description = "Validation error — invalid field values"),
        @ApiResponse(responseCode = "409", description = "Conflict — duplicate serialNumber + macAddress")
    })
    public ResponseEntity<?> createDevice(
            @Valid @RequestBody CreateDeviceResource resource,
            Locale locale) {

        try {
            var command = CreateDeviceCommandFromResourceAssembler.toCommandFromResource(resource);
            var result = deviceCommandService.handle(command);

            if (result.isFailure()) {
                var error = result.getError();
                var httpStatus = switch (error.type()) {
                    case CONFLICT -> HttpStatus.CONFLICT;
                    case NOT_FOUND -> HttpStatus.NOT_FOUND;
                    case VALIDATION -> HttpStatus.BAD_REQUEST;
                    default -> HttpStatus.INTERNAL_SERVER_ERROR;
                };
                // Intentar resolver el mensaje como clave i18n
                String msgKey = error.message().contains(":") 
                    ? error.message().substring(error.message().indexOf(":") + 1).trim()
                    : error.message();
                String msg = messageSource.getMessage(msgKey, null, msgKey, locale);
                return ResponseEntity.status(httpStatus).body(Map.of("error", msg));
            }

            var deviceOpt = deviceQueryService.handle(new GetDeviceById(result.getValue()));
            if (deviceOpt.isEmpty()) {
                String msg = messageSource.getMessage("error.unexpected", null, locale);
                return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                        .body(Map.of("error", msg));
            }

            return ResponseEntity
                    .status(HttpStatus.CREATED)
                    .body(DeviceResourceFromEntityAssembler.toResourceFromEntity(deviceOpt.get()));

        } catch (IllegalArgumentException e) {
            return ResponseEntity.badRequest().body(Map.of("error", e.getMessage()));
        }
    }

    /**
     * Handles Jakarta Bean Validation errors with localized messages.
     *
     * @param ex     the validation exception
     * @param locale the request locale
     * @return 400 Bad Request with field-level error details
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<?> handleValidationErrors(
            MethodArgumentNotValidException ex, Locale locale) {

        var details = ex.getBindingResult().getFieldErrors().stream()
                .collect(Collectors.toMap(
                        fe -> fe.getField(),
                        fe -> {
                            String key = fe.getDefaultMessage();
                            return messageSource.getMessage(key, null, key, locale);
                        },
                        (a, b) -> a
                ));

        return ResponseEntity.badRequest().body(Map.of(
                "error", messageSource.getMessage("validation.error", null, locale),
                "details", details
        ));
    }
}
```

---

## PASO 13 — README.md correcto

El del zip dice "ACME Learning Center". Reemplázalo con contenido de Sensibo:

```markdown
# Sensibo Climate Control Platform — Device Registry API

RESTful API for Sensibo IoT device registration. Built with Java 25 and Spring Boot 3.5.0,
applying Domain-Driven Design (DDD), layered architecture, CQRS pattern, and bounded contexts.

## Author

- **Name:** Tu Nombre y Apellido
- **Student Code:** u<codigo>
- **Course:** Desarrollo de Aplicaciones Open Source (1ASI0729)
- **University:** Universidad Peruana de Ciencias Aplicadas (UPC)

## Tech Stack

- Java 25 · Spring Boot 3.5.0 · Spring Data JPA · PostgreSQL
- SpringDoc OpenAPI (Swagger UI) · Lombok · Maven

## Setup

### 1. Database
```sql
CREATE DATABASE sensibo;
\c sensibo
CREATE SCHEMA IF NOT EXISTS sensibo;
```

### 2. Run
```bash
mvn spring-boot:run
```

### 3. API Docs
- Swagger UI: http://localhost:8096/swagger-ui/index.html
- OpenAPI JSON: http://localhost:8096/v3/api-docs

## Endpoint

| Method | Path | Description |
|--------|------|-------------|
| POST | /api/v1/devices | Register a new Sensibo device |

## Request Example

```json
{
  "userId": 1,
  "deviceType": "SMART_AC_CONTROLLER",
  "modelName": "Sensibo Air PRO",
  "macAddress": "AA:BB:CC:DD:EE:FF",
  "firmwareVersion": "2.1.0",
  "installationDate": "2025-01-15",
  "roomLocation": "Living Room"
}
```

## Response Example (201 Created)

```json
{
  "id": 1,
  "serialNumber": "550e8400-e29b-41d4-a716-446655440000",
  "deviceType": "SMART_AC_CONTROLLER",
  "modelName": "Sensibo Air PRO",
  "macAddress": "AA:BB:CC:DD:EE:FF",
  "firmwareVersion": "2.1.0",
  "installationDate": "2025-01-15",
  "roomLocation": "Living Room"
}
```
```

---

## PASO 14 — Checklist antes de comprimir

```
C01 — Building y ejecución
  □ Arranca sin errores en puerto 8096
  □ Swagger UI accesible: http://localhost:8096/swagger-ui/index.html
  □ Tabla "devices" creada en esquema "sensibo" de PostgreSQL

C02 — Endpoint
  □ POST /api/v1/devices → 201 con id y serialNumber generados
  □ Request NO incluye serialNumber (es autogenerado)
  □ Response incluye: id, serialNumber, deviceType, modelName, macAddress, firmwareVersion, installationDate, roomLocation
  □ Todos son String/Long/Date — no objetos anidados
  □ Tabla en BD: nombre snake_case plural ("devices"), columnas en snake_case

C04 — Business Rules
  □ macAddress duplicada con mismo serialNumber → 409
  □ macAddress con formato inválido → 400
  □ firmwareVersion con formato inválido (no X.Y.Z) → 400
  □ userId null o ≤ 0 → 400
  □ deviceType inválido (no es uno de los 4) → 400
  □ installationDate futura → 400
  □ modelName en blanco o nulo → 400
  □ roomLocation en blanco o nulo → 400

C05 — Code Organization
  □ Bounded contexts: registry y shared bien separados
  □ Dentro de registry: domain / application / infrastructure / interfaces
  □ application/internal para implementaciones, application/ para interfaces
  □ interfaces/rest/ con resources/ y transform/
  □ shared completo con sus sub-packages

C06 — Code Quality
  □ Puerto 8096 ✓
  □ Schema "sensibo" en application.properties ✓
  □ SnakeCaseWithPluralizedTablePhysicalNamingStrategy configurada ✓
  □ @EnableJpaAuditing en clase principal ✓
  □ messages.properties (EN) + messages_es.properties (ES) ✓
  □ Accept-Language header funciona con EN, EN-US, ES, ES-PE ✓
  □ Swagger con @Tag, @Operation, @ApiResponse ✓
  □ README habla de Sensibo con tu nombre ✓
  □ Todos los textos de mensajes en inglés en el código ✓

C07 — Naming
  □ Packages: inglés, lowercase
  □ Clases: UpperCamelCase
  □ Métodos y atributos: lowerCamelCase
  □ URLs: /api/v1/devices (minúsculas, plural)
  □ Tablas BD: plural snake_case ("devices")
  □ Columnas BD: snake_case (user_id, device_type, model_name, etc.)
  □ JavaDoc en inglés con @author en clases y métodos principales
  □ Nombre del zip: pc2<NRC><CODE>.zip
```

---

## DIFERENCIAS CONCRETAS ENTRE EL ZIP Y LO QUE PIDE EL ENUNCIADO

| Punto | ZIP de referencia | Enunciado pide | Tu solución |
|-------|-----------------|----------------|-------------|
| serialNumber en Command | Sí está como parámetro | No se ingresa, es autogenerado | NO va en el Command |
| serialNumber en Device | `new SerialNumber(command.serialNumber())` | UUID autogenerado al almacenar | `new SerialNumber()` sin args |
| DevicePersistenceEntity | Archivo vacío | Entidad JPA completa | Completa con @Entity y columnas |
| DevicePersistenceRepository | No existe | Spring Data JPA repo | Crear con el método exists |
| interfaces/rest/ | No existe | Controller + Resources + Transform | Crear completo |
| messages.properties | No existe | i18n EN y ES requerido | Crear ambos archivos |
| README.md | Dice "ACME Learning Center" | Info de Sensibo + autor | Reescribir |
| Spring Boot | 4.0.7 | 3.5.0 (technical constraint #1) | Usar 3.5.0 |
| Java | 26 | 25 (technical constraint #1) | Usar 25 |
