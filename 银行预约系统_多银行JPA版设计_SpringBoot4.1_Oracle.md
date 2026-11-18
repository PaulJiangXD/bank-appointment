# 银行网点预约系统 · 多银行 JPA 版设计文档

**版本：** 3.0（Spring Boot 4.1.x）  
**日期：** 2026-11-18  
**技术栈：** Spring Boot 4.1.x · Java 21+ · Spring Data JPA · Hibernate 7.x · Oracle 19c+ · jakarta.persistence 3.x

---

## 一、架构概览

```
多银行（同一集团/独立运营均可）
└── bank（银行/业务条线）
     └── branch（网点）
          └── branch_capacity（时段容量）
               └── appointment（预约）
```

**核心设计原则：**
- `bank_id` 为第一隔离维度，贯穿所有业务表
- 个人银行、商业银行共用一套系统，数据按 `bank_id` 隔离
- 客户表跨银行共享（手机号全局唯一）
- 乐观锁防超卖，无 `SELECT FOR UPDATE`，不死锁
- 节假日支持银行自定义，兜底全局默认

---

## 二、与 Spring Boot 3.x 版本的关键差异

| 项目 | Spring Boot 3.x | Spring Boot 4.1.x |
|------|----------------|-------------------|
| Java 基线 | 17+ | **21+** |
| Hibernate | 6.x | **7.x** |
| Jakarta EE | 10 | **11** |
| `@Table(uniqueConstraints)` | 正常 | 属性名变更，改用 `@UniqueConstraint` |
| 自动配置 | `@EnableAutoConfiguration` | 新增 `spring.config.import` 强制校验 |
| 依赖管理 | `spring-boot-starter-*` | 部分 starter 拆分，需显式引入 |
| 虚拟线程 | 需手动开启 | **默认启用**（Tomcat 虚拟线程） |

---

## 三、数据库设计（Oracle DDL）

### 3.1 序列

```sql
CREATE SEQUENCE seq_bank               START WITH 1 INCREMENT BY 1 NOCACHE;
CREATE SEQUENCE seq_branch             START WITH 1 INCREMENT BY 1 NOCACHE;
CREATE SEQUENCE seq_branch_capacity    START WITH 1 INCREMENT BY 1 NOCACHE;
CREATE SEQUENCE seq_appointment        START WITH 1 INCREMENT BY 1 NOCACHE;
CREATE SEQUENCE seq_appointment_log    START WITH 1 INCREMENT BY 1 NOCACHE;
CREATE SEQUENCE seq_customer           START WITH 1 INCREMENT BY 1 NOCACHE;
```

### 3.2 银行表

```sql
CREATE TABLE bank (
    id              NUMBER(19) PRIMARY KEY,
    bank_code       VARCHAR2(50) NOT NULL UNIQUE,
    bank_name       VARCHAR2(200) NOT NULL,
    bank_type       NUMBER(1) NOT NULL,
    logo_url        VARCHAR2(500),
    contact_phone   VARCHAR2(50),
    status          NUMBER(1) DEFAULT 1 NOT NULL,
    created_at      DATE DEFAULT SYSDATE,
    updated_at      DATE DEFAULT SYSDATE
);

COMMENT ON TABLE bank IS '银行/业务条线表';
COMMENT ON COLUMN bank.bank_type IS '1-个人银行 2-商业银行 3-信用卡 4-投行';
```

### 3.3 网点表

```sql
CREATE TABLE branch (
    id              NUMBER(19) PRIMARY KEY,
    bank_id         NUMBER(19) NOT NULL,
    branch_code     VARCHAR2(50) NOT NULL,
    branch_name     VARCHAR2(200) NOT NULL,
    address         VARCHAR2(500),
    phone           VARCHAR2(50),
    business_start  VARCHAR2(8) DEFAULT '09:00:00' NOT NULL,
    business_end    VARCHAR2(8) DEFAULT '17:00:00' NOT NULL,
    slot_minutes    NUMBER(3) DEFAULT 30 NOT NULL,
    default_cap     NUMBER(5) DEFAULT 5 NOT NULL,
    status          NUMBER(1) DEFAULT 1 NOT NULL,
    created_at      DATE DEFAULT SYSDATE,
    updated_at      DATE DEFAULT SYSDATE,
    CONSTRAINT uk_bank_branch_code UNIQUE (bank_id, branch_code),
    CONSTRAINT fk_branch_bank FOREIGN KEY (bank_id) REFERENCES bank(id)
);

CREATE INDEX idx_branch_bank ON branch(bank_id);
```

### 3.4 时段容量表

```sql
CREATE TABLE branch_capacity (
    id              NUMBER(19) PRIMARY KEY,
    bank_id         NUMBER(19) NOT NULL,
    branch_id       NUMBER(19) NOT NULL,
    capacity_date   DATE NOT NULL,
    time_slot       VARCHAR2(20) NOT NULL,
    slot_start      VARCHAR2(8) NOT NULL,
    total_cap       NUMBER(5) NOT NULL,
    used_count      NUMBER(5) DEFAULT 0 NOT NULL,
    version         NUMBER(10) DEFAULT 0 NOT NULL,
    created_at      DATE DEFAULT SYSDATE,
    updated_at      DATE DEFAULT SYSDATE,
    CONSTRAINT uk_bank_branch_slot UNIQUE (bank_id, branch_id, capacity_date, time_slot),
    CONSTRAINT fk_cap_bank FOREIGN KEY (bank_id) REFERENCES bank(id),
    CONSTRAINT fk_cap_branch FOREIGN KEY (branch_id) REFERENCES branch(id)
);

CREATE INDEX idx_cap_bank_date ON branch_capacity(bank_id, capacity_date);
```

### 3.5 预约表

```sql
CREATE TABLE appointment (
    id              NUMBER(19) PRIMARY KEY,
    appointment_no  VARCHAR2(50) NOT NULL UNIQUE,
    bank_id         NUMBER(19) NOT NULL,
    branch_id       NUMBER(19) NOT NULL,
    customer_id     NUMBER(19),
    customer_name   VARCHAR2(100) NOT NULL,
    phone           VARCHAR2(50) NOT NULL,
    id_card_enc     VARCHAR2(255),
    appt_date       DATE NOT NULL,
    time_slot       VARCHAR2(20) NOT NULL,
    slot_start      VARCHAR2(8) NOT NULL,
    slot_end        VARCHAR2(8) NOT NULL,
    service_type    VARCHAR2(50) NOT NULL,
    queue_no        VARCHAR2(20),
    status          NUMBER(1) DEFAULT 1 NOT NULL,
    source          VARCHAR2(30),
    cancel_reason   VARCHAR2(300),
    remark          VARCHAR2(500),
    expire_at       DATE,
    created_at      DATE DEFAULT SYSDATE,
    updated_at      DATE DEFAULT SYSDATE,
    CONSTRAINT fk_appt_bank FOREIGN KEY (bank_id) REFERENCES bank(id),
    CONSTRAINT fk_appt_branch FOREIGN KEY (branch_id) REFERENCES branch(id)
);

-- 同一银行内，同一手机号同一天只能有一个有效预约
CREATE UNIQUE INDEX uk_bank_phone_date_active ON appointment(bank_id, phone, appt_date, status);

CREATE INDEX idx_appt_bank_date ON appointment(bank_id, branch_id, appt_date, status);
CREATE INDEX idx_appt_bank_status_expire ON appointment(bank_id, status, expire_at);
```

### 3.6 客户表

```sql
CREATE TABLE customer (
    id              NUMBER(19) PRIMARY KEY,
    phone           VARCHAR2(50) NOT NULL UNIQUE,
    customer_name   VARCHAR2(100),
    id_card_enc     VARCHAR2(255),
    created_at      DATE DEFAULT SYSDATE,
    updated_at      DATE DEFAULT SYSDATE
);

COMMENT ON TABLE customer IS '客户表（跨银行共享）';
```

### 3.7 节假日表

```sql
CREATE TABLE national_holiday (
    bank_id         NUMBER(19) DEFAULT 0 NOT NULL,
    holiday_date    DATE NOT NULL,
    holiday_name    VARCHAR2(50),
    day_type        NUMBER(1) NOT NULL,
    holiday_year    NUMBER(4) NOT NULL,
    PRIMARY KEY (bank_id, holiday_date),
    CONSTRAINT fk_holiday_bank FOREIGN KEY (bank_id) REFERENCES bank(id)
);

COMMENT ON COLUMN national_holiday.bank_id IS '0=全局默认，其他=银行自定义';
COMMENT ON COLUMN national_holiday.day_type IS '1-法定假日 2-调休工作日';
```

### 3.8 特殊日期表

```sql
CREATE TABLE branch_special_date (
    bank_id         NUMBER(19) NOT NULL,
    branch_id       NUMBER(19) NOT NULL,
    special_date    DATE NOT NULL,
    is_closed       NUMBER(1) DEFAULT 0 NOT NULL,
    cap_override    NUMBER(5),
    reason          VARCHAR2(200),
    created_at      DATE DEFAULT SYSDATE,
    PRIMARY KEY (bank_id, branch_id, special_date),
    CONSTRAINT fk_special_branch FOREIGN KEY (branch_id) REFERENCES branch(id)
);

COMMENT ON TABLE branch_special_date IS '网点特殊日期配置';
COMMENT ON COLUMN branch_special_date.is_closed IS '0-正常 1-闭店';
```

### 3.9 预约日志表

```sql
CREATE TABLE appointment_log (
    id              NUMBER(19) PRIMARY KEY,
    appointment_id  NUMBER(19) NOT NULL,
    appointment_no  VARCHAR2(50) NOT NULL,
    bank_id         NUMBER(19) NOT NULL,
    operation       VARCHAR2(30) NOT NULL,
    operator_type   VARCHAR2(20) NOT NULL,
    operator_id     VARCHAR2(100),
    operator_name   VARCHAR2(100),
    from_status     NUMBER(1),
    to_status       NUMBER(1),
    remark          VARCHAR2(500),
    created_at      DATE DEFAULT SYSDATE,
    CONSTRAINT fk_log_appt FOREIGN KEY (appointment_id) REFERENCES appointment(id)
);

COMMENT ON TABLE appointment_log IS '预约操作日志';
CREATE INDEX idx_log_appt_id ON appointment_log(appointment_id);
CREATE INDEX idx_log_bank ON appointment_log(bank_id);
```

---

## 四、pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>4.1.2</version>
    </parent>

    <groupId>com.bank</groupId>
    <artifactId>appointment-system</artifactId>
    <version>1.0.0</version>
    <name>Bank Appointment System</name>

    <properties>
        <java.version>21</java.version>
        <oracle.jdbc.version>23.7.0.25.01</oracle.jdbc.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <dependency>
            <groupId>com.oracle.database.jdbc</groupId>
            <artifactId>ojdbc11</artifactId>
            <version>${oracle.jdbc.version}</version>
        </dependency>

        <dependency>
            <groupId>com.oracle.database.jdbc</groupId>
            <artifactId>ucp11</artifactId>
            <version>${oracle.jdbc.version}</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## 五、application.yml

```yaml
server:
  port: 8080
  tomcat:
    threads:
      virtual:
        enabled: true  # Spring Boot 4.x 默认虚拟线程

spring:
  application:
    name: bank-appointment-system

  datasource:
    url: jdbc:oracle:thin:@//localhost:1521/ORCLPDB1
    username: ${DB_USERNAME:appointment_user}
    password: ${DB_PASSWORD}
    driver-class-name: oracle.jdbc.OracleDriver
    type: oracle.ucp.jdbc.PoolDataSource
    oracleucp:
      initial-pool-size: 10
      max-pool-size: 50
      min-pool-size: 5
      connection-factory-class-name: oracle.jdbc.pool.OracleDataSource

  jpa:
    database-platform: org.hibernate.dialect.OracleDialect
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate.format_sql: true
      hibernate.jdbc.batch_size: 50
      hibernate.order_inserts: true
      hibernate.order_updates: true
      hibernate.generate_statistics: false

  task:
    scheduling:
      enabled: true

logging:
  level:
    com.bank.appointment: DEBUG
    org.hibernate.SQL: DEBUG
```

---

## 六、实体类（Hibernate 7.x 适配）

### 6.1 Bank

```java
package com.bank.appointment.entity;

import jakarta.persistence.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.io.Serializable;
import java.time.LocalDateTime;

@Entity
@Table(name = "bank", uniqueConstraints = {
    @UniqueConstraint(name = "uk_bank_code", columnNames = "bank_code")
})
@EntityListeners(AuditingEntityListener.class)
public class Bank implements Serializable {

    @Id
    @Column(name = "id")
    private Long id;

    @Column(name = "bank_code", nullable = false, length = 50)
    private String bankCode;

    @Column(name = "bank_name", nullable = false, length = 200)
    private String bankName;

    /** 1-个人银行 2-商业银行 3-信用卡 4-投行 */
    @Column(name = "bank_type", nullable = false)
    private Integer bankType;

    @Column(name = "logo_url", length = 500)
    private String logoUrl;

    @Column(name = "contact_phone", length = 50)
    private String contactPhone;

    @Column(name = "status", nullable = false)
    private Integer status = 1;

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    // getters / setters
}
```

### 6.2 Branch

```java
package com.bank.appointment.entity;

import jakarta.persistence.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.io.Serializable;
import java.time.LocalDateTime;

@Entity
@Table(name = "branch", uniqueConstraints = {
    @UniqueConstraint(name = "uk_bank_branch_code", columnNames = {"bank_id", "branch_code"})
}, indexes = {
    @Index(name = "idx_branch_bank", columnList = "bank_id")
})
@EntityListeners(AuditingEntityListener.class)
public class Branch implements Serializable {

    @Id
    @Column(name = "id")
    private Long id;

    @Column(name = "bank_id", nullable = false)
    private Long bankId;

    @Column(name = "branch_code", nullable = false, length = 50)
    private String branchCode;

    @Column(name = "branch_name", nullable = false, length = 200)
    private String branchName;

    @Column(name = "address", length = 500)
    private String address;

    @Column(name = "phone", length = 50)
    private String phone;

    @Column(name = "business_start", nullable = false, length = 8)
    private String businessStart = "09:00:00";

    @Column(name = "business_end", nullable = false, length = 8)
    private String businessEnd = "17:00:00";

    @Column(name = "slot_minutes", nullable = false)
    private Integer slotMinutes = 30;

    @Column(name = "default_cap", nullable = false)
    private Integer defaultCap = 5;

    @Column(name = "status", nullable = false)
    private Integer status = 1;

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    // getters / setters
}
```

### 6.3 BranchCapacity（乐观锁核心）

```java
package com.bank.appointment.entity;

import jakarta.persistence.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.io.Serializable;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "branch_capacity", uniqueConstraints = {
    @UniqueConstraint(name = "uk_bank_branch_slot",
        columnNames = {"bank_id", "branch_id", "capacity_date", "time_slot"})
}, indexes = {
    @Index(name = "idx_cap_bank_date", columnList = "bank_id, capacity_date")
})
@EntityListeners(AuditingEntityListener.class)
public class BranchCapacity implements Serializable {

    @Id
    @Column(name = "id")
    private Long id;

    @Column(name = "bank_id", nullable = false)
    private Long bankId;

    @Column(name = "branch_id", nullable = false)
    private Long branchId;

    @Column(name = "capacity_date", nullable = false)
    private LocalDate capacityDate;

    @Column(name = "time_slot", nullable = false, length = 20)
    private String timeSlot;

    @Column(name = "slot_start", nullable = false, length = 8)
    private String slotStart;

    @Column(name = "total_cap", nullable = false)
    private Integer totalCap;

    @Column(name = "used_count", nullable = false)
    private Integer usedCount = 0;

    /**
     * Hibernate 7.x 乐观锁
     * UPDATE 自动带 WHERE version = ?，版本不匹配抛 OptimisticLockingFailureException
     */
    @Version
    @Column(name = "version", nullable = false)
    private Integer version = 0;

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    // getters / setters
}
```

### 6.4 Appointment

```java
package com.bank.appointment.entity;

import jakarta.persistence.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.io.Serializable;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "appointment", indexes = {
    @Index(name = "idx_appt_bank_date", columnList = "bank_id, branch_id, appt_date, status"),
    @Index(name = "idx_appt_bank_status_expire", columnList = "bank_id, status, expire_at")
})
@EntityListeners(AuditingEntityListener.class)
public class Appointment implements Serializable {

    @Id
    @Column(name = "id")
    private Long id;

    @Column(name = "appointment_no", nullable = false, unique = true, length = 50)
    private String appointmentNo;

    @Column(name = "bank_id", nullable = false)
    private Long bankId;

    @Column(name = "branch_id", nullable = false)
    private Long branchId;

    @Column(name = "customer_id")
    private Long customerId;

    @Column(name = "customer_name", nullable = false, length = 100)
    private String customerName;

    @Column(name = "phone", nullable = false, length = 50)
    private String phone;

    @Column(name = "id_card_enc", length = 255)
    private String idCardEnc;

    @Column(name = "appt_date", nullable = false)
    private LocalDate apptDate;

    @Column(name = "time_slot", nullable = false, length = 20)
    private String timeSlot;

    @Column(name = "slot_start", nullable = false, length = 8)
    private String slotStart;

    @Column(name = "slot_end", nullable = false, length = 8)
    private String slotEnd;

    @Column(name = "service_type", nullable = false, length = 50)
    private String serviceType;

    @Column(name = "queue_no", length = 20)
    private String queueNo;

    /** 0-待确认 1-已预约 2-已完成 3-已取消 4-已过号 */
    @Column(name = "status", nullable = false)
    private Integer status = 1;

    @Column(name = "source", length = 30)
    private String source;

    @Column(name = "cancel_reason", length = 300)
    private String cancelReason;

    @Column(name = "remark", length = 500)
    private String remark;

    @Column(name = "expire_at")
    private LocalDateTime expireAt;

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    // getters / setters
}
```

### 6.5 Customer

```java
package com.bank.appointment.entity;

import jakarta.persistence.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.io.Serializable;
import java.time.LocalDateTime;

@Entity
@Table(name = "customer", uniqueConstraints = {
    @UniqueConstraint(name = "uk_customer_phone", columnNames = "phone")
})
@EntityListeners(AuditingEntityListener.class)
public class Customer implements Serializable {

    @Id
    @Column(name = "id")
    private Long id;

    @Column(name = "phone", nullable = false, length = 50)
    private String phone;

    @Column(name = "customer_name", length = 100)
    private String customerName;

    @Column(name = "id_card_enc", length = 255)
    private String idCardEnc;

    @CreatedDate
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    // getters / setters
}
```

### 6.6 NationalHoliday（复合主键）

```java
package com.bank.appointment.entity;

import jakarta.persistence.*;
import java.io.Serializable;
import java.time.LocalDate;
import java.util.Objects;

@Entity
@Table(name = "national_holiday")
@IdClass(NationalHolidayId.class)
public class NationalHoliday implements Serializable {

    @Id
    @Column(name = "bank_id")
    private Long bankId; // 0=全局默认

    @Id
    @Column(name = "holiday_date")
    private LocalDate holidayDate;

    @Column(name = "holiday_name", length = 50)
    private String holidayName;

    /** 1-法定假日 2-调休工作日 */
    @Column(name = "day_type", nullable = false)
    private Integer dayType;

    @Column(name = "holiday_year", nullable = false)
    private Integer year;

    // getters / setters
}

class NationalHolidayId implements Serializable {
    private Long bankId;
    private LocalDate holidayDate;

    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof NationalHolidayId that)) return false;
        return Objects.equals(bankId, that.bankId)
            && Objects.equals(holidayDate, that.holidayDate);
    }

    public int hashCode() {
        return Objects.hash(bankId, holidayDate);
    }
}
```

### 6.7 BranchSpecialDate（复合主键）

```java
package com.bank.appointment.entity;

import jakarta.persistence.*;
import java.io.Serializable;
import java.time.LocalDate;
import java.util.Objects;

@Entity
@Table(name = "branch_special_date")
@IdClass(BranchSpecialDateId.class)
public class BranchSpecialDate implements Serializable {

    @Id
    @Column(name = "bank_id")
    private Long bankId;

    @Id
    @Column(name = "branch_id")
    private Long branchId;

    @Id
    @Column(name = "special_date")
    private LocalDate specialDate;

    @Column(name = "is_closed")
    private Integer isClosed = 0;

    @Column(name = "cap_override")
    private Integer capOverride;

    @Column(name = "reason", length = 200)
    private String reason;

    // getters / setters
}

class BranchSpecialDateId implements Serializable {
    private Long bankId;
    private Long branchId;
    private LocalDate specialDate;

    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof BranchSpecialDateId that)) return false;
        return Objects.equals(bankId, that.bankId)
            && Objects.equals(branchId, that.branchId)
            && Objects.equals(specialDate, that.specialDate);
    }

    public int hashCode() {
        return Objects.hash(bankId, branchId, specialDate);
    }
}
```

### 6.8 AppointmentLog

```java
package com.bank.appointment.entity;

import jakarta.persistence.*;
import java.io.Serializable;
import java.time.LocalDateTime;

@Entity
@Table(name = "appointment_log", indexes = {
    @Index(name = "idx_log_appt_id", columnList = "appointment_id"),
    @Index(name = "idx_log_bank", columnList = "bank_id")
})
public class AppointmentLog implements Serializable {

    @Id
    @Column(name = "id")
    private Long id;

    @Column(name = "appointment_id", nullable = false)
    private Long appointmentId;

    @Column(name = "appointment_no", nullable = false, length = 50)
    private String appointmentNo;

    @Column(name = "bank_id", nullable = false)
    private Long bankId;

    @Column(name = "operation", nullable = false, length = 30)
    private String operation;

    @Column(name = "operator_type", nullable = false, length = 20)
    private String operatorType;

    @Column(name = "operator_id", length = 100)
    private String operatorId;

    @Column(name = "operator_name", length = 100)
    private String operatorName;

    @Column(name = "from_status")
    private Integer fromStatus;

    @Column(name = "to_status")
    private Integer toStatus;

    @Column(name = "remark", length = 500)
    private String remark;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    // getters / setters
}
```

---

## 七、Repository 层

```java
package com.bank.appointment.repository;

import com.bank.appointment.entity.*;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

public interface BankRepository extends JpaRepository<Bank, Long> {
    Optional<Bank> findByBankCode(String bankCode);
    List<Bank> findByStatus(Integer status);
}

public interface BranchRepository extends JpaRepository<Branch, Long> {
    List<Branch> findByBankIdAndStatus(Long bankId, Integer status);
    Optional<Branch> findByBankIdAndBranchCode(Long bankId, String branchCode);
}

public interface BranchCapacityRepository extends JpaRepository<BranchCapacity, Long> {

    @Query("""
        SELECT c FROM BranchCapacity c
        WHERE c.bankId = :bankId AND c.branchId = :branchId
          AND c.capacityDate = :date AND c.timeSlot = :slot
        """)
    Optional<BranchCapacity> findByBankAndBranchAndDateAndSlot(
        @Param("bankId") Long bankId,
        @Param("branchId") Long branchId,
        @Param("date") LocalDate date,
        @Param("slot") String slot);

    @Query("""
        SELECT c FROM BranchCapacity c
        WHERE c.bankId = :bankId AND c.branchId = :branchId
          AND c.capacityDate = :date AND c.usedCount < c.totalCap
        ORDER BY c.slotStart
        """)
    List<BranchCapacity> findAvailableSlots(
        @Param("bankId") Long bankId,
        @Param("branchId") Long branchId,
        @Param("date") LocalDate date);

    /** 调用 Oracle 函数判断营业日 */
    @Query(value = "SELECT fn_is_business_day(:bankId, :branchId, :date) FROM dual", nativeQuery = true)
    Integer isBusinessDay(@Param("bankId") Long bankId, @Param("branchId") Long branchId, @Param("date") LocalDate date);

    /** 乐观锁扣减：used_count + 1, version + 1 */
    @Modifying
    @Query(value = """
        UPDATE branch_capacity
        SET used_count = used_count + 1, version = version + 1, updated_at = SYSDATE
        WHERE id = :id AND version = :version AND used_count < total_cap
        """, nativeQuery = true)
    int incrementUsed(@Param("id") Long id, @Param("version") int version);

    /** 释放容量：used_count - 1 */
    @Modifying
    @Query(value = """
        UPDATE branch_capacity
        SET used_count = GREATEST(used_count - 1, 0), version = version + 1, updated_at = SYSDATE
        WHERE id = :id
        """, nativeQuery = true)
    int decrementUsed(@Param("id") Long id);

    @Modifying
    @Query("DELETE FROM BranchCapacity c WHERE c.bankId = :bankId AND c.branchId = :branchId AND c.capacityDate = :date")
    void deleteByBankIdAndBranchIdAndCapacityDate(@Param("bankId") Long bankId, @Param("branchId") Long branchId, @Param("date") LocalDate date);
}

public interface AppointmentRepository extends JpaRepository<Appointment, Long> {

    Optional<Appointment> findByAppointmentNo(String appointmentNo);

    @Query("""
        SELECT a FROM Appointment a
        WHERE a.bankId = :bankId AND a.phone = :phone
          AND a.apptDate = :date AND a.status = 1
        """)
    List<Appointment> findActiveByBankAndPhoneAndDate(
        @Param("bankId") Long bankId, @Param("phone") String phone, @Param("date") LocalDate date);

    @Query("""
        SELECT a FROM Appointment a
        WHERE a.bankId = :bankId AND a.phone = :phone
        ORDER BY a.createdAt DESC
        """)
    List<Appointment> findByBankIdAndPhone(@Param("bankId") Long bankId, @Param("phone") String phone);

    @Query("SELECT a FROM Appointment a WHERE a.status = 1 AND a.expireAt < :now")
    List<Appointment> findExpired(@Param("now") LocalDateTime now);

    @Query("""
        SELECT COUNT(a) FROM Appointment a
        WHERE a.bankId = :bankId AND a.branchId = :branchId
          AND a.apptDate = :date AND a.timeSlot = :slot AND a.status = 1
        """)
    long countActive(@Param("bankId") Long bankId, @Param("branchId") Long branchId,
                     @Param("date") LocalDate date, @Param("slot") String slot);
}

public interface AppointmentLogRepository extends JpaRepository<AppointmentLog, Long> {
    List<AppointmentLog> findByAppointmentIdOrderByCreatedAtDesc(Long appointmentId);
}

public interface CustomerRepository extends JpaRepository<Customer, Long> {
    Optional<Customer> findByPhone(String phone);
}

public interface NationalHolidayRepository extends JpaRepository<NationalHoliday, NationalHolidayId> {
}

public interface BranchSpecialDateRepository extends JpaRepository<BranchSpecialDate, BranchSpecialDateId> {
    Optional<BranchSpecialDate> findByBankIdAndBranchIdAndSpecialDate(
        Long bankId, Long branchId, LocalDate date);
}
```

---

## 八、Service 层

### 8.1 序列服务

```java
package com.bank.appointment.service;

import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class SequenceService {

    @PersistenceContext
    private EntityManager em;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public Long nextId(String seqName) {
        return ((Number) em.createNativeQuery("SELECT " + seqName + ".NEXTVAL FROM dual")
            .getSingleResult()).longValue();
    }
}
```

### 8.2 预约 Service（核心）

```java
package com.bank.appointment.service;

import com.bank.appointment.entity.*;
import com.bank.appointment.repository.*;
import org.springframework.dao.OptimisticLockingFailureException;
import org.springframework.orm.ObjectOptimisticLockingFailureException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.List;
import java.util.concurrent.ThreadLocalRandom;

@Service
@Transactional
public class AppointmentService {

    private final AppointmentRepository appointmentRepo;
    private final BranchCapacityRepository capacityRepo;
    private final AppointmentLogRepository logRepo;
    private final CustomerRepository customerRepo;
    private final SequenceService sequenceService;

    public AppointmentService(AppointmentRepository appointmentRepo,
                              BranchCapacityRepository capacityRepo,
                              AppointmentLogRepository logRepo,
                              CustomerRepository customerRepo,
                              SequenceService sequenceService) {
        this.appointmentRepo = appointmentRepo;
        this.capacityRepo = capacityRepo;
        this.logRepo = logRepo;
        this.customerRepo = customerRepo;
        this.sequenceService = sequenceService;
    }

    private static final int MAX_RETRY = 3;

    public AppointmentDTO create(CreateAppointmentReq req) {
        int retry = 0;
        while (retry < MAX_RETRY) {
            try {
                return doCreate(req);
            } catch (OptimisticLockingFailureException | ObjectOptimisticLockingFailureException e) {
                retry++;
                if (retry >= MAX_RETRY) {
                    throw new BizException("当前预约人数较多，请稍后重试");
                }
                try { Thread.sleep(30L * retry); } catch (InterruptedException ie) {}
            }
        }
        throw new BizException("系统繁忙");
    }

    @Transactional(isolation = Isolation.READ_COMMITTED)
    protected AppointmentDTO doCreate(CreateAppointmentReq req) {

        // 1. 校验营业日
        Integer isBiz = capacityRepo.isBusinessDay(req.getBankId(), req.getBranchId(), req.getApptDate());
        if (isBiz == null || isBiz == 0) {
            throw new BizException("该日期网点不营业");
        }

        // 2. 读取容量
        BranchCapacity cap = capacityRepo.findByBankAndBranchAndDateAndSlot(
            req.getBankId(), req.getBranchId(), req.getApptDate(), req.getTimeSlot())
            .orElseThrow(() -> new BizException("该时段不存在"));

        if (cap.getUsedCount() >= cap.getTotalCap()) {
            throw new BizException("该时段已约满");
        }

        // 3. 乐观锁扣减
        int affected = capacityRepo.incrementUsed(cap.getId(), cap.getVersion());
        if (affected == 0) {
            throw new OptimisticLockingFailureException("并发冲突");
        }

        // 4. 重复预约校验
        List<Appointment> existing = appointmentRepo.findActiveByBankAndPhoneAndDate(
            req.getBankId(), req.getPhone(), req.getApptDate());
        if (!existing.isEmpty()) {
            capacityRepo.decrementUsed(cap.getId());
            throw new BizException("您当天已有预约，请勿重复提交");
        }

        // 5. 插入预约
        Appointment appt = new Appointment();
        appt.setId(sequenceService.nextId("seq_appointment"));
        appt.setAppointmentNo(generateNo());
        appt.setBankId(req.getBankId());
        appt.setBranchId(req.getBranchId());
        appt.setCustomerName(req.getCustomerName());
        appt.setPhone(req.getPhone());
        appt.setIdCardEnc(AesUtil.encrypt(req.getIdCard()));
        appt.setApptDate(req.getApptDate());
        appt.setTimeSlot(req.getTimeSlot());
        appt.setSlotStart(req.getSlotStart());
        appt.setSlotEnd(req.getSlotEnd());
        appt.setServiceType(req.getServiceType());
        appt.setStatus(1);
        appt.setSource(req.getSource());
        appt.setExpireAt(LocalDateTime.of(req.getApptDate(),
            java.time.LocalTime.parse(req.getSlotEnd())));
        appointmentRepo.save(appt);

        // 6. 写日志
        AppointmentLog log = new AppointmentLog();
        log.setId(sequenceService.nextId("seq_appointment_log"));
        log.setAppointmentId(appt.getId());
        log.setAppointmentNo(appt.getAppointmentNo());
        log.setBankId(appt.getBankId());
        log.setOperation("CREATE");
        log.setOperatorType("CUSTOMER");
        log.setOperatorId(req.getPhone());
        log.setOperatorName(req.getCustomerName());
        log.setFromStatus(null);
        log.setToStatus(1);
        log.setRemark("客户提交预约");
        log.setCreatedAt(LocalDateTime.now());
        logRepo.save(log);

        // 7. 客户 upsert
        customerRepo.findByPhone(req.getPhone()).ifPresentOrElse(
            c -> {},
            () -> {
                Customer c = new Customer();
                c.setId(sequenceService.nextId("seq_customer"));
                c.setPhone(req.getPhone());
                c.setCustomerName(req.getCustomerName());
                c.setIdCardEnc(AesUtil.encrypt(req.getIdCard()));
                customerRepo.save(c);
            }
        );

        return convertToDTO(appt);
    }

    @Transactional
    public void cancel(Long bankId, Long appointmentId, String phone, String reason) {
        Appointment appt = appointmentRepo.findById(appointmentId)
            .orElseThrow(() -> new BizException("预约不存在"));
        if (!appt.getPhone().equals(phone) || !appt.getBankId().equals(bankId)) {
            throw new BizException("无权限操作");
        }
        if (appt.getStatus() != 1) {
            throw new BizException("当前状态不可取消");
        }

        appt.setStatus(3);
        appt.setCancelReason(reason);
        appointmentRepo.save(appt);

        // 释放容量
        capacityRepo.findByBankAndBranchAndDateAndSlot(
            appt.getBankId(), appt.getBranchId(), appt.getApptDate(), appt.getTimeSlot())
            .ifPresent(c -> capacityRepo.decrementUsed(c.getId()));

        // 日志
        AppointmentLog log = new AppointmentLog();
        log.setId(sequenceService.nextId("seq_appointment_log"));
        log.setAppointmentId(appt.getId());
        log.setAppointmentNo(appt.getAppointmentNo());
        log.setBankId(appt.getBankId());
        log.setOperation("CANCEL");
        log.setOperatorType("CUSTOMER");
        log.setOperatorId(phone);
        log.setOperatorName(appt.getCustomerName());
        log.setFromStatus(1);
        log.setToStatus(3);
        log.setRemark(reason);
        log.setCreatedAt(LocalDateTime.now());
        logRepo.save(log);
    }

    @Transactional
    public void markExpiredAppointments() {
        List<Appointment> expired = appointmentRepo.findExpired(LocalDateTime.now());
        for (Appointment appt : expired) {
            appt.setStatus(4);
            appointmentRepo.save(appt);

            AppointmentLog log = new AppointmentLog();
            log.setId(sequenceService.nextId("seq_appointment_log"));
            log.setAppointmentId(appt.getId());
            log.setAppointmentNo(appt.getAppointmentNo());
            log.setBankId(appt.getBankId());
            log.setOperation("EXPIRE");
            log.setOperatorType("SYSTEM");
            log.setFromStatus(1);
            log.setToStatus(4);
            log.setRemark("预约过期自动标记");
            log.setCreatedAt(LocalDateTime.now());
            logRepo.save(log);
        }
    }

    private String generateNo() {
        return "AP" + LocalDate.now().format(DateTimeFormatter.BASIC_ISO_DATE)
             + String.format("%06d", ThreadLocalRandom.current().nextInt(1_000_000));
    }

    private AppointmentDTO convertToDTO(Appointment a) { return null; /* 实现省略 */ }
}
```

### 8.3 容量生成 Service

```java
package com.bank.appointment.service;

import com.bank.appointment.entity.*;
import com.bank.appointment.repository.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDate;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;

@Service
public class CapacityService {

    private final BranchCapacityRepository capacityRepo;
    private final BranchRepository branchRepo;
    private final BranchSpecialDateRepository specialDateRepo;
    private final SequenceService sequenceService;

    public CapacityService(BranchCapacityRepository capacityRepo,
                           BranchRepository branchRepo,
                           BranchSpecialDateRepository specialDateRepo,
                           SequenceService sequenceService) {
        this.capacityRepo = capacityRepo;
        this.branchRepo = branchRepo;
        this.specialDateRepo = specialDateRepo;
        this.sequenceService = sequenceService;
    }

    @Transactional
    public void generateCapacity(Long bankId, LocalDate start, LocalDate end) {
        var branches = (bankId == null)
            ? branchRepo.findAll()
            : branchRepo.findByBankIdAndStatus(bankId, 1);

        for (Branch branch : branches) {
            LocalDate d = start;
            while (!d.isAfter(end)) {

                int isBiz = capacityRepo.isBusinessDay(branch.getBankId(), branch.getId(), d);
                if (isBiz == 1) {

                    BranchSpecialDate special = specialDateRepo
                        .findByBankIdAndBranchIdAndSpecialDate(branch.getBankId(), branch.getId(), d)
                        .orElse(null);

                    int cap = (special != null && special.getCapOverride() != null)
                        ? special.getCapOverride() : branch.getDefaultCap();

                    LocalTime slot = LocalTime.parse(branch.getBusinessStart());
                    LocalTime endT = LocalTime.parse(branch.getBusinessEnd());

                    while (slot.isBefore(endT)) {
                        LocalTime slotEnd = slot.plusMinutes(branch.getSlotMinutes());
                        if (slotEnd.isAfter(endT)) slotEnd = endT;

                        String slotStr = slot.format(DateTimeFormatter.ofPattern("HH:mm"))
                            + "-" + slotEnd.format(DateTimeFormatter.ofPattern("HH:mm"));

                        BranchCapacity existing = capacityRepo.findByBankAndBranchAndDateAndSlot(
                            branch.getBankId(), branch.getId(), d, slotStr).orElse(null);

                        if (existing == null) {
                            BranchCapacity c = new BranchCapacity();
                            c.setId(sequenceService.nextId("seq_branch_capacity"));
                            c.setBankId(branch.getBankId());
                            c.setBranchId(branch.getId());
                            c.setCapacityDate(d);
                            c.setTimeSlot(slotStr);
                            c.setSlotStart(slot.format(DateTimeFormatter.ofPattern("HH:mm:ss")));
                            c.setTotalCap(cap);
                            c.setUsedCount(0);
                            c.setVersion(0);
                            capacityRepo.save(c);
                        } else {
                            existing.setTotalCap(cap);
                            capacityRepo.save(existing);
                        }

                        slot = slotEnd;
                    }
                } else {
                    capacityRepo.deleteByBankIdAndBranchIdAndCapacityDate(
                        branch.getBankId(), branch.getId(), d);
                }

                d = d.plusDays(1);
            }
        }
    }
}
```

---

## 九、Controller

```java
package com.bank.appointment.controller;

import com.bank.appointment.dto.*;
import com.bank.appointment.service.*;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

@RestController
@RequestMapping("/api/{bankId}")
public class AppointmentController {

    private final AppointmentService appointmentService;
    private final CapacityService capacityService;

    public AppointmentController(AppointmentService appointmentService, CapacityService capacityService) {
        this.appointmentService = appointmentService;
        this.capacityService = capacityService;
    }

    @PostMapping("/appointment/create")
    public Result<AppointmentDTO> create(
            @PathVariable Long bankId,
            @RequestBody @Valid CreateAppointmentReq req) {
        req.setBankId(bankId);
        return Result.success(appointmentService.create(req));
    }

    @PostMapping("/appointment/cancel")
    public Result<Void> cancel(
            @PathVariable Long bankId,
            @RequestBody @Valid CancelAppointmentReq req) {
        appointmentService.cancel(bankId, req.getAppointmentId(), req.getPhone(), req.getReason());
        return Result.success();
    }

    @GetMapping("/appointment/detail/{appointmentNo}")
    public Result<AppointmentDTO> detail(
            @PathVariable Long bankId,
            @PathVariable String appointmentNo) {
        return Result.success(appointmentService.getDetail(bankId, appointmentNo));
    }

    @GetMapping("/appointment/my")
    public Result<List<AppointmentDTO>> myAppointments(
            @PathVariable Long bankId,
            @RequestParam String phone) {
        return Result.success(appointmentService.getMyAppointments(bankId, phone));
    }

    @GetMapping("/appointment/available-dates")
    public Result<List<AvailableDateVO>> availableDates(
            @PathVariable Long bankId,
            @RequestParam Long branchId,
            @RequestParam(required = false) String start,
            @RequestParam(required = false) String end) {

        LocalDate s = start != null ? LocalDate.parse(start) : LocalDate.now();
        LocalDate e = end != null ? LocalDate.parse(end) : s.plusDays(14);
        return Result.success(appointmentService.getAvailableDates(bankId, branchId, s, e));
    }

    @GetMapping("/appointment/available-slots")
    public Result<List<TimeSlotVO>> availableSlots(
            @PathVariable Long bankId,
            @RequestParam Long branchId,
            @RequestParam String date) {

        LocalDate d = LocalDate.parse(date);
        return Result.success(appointmentService.getAvailableSlots(bankId, branchId, d));
    }
}
```

---

## 十、定时任务

```java
package com.bank.appointment.scheduler;

import com.bank.appointment.service.*;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.time.LocalDate;

@Component
public class AppointmentScheduler {

    private final CapacityService capacityService;
    private final AppointmentService appointmentService;

    public AppointmentScheduler(CapacityService capacityService, AppointmentService appointmentService) {
        this.capacityService = capacityService;
        this.appointmentService = appointmentService;
    }

    /** 每天凌晨1点生成未来14天容量 */
    @Scheduled(cron = "0 0 1 * * ?")
    public void generateCapacity() {
        LocalDate start = LocalDate.now().plusDays(1);
        LocalDate end = start.plusDays(14);
        capacityService.generateCapacity(null, start, end);
    }

    /** 每5分钟标记过号 */
    @Scheduled(cron = "0 */5 * * * ?")
    public void markExpired() {
        appointmentService.markExpiredAppointments();
    }
}
```

---

## 十一、启动类

```java
package com.bank.appointment;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableJpaAuditing
@EnableScheduling
public class AppointmentApplication {

    public static void main(String[] args) {
        SpringApplication.run(AppointmentApplication.class, args);
    }
}
```

---

## 十二、DTO 定义

### 12.1 请求 DTO

```java
package com.bank.appointment.dto.req;

import jakarta.validation.constraints.*;
import java.time.LocalDate;

public record CreateAppointmentReq(
    @NotNull(message = "银行ID不能为空")
    Long bankId,

    @NotNull(message = "网点ID不能为空")
    Long branchId,

    @NotBlank(message = "客户姓名不能为空")
    @Size(max = 100)
    String customerName,

    @NotBlank(message = "手机号不能为空")
    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
    String phone,

    @Size(max = 18)
    String idCard,

    @NotNull(message = "预约日期不能为空")
    @FutureOrPresent(message = "预约日期不能早于今天")
    LocalDate apptDate,

    @NotBlank(message = "时段不能为空")
    String timeSlot,

    @NotBlank(message = "开始时间不能为空")
    String slotStart,

    @NotBlank(message = "结束时间不能为空")
    String slotEnd,

    @NotBlank(message = "业务类型不能为空")
    @Size(max = 50)
    String serviceType,

    @Size(max = 30)
    String source
) {}
```

```java
public record CancelAppointmentReq(
    @NotNull(message = "预约ID不能为空")
    Long appointmentId,

    @NotBlank(message = "手机号不能为空")
    String phone,

    @Size(max = 300)
    String reason
) {}
```

```java
public record CloseDayReq(
    @NotNull Long bankId,
    @NotNull Long branchId,
    @NotNull LocalDate date,
    Integer capOverride,
    String reason
) {}
```

### 12.2 响应 DTO

```java
package com.bank.appointment.dto.resp;

import java.time.LocalDate;
import java.time.LocalDateTime;

public record AppointmentDTO(
    Long id,
    String appointmentNo,
    Long bankId,
    Long branchId,
    String branchName,
    String customerName,
    String phone,
    LocalDate apptDate,
    String timeSlot,
    String slotStart,
    String slotEnd,
    String serviceType,
    String queueNo,
    Integer status,
    LocalDateTime expireAt,
    LocalDateTime createdAt
) {}
```

```java
public record AvailableDateVO(
    LocalDate date,
    Boolean available,
    Integer totalSlots,
    Integer availableSlots
) {}
```

```java
public record TimeSlotVO(
    String timeSlot,
    String slotStart,
    String slotEnd,
    Integer totalCap,
    Integer usedCount,
    Integer remaining,
    Boolean available
) {}
```

```java
public record BankVO(
    Long id,
    String bankCode,
    String bankName,
    Integer bankType,
    String logoUrl,
    String contactPhone
) {}
```

```java
public record BranchVO(
    Long id,
    Long bankId,
    String branchCode,
    String branchName,
    String address,
    String phone,
    String businessStart,
    String businessEnd,
    Integer slotMinutes,
    Integer defaultCap
) {}
```

### 12.3 统一响应体

```java
package com.bank.appointment.dto;

import com.fasterxml.jackson.annotation.JsonInclude;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record Result<T>(
    int code,
    String message,
    T data,
    Long timestamp
) {
    public static <T> Result<T> success(T data) {
        return new Result<>(0, "success", data, System.currentTimeMillis());
    }

    public static <T> Result<T> success() {
        return new Result<>(0, "success", null, System.currentTimeMillis());
    }

    public static <T> Result<T> fail(int code, String message) {
        return new Result<>(code, message, null, System.currentTimeMillis());
    }
}
```

---

## 十三、异常处理

### 13.1 业务异常

```java
package com.bank.appointment.exception;

public class BizException extends RuntimeException {
    private final int code;

    public BizException(String message) {
        super(message);
        this.code = 400;
    }

    public BizException(int code, String message) {
        super(message);
        this.code = code;
    }

    public int getCode() {
        return code;
    }
}
```

### 13.2 全局异常处理器

```java
package com.bank.appointment.exception;

import com.bank.appointment.dto.Result;
import org.springframework.dao.DuplicateKeyException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.orm.ObjectOptimisticLockingFailureException;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BizException.class)
    public ResponseEntity<Result<Void>> handleBizException(BizException e) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(Result.fail(e.getCode(), e.getMessage()));
    }

    @ExceptionHandler(ObjectOptimisticLockingFailureException.class)
    public ResponseEntity<Result<Void>> handleOptimisticLock(ObjectOptimisticLockingFailureException e) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(Result.fail(409, "系统繁忙，请重试"));
    }

    @ExceptionHandler(DuplicateKeyException.class)
    public ResponseEntity<Result<Void>> handleDuplicateKey(DuplicateKeyException e) {
        String msg = e.getMessage() != null && e.getMessage().contains("uk_bank_phone_date_active")
            ? "您当天已有预约，请勿重复提交"
            : "数据重复，请检查后重试";
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(Result.fail(409, msg));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Result<Void>> handleValidation(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
            .map(FieldError::getDefaultMessage)
            .findFirst()
            .orElse("参数校验失败");
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(Result.fail(400, msg));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<Result<Void>> handleGeneric(Exception e) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(Result.fail(500, "系统内部错误"));
    }
}
```

---

## 十四、Oracle 存储过程与函数

### 14.1 节假日判断函数

```sql
CREATE OR REPLACE FUNCTION fn_is_business_day(
    p_bank_id    IN NUMBER,
    p_branch_id  IN NUMBER,
    p_date       IN DATE
) RETURN NUMBER
IS
    v_is_closed     NUMBER(1) := 0;
    v_cap_override  NUMBER(5);
    v_day_type      NUMBER(1);
    v_weekday       NUMBER(1);
    v_branch_status NUMBER(1);
BEGIN
    SELECT NVL(status, 0) INTO v_branch_status FROM branch WHERE id = p_branch_id;
    IF v_branch_status = 0 THEN RETURN 0; END IF;

    -- 优先级1：网点特殊日期
    BEGIN
        SELECT is_closed, cap_override INTO v_is_closed, v_cap_override
        FROM branch_special_date
        WHERE bank_id = p_bank_id AND branch_id = p_branch_id AND special_date = TRUNC(p_date);
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            v_is_closed := 0; v_cap_override := NULL;
    END;

    IF v_is_closed = 1 THEN RETURN 0; END IF;
    IF v_cap_override IS NOT NULL THEN RETURN 1; END IF;

    -- 优先级2：节假日（先银行自定义，再全局）
    BEGIN
        SELECT day_type INTO v_day_type
        FROM national_holiday
        WHERE bank_id = p_bank_id AND holiday_date = TRUNC(p_date);
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            BEGIN
                SELECT day_type INTO v_day_type
                FROM national_holiday
                WHERE bank_id = 0 AND holiday_date = TRUNC(p_date);
            EXCEPTION
                WHEN NO_DATA_FOUND THEN v_day_type := NULL;
            END;
    END;

    IF v_day_type = 1 THEN RETURN 0; END IF;
    IF v_day_type = 2 THEN RETURN 1; END IF;

    -- 优先级3：默认周历（1=周日 7=周六，不营业）
    v_weekday := TO_NUMBER(TO_CHAR(p_date, 'D'));
    IF v_weekday IN (1, 7) THEN RETURN 0; END IF;

    RETURN 1;
END fn_is_business_day;
/
```

### 14.2 容量生成存储过程

```sql
CREATE OR REPLACE PROCEDURE sp_generate_capacity(
    p_bank_id     IN NUMBER DEFAULT NULL,
    p_start_date  IN DATE,
    p_end_date    IN DATE
) IS
    CURSOR cur_branch IS
        SELECT b.bank_id, b.id AS branch_id, b.business_start, b.business_end,
               b.slot_minutes, b.default_cap
        FROM branch b
        WHERE b.status = 1
          AND (p_bank_id IS NULL OR b.bank_id = p_bank_id);

    v_bank_id       NUMBER(19);
    v_branch_id     NUMBER(19);
    v_biz_start     VARCHAR2(8);
    v_biz_end       VARCHAR2(8);
    v_slot_minutes  NUMBER(3);
    v_default_cap   NUMBER(5);
    v_date          DATE;
    v_slot_start    DATE;
    v_slot_end      DATE;
    v_slot_str      VARCHAR2(20);
    v_is_biz        NUMBER(1);
    v_cap_override  NUMBER(5);
BEGIN
    OPEN cur_branch;
    LOOP
        FETCH cur_branch INTO v_bank_id, v_branch_id, v_biz_start, v_biz_end, v_slot_minutes, v_default_cap;
        EXIT WHEN cur_branch%NOTFOUND;

        v_date := TRUNC(p_start_date);
        WHILE v_date <= TRUNC(p_end_date) LOOP

            v_is_biz := fn_is_business_day(v_bank_id, v_branch_id, v_date);

            IF v_is_biz = 1 THEN
                BEGIN
                    SELECT cap_override INTO v_cap_override
                    FROM branch_special_date
                    WHERE bank_id = v_bank_id AND branch_id = v_branch_id AND special_date = v_date;
                EXCEPTION
                    WHEN NO_DATA_FOUND THEN v_cap_override := NULL;
                END;

                v_slot_start := TO_DATE(TO_CHAR(v_date,'YYYY-MM-DD')||' '||v_biz_start,'YYYY-MM-DD HH24:MI:SS');
                WHILE v_slot_start < TO_DATE(TO_CHAR(v_date,'YYYY-MM-DD')||' '||v_biz_end,'YYYY-MM-DD HH24:MI:SS') LOOP
                    v_slot_end := v_slot_start + (v_slot_minutes / 1440);
                    IF v_slot_end > TO_DATE(TO_CHAR(v_date,'YYYY-MM-DD')||' '||v_biz_end,'YYYY-MM-DD HH24:MI:SS') THEN
                        v_slot_end := TO_DATE(TO_CHAR(v_date,'YYYY-MM-DD')||' '||v_biz_end,'YYYY-MM-DD HH24:MI:SS');
                    END IF;
                    v_slot_str := TO_CHAR(v_slot_start,'HH24:MI')||'-'||TO_CHAR(v_slot_end,'HH24:MI');

                    MERGE INTO branch_capacity c
                    USING (SELECT v_bank_id AS bank_id, v_branch_id AS branch_id, v_date AS capacity_date, v_slot_str AS time_slot FROM dual) s
                    ON (c.bank_id = s.bank_id AND c.branch_id = s.branch_id AND c.capacity_date = s.capacity_date AND c.time_slot = s.time_slot)
                    WHEN MATCHED THEN
                        UPDATE SET c.total_cap = NVL(v_cap_override, v_default_cap), c.updated_at = SYSDATE
                    WHEN NOT MATCHED THEN
                        INSERT (id, bank_id, branch_id, capacity_date, time_slot, slot_start, total_cap, used_count, version)
                        VALUES (seq_branch_capacity.NEXTVAL, v_bank_id, v_branch_id, v_date, v_slot_str,
                                TO_CHAR(v_slot_start,'HH24:MI:SS'), NVL(v_cap_override, v_default_cap), 0, 0);

                    v_slot_start := v_slot_end;
                END LOOP;
            ELSE
                DELETE FROM branch_capacity
                WHERE bank_id = v_bank_id AND branch_id = v_branch_id AND capacity_date = v_date;
            END IF;

            v_date := v_date + 1;
        END LOOP;
    END LOOP;
    CLOSE cur_branch;
    COMMIT;
END sp_generate_capacity;
/
```

### 14.3 容量对账存储过程

```sql
CREATE OR REPLACE PROCEDURE sp_reconcile_capacity IS
    CURSOR cur_cap IS
        SELECT id, bank_id, branch_id, capacity_date, time_slot, used_count
        FROM branch_capacity
        WHERE capacity_date >= TRUNC(SYSDATE);

    v_actual_count NUMBER(5);
BEGIN
    FOR r IN cur_cap LOOP
        SELECT COUNT(*) INTO v_actual_count
        FROM appointment
        WHERE bank_id = r.bank_id
          AND branch_id = r.branch_id
          AND TRUNC(appt_date) = r.capacity_date
          AND time_slot = r.time_slot
          AND status IN (1, 2); -- 已预约 + 已完成

        IF v_actual_count != r.used_count THEN
            UPDATE branch_capacity
            SET used_count = v_actual_count,
                updated_at = SYSDATE
            WHERE id = r.id;
        END IF;
    END LOOP;
    COMMIT;
END sp_reconcile_capacity;
/
```

### 14.4 DBMS_SCHEDULER 定时任务

```sql
-- 每天凌晨1点生成容量
BEGIN
    DBMS_SCHEDULER.CREATE_JOB (
        job_name        => 'JOB_GENERATE_CAPACITY',
        job_type        => 'STORED_PROCEDURE',
        job_action      => 'sp_generate_capacity',
        number_of_arguments => 3,
        start_date      => SYSTIMESTAMP,
        repeat_interval => 'FREQ=DAILY; BYHOUR=1; BYMINUTE=0',
        enabled         => FALSE,
        auto_drop       => FALSE
    );

    DBMS_SCHEDULER.SET_JOB_ARGUMENT_VALUE('JOB_GENERATE_CAPACITY', 1, NULL);
    DBMS_SCHEDULER.SET_JOB_ARGUMENT_VALUE('JOB_GENERATE_CAPACITY', 2, 'SYSDATE+1');
    DBMS_SCHEDULER.SET_JOB_ARGUMENT_VALUE('JOB_GENERATE_CAPACITY', 3, 'SYSDATE+15');
    DBMS_SCHEDULER.ENABLE('JOB_GENERATE_CAPACITY');
END;
/

-- 每小时对账
BEGIN
    DBMS_SCHEDULER.CREATE_JOB (
        job_name        => 'JOB_RECONCILE_CAPACITY',
        job_type        => 'STORED_PROCEDURE',
        job_action      => 'sp_reconcile_capacity',
        start_date      => SYSTIMESTAMP,
        repeat_interval => 'FREQ=HOURLY; INTERVAL=1',
        enabled         => TRUE,
        auto_drop       => FALSE
    );
END;
/

-- 每5分钟标记过号
BEGIN
    DBMS_SCHEDULER.CREATE_JOB (
        job_name        => 'JOB_MARK_EXPIRED',
        job_type        => 'PLSQL_BLOCK',
        job_action      => 'BEGIN
            UPDATE appointment
            SET status = 4, updated_at = SYSDATE
            WHERE status = 1 AND expire_at < SYSDATE;
            COMMIT;
        END;',
        start_date      => SYSTIMESTAMP,
        repeat_interval => 'FREQ=MINUTELY; INTERVAL=5',
        enabled         => TRUE,
        auto_drop       => FALSE
    );
END;
/
```

---

## 十五、日历导出（.ics）

```java
package com.bank.appointment.util;

import com.bank.appointment.entity.Appointment;

import java.io.IOException;
import java.io.PrintWriter;
import java.time.format.DateTimeFormatter;

public class IcsUtil {

    private static final DateTimeFormatter DATE_FMT = DateTimeFormatter.ofPattern("yyyyMMdd'T'HHmmss");

    public static void writeIcs(Appointment appt, PrintWriter writer) throws IOException {
        writer.println("BEGIN:VCALENDAR");
        writer.println("VERSION:2.0");
        writer.println("PRODID:-//Bank Appointment System//CN");
        writer.println("CALSCALE:GREGORIAN");
        writer.println("METHOD:PUBLISH");
        writer.println("BEGIN:VEVENT");
        writer.println("UID:" + appt.getAppointmentNo() + "@bank-appointment.com");
        writer.println("DTSTAMP:" + java.time.LocalDateTime.now().format(DATE_FMT) + "Z");
        writer.println("DTSTART:" + appt.getApptDate().format(DateTimeFormatter.ofPattern("yyyyMMdd"))
            + "T" + appt.getSlotStart().replace(":", "") + "00");
        writer.println("DTEND:" + appt.getApptDate().format(DateTimeFormatter.ofPattern("yyyyMMdd"))
            + "T" + appt.getSlotEnd().replace(":", "") + "00");
        writer.println("SUMMARY:银行预约 - " + appt.getServiceType());
        writer.println("DESCRIPTION:预约号:" + appt.getAppointmentNo()
            + "\\n客户:" + appt.getCustomerName()
            + "\\n业务:" + appt.getServiceType()
            + "\\n时段:" + appt.getTimeSlot());
        writer.println("STATUS:CONFIRMED");
        writer.println("END:VEVENT");
        writer.println("END:VCALENDAR");
        writer.flush();
    }
}
```

Controller 中调用：

```java
@GetMapping("/appointment/download-ics/{appointmentNo}")
public void downloadIcs(@PathVariable Long bankId, @PathVariable String appointmentNo, HttpServletResponse response) throws IOException {
    Appointment appt = appointmentRepo.findByAppointmentNo(appointmentNo)
        .orElseThrow(() -> new BizException("预约不存在"));
    if (!appt.getBankId().equals(bankId)) throw new BizException("无权限");

    response.setContentType("text/calendar;charset=UTF-8");
    response.setHeader("Content-Disposition", "attachment; filename=\"appointment-" + appt.getAppointmentNo() + ".ics\"");

    try (PrintWriter writer = response.getWriter()) {
        IcsUtil.writeIcs(appt, writer);
    }
}
```

---

## 十六、加密工具

```java
package com.bank.appointment.util;

import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import javax.crypto.spec.GCMParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import java.nio.ByteBuffer;
import java.nio.charset.StandardCharsets;
import java.security.SecureRandom;
import java.util.Base64;

public class AesUtil {

    private static final String ALGORITHM = "AES/GCM/NoPadding";
    private static final int GCM_IV_LENGTH = 12;
    private static final int GCM_TAG_LENGTH = 16;
    private static final byte[] KEY = System.getenv("APPOINTMENT_AES_KEY") != null
        ? Base64.getDecoder().decode(System.getenv("APPOINTMENT_AES_KEY"))
        : new byte[16]; // 生产环境务必通过环境变量注入

    public static String encrypt(String plainText) {
        if (plainText == null) return null;
        try {
            byte[] iv = new byte[GCM_IV_LENGTH];
            new SecureRandom().nextBytes(iv);

            Cipher cipher = Cipher.getInstance(ALGORITHM);
            GCMParameterSpec spec = new GCMParameterSpec(GCM_TAG_LENGTH * 8, iv);
            cipher.init(Cipher.ENCRYPT_MODE, new SecretKeySpec(KEY, "AES"), spec);

            byte[] cipherText = cipher.doFinal(plainText.getBytes(StandardCharsets.UTF_8));
            ByteBuffer byteBuffer = ByteBuffer.allocate(iv.length + cipherText.length);
            byteBuffer.put(iv);
            byteBuffer.put(cipherText);
            return Base64.getEncoder().encodeToString(byteBuffer.array());
        } catch (Exception e) {
            throw new RuntimeException("加密失败", e);
        }
    }

    public static String decrypt(String cipherText) {
        if (cipherText == null) return null;
        try {
            byte[] decoded = Base64.getDecoder().decode(cipherText);
            ByteBuffer byteBuffer = ByteBuffer.wrap(decoded);

            byte[] iv = new byte[GCM_IV_LENGTH];
            byteBuffer.get(iv);

            byte[] cipherBytes = new byte[byteBuffer.remaining()];
            byteBuffer.get(cipherBytes);

            Cipher cipher = Cipher.getInstance(ALGORITHM);
            GCMParameterSpec spec = new GCMParameterSpec(GCM_TAG_LENGTH * 8, iv);
            cipher.init(Cipher.DECRYPT_MODE, new SecretKeySpec(KEY, "AES"), spec);

            byte[] plainText = cipher.doFinal(cipherBytes);
            return new String(plainText, StandardCharsets.UTF_8);
        } catch (Exception e) {
            throw new RuntimeException("解密失败", e);
        }
    }
}
```

---

## 十七、初始化数据

```sql
-- 个人银行
INSERT INTO bank (id, bank_code, bank_name, bank_type, status)
VALUES (seq_bank.NEXTVAL, 'RETAIL', '个人银行', 1, 1);

-- 商业银行
INSERT INTO bank (id, bank_code, bank_name, bank_type, status)
VALUES (seq_bank.NEXTVAL, 'CORP', '商业银行', 2, 1);

-- 个人银行网点
INSERT INTO branch (id, bank_id, branch_code, branch_name, address, business_start, business_end, slot_minutes, default_cap, status)
VALUES (seq_branch.NEXTVAL, 1, 'R001', '个人银行高新支行', '西安市雁塔区科技路XX号', '09:00:00', '17:00:00', 30, 5, 1);

-- 商业银行网点
INSERT INTO branch (id, bank_id, branch_code, branch_name, address, business_start, business_end, slot_minutes, default_cap, status)
VALUES (seq_branch.NEXTVAL, 2, 'C001', '商业银行高新支行', '西安市雁塔区科技路XX号', '09:00:00', '18:00:00', 60, 2, 1);

COMMIT;

-- 生成容量（调用存储过程）
BEGIN
    sp_generate_capacity(NULL, SYSDATE, SYSDATE + 14);
END;
/

-- 全局节假日示例（2026年）
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-01-01', '元旦', 1, 2026);
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-01-02', '元旦补休', 1, 2026);
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-02-17', '春节', 1, 2026);
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-02-18', '春节', 1, 2026);
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-02-19', '春节', 1, 2026);
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-04-04', '清明节', 1, 2026);
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-05-01', '劳动节', 1, 2026);
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-05-02', '劳动节补休', 1, 2026);
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-05-03', '劳动节补休', 1, 2026);
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-10-01', '国庆节', 1, 2026);
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-10-02', '国庆节', 1, 2026);
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-10-03', '国庆节', 1, 2026);
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-10-04', '国庆节', 1, 2026);
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-10-05', '国庆节', 1, 2026);
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-10-06', '国庆节', 1, 2026);
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-10-07', '国庆节', 1, 2026);

-- 调休工作日示例
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-02-08', '春节调休', 2, 2026);
INSERT INTO national_holiday (bank_id, holiday_date, holiday_name, day_type, holiday_year) VALUES (0, DATE '2026-02-15', '春节调休', 2, 2026);

COMMIT;
```

---

## 十八、项目文件结构

```
src/main/java/com/bank/appointment/
├── AppointmentApplication.java
├── config/
│   └── JpaAuditingConfig.java
├── controller/
│   ├── AppointmentController.java
│   ├── BranchController.java
│   └── BankController.java
├── dto/
│   ├── req/
│   │   ├── CreateAppointmentReq.java
│   │   ├── CancelAppointmentReq.java
│   │   └── CloseDayReq.java
│   ├── resp/
│   │   ├── AppointmentDTO.java
│   │   ├── AvailableDateVO.java
│   │   ├── TimeSlotVO.java
│   │   ├── BankVO.java
│   │   └── BranchVO.java
│   └── Result.java
├── entity/
│   ├── Bank.java
│   ├── Branch.java
│   ├── BranchCapacity.java
│   ├── Appointment.java
│   ├── AppointmentLog.java
│   ├── Customer.java
│   ├── NationalHoliday.java
│   ├── NationalHolidayId.java
│   ├── BranchSpecialDate.java
│   └── BranchSpecialDateId.java
├── exception/
│   ├── BizException.java
│   └── GlobalExceptionHandler.java
├── repository/
│   ├── BankRepository.java
│   ├── BranchRepository.java
│   ├── BranchCapacityRepository.java
│   ├── AppointmentRepository.java
│   ├── AppointmentLogRepository.java
│   ├── CustomerRepository.java
│   ├── NationalHolidayRepository.java
│   └── BranchSpecialDateRepository.java
├── scheduler/
│   └── AppointmentScheduler.java
├── service/
│   ├── AppointmentService.java
│   ├── CapacityService.java
│   └── SequenceService.java
└── util/
    ├── AesUtil.java
    └── IcsUtil.java

src/main/resources/
├── application.yml
└── data.sql

src/main/resources/db/
└── init.sql
```

---

## 十九、并发安全总结

| 机制 | 实现方式 |
|------|---------|
| 防超卖 | `@Version` 乐观锁 + native UPDATE `WHERE version = ? AND used_count < total_cap` |
| 防重复预约 | 数据库唯一索引 `uk_bank_phone_date_active` + 应用层校验 |
| 无死锁 | 全程无 `SELECT FOR UPDATE`，快照读 |
| 并发冲突重试 | `OptimisticLockingFailureException` 捕获，最多重试3次 |
| 容量对账 | 定时任务每小时核对 `used_count` vs 实际预约数 |

---

## 二十、接口清单

| # | 方法 | 路径 | 说明 |
|---|------|------|------|
| 1 | GET | `/api/{bankId}/branches` | 网点列表 |
| 2 | GET | `/api/{bankId}/branches/{branchId}` | 网点详情 |
| 3 | POST | `/api/{bankId}/appointment/create` | 提交预约 |
| 4 | POST | `/api/{bankId}/appointment/cancel` | 取消预约 |
| 5 | GET | `/api/{bankId}/appointment/detail/{no}` | 预约详情 |
| 6 | GET | `/api/{bankId}/appointment/my` | 我的预约 |
| 7 | GET | `/api/{bankId}/appointment/available-dates` | 可预约日期 |
| 8 | GET | `/api/{bankId}/appointment/available-slots` | 可用时段 |
| 9 | GET | `/api/{bankId}/appointment/download-ics/{no}` | 日历下载 |
| 10 | POST | `/api/{bankId}/admin/branch/{branchId}/close-day` | 闭店 |
| 11 | POST | `/api/{bankId}/admin/branch/{branchId}/open-day` | 恢复营业 |

---

## 二十一、总结

| 维度 | 设计决策 |
|------|---------|
| 隔离维度 | `bank_id` 贯穿所有表，无集团层 |
| 并发控制 | `@Version` 乐观锁 + native UPDATE `WHERE version = ? AND used_count < total_cap` |
| 防重复 | 数据库唯一索引 `uk_bank_phone_date_active` + 应用层校验 |
| 死锁防护 | 全程无 `SELECT FOR UPDATE`，快照读 |
| 节假日 | Oracle 函数 `fn_is_business_day`，支持银行自定义 > 全局默认 |
| 容量生成 | 存储过程 `sp_generate_capacity` + Spring Scheduler 双保险 |
| 对账 | 存储过程 `sp_reconcile_capacity` + 定时任务 |
| 加密 | AES-GCM，密钥通过环境变量注入 |
| 日历导出 | 标准 iCalendar (.ics) |
| 虚拟线程 | Spring Boot 4.1.x 默认开启 |
