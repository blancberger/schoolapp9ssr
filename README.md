# SchoolApp 9 SSR

Ένα **server-side rendered (SSR)** web application για τη **διαχείριση καθηγητών & χρηστών**, χτισμένο με Spring Boot, Thymeleaf, Spring Security, Spring Data JPA, και Flyway.

> Developed as part of the [Coding Factory](https://codingfactory.aueb.gr/) @ AUEB curriculum.

---

## 📋 Περιεχόμενα

- [Tech Stack](#-tech-stack)
- [Αρχιτεκτονική](#-αρχιτεκτονική)
- [Domain Model](#-domain-model)
- [Ασφάλεια — Roles & Capabilities](#-ασφάλεια--roles--capabilities)
- [Απαιτήσεις](#-απαιτήσεις)
- [Εγκατάσταση & Εκτέλεση](#-εγκατάσταση--εκτέλεση)
- [Environment Variables](#-environment-variables)
- [Spring Profiles](#-spring-profiles)
- [Database Migrations (Flyway)](#-database-migrations-flyway)
- [Logging](#-logging)
- [Internationalization (i18n)](#-internationalization-i18n)
- [Δομή Project](#-δομή-project)
- [Gradle Tasks](#-gradle-tasks)
- [Tests](#-tests)
- [Troubleshooting](#-troubleshooting)
- [License](#-license)

---

## 🛠 Tech Stack

| Layer | Technology |
|---|---|
| **Language** | Java 21 (Amazon Corretto — auto-provisioned via Gradle Toolchain) |
| **Framework** | Spring Boot 3.5.10 |
| **Web / Views** | Spring MVC + Thymeleaf (SSR) |
| **Security** | Spring Security 6 + `thymeleaf-extras-springsecurity6` |
| **Persistence** | Spring Data JPA / Hibernate |
| **Database** | MySQL 8+ |
| **Migrations** | Flyway |
| **Validation** | Bean Validation (Spring Boot Starter Validation) |
| **Code Gen** | Lombok |
| **Build** | Gradle (Wrapper included) |
| **Dev Tools** | Spring Boot DevTools (hot-reload) |

---

## 🏗 Αρχιτεκτονική

Η εφαρμογή ακολουθεί **layered architecture** (Controller → Service → Repository):

```
Browser ──► Thymeleaf SSR ──► Controller ──► Service ──► Repository ──► MySQL
                                  │               │
                              Validator        Mapper
                              (custom)     (DTO ↔ Entity)
```

- **Controller** — MVC controllers χειρίζονται HTTP requests και επιστρέφουν Thymeleaf views.
- **Service** — Business logic, transactional boundaries, `@PreAuthorize` security.
- **Repository** — Spring Data JPA interfaces.
- **Mapper** — Custom `@Component` για μετατροπή μεταξύ DTOs και Entities.
- **Validator** — Custom validators (π.χ. duplicate VAT check) πέρα από το Bean Validation.

### Post-Redirect-Get (PRG)

Μετά από κάθε επιτυχημένο POST (insert/update/delete), γίνεται redirect σε success page μέσω `RedirectAttributes` flash attributes, αποφεύγοντας duplicate form submissions.

---

## 📦 Domain Model

```
┌──────────────────────┐       ┌──────────────────────┐
│    AbstractEntity     │       │        Region        │
│───────────────────────│       │──────────────────────│
│ createdAt  (DATETIME)│       │ id     (BIGINT PK)   │
│ updatedAt  (DATETIME)│       │ name   (VARCHAR UK)   │
│ deleted    (BOOLEAN) │       │                      │
│ deletedAt  (DATETIME)│       │ teachers (1:N)       │
│                      │       └──────────┬───────────┘
│ softDelete()         │                  │ ManyToOne
└──────────┬───────────┘                  │
           │ extends                      │
   ┌───────┴────────┐            ┌────────┴───────────┐
   │    Teacher      │            │       User         │
   │────────────────│            │────────────────────│
   │ id   (PK)      │            │ id   (PK)          │
   │ uuid (UK)      │            │ uuid (UK)          │
   │ vat  (UK)      │            │ username (UK)      │
   │ firstname      │            │ password (BCrypt)  │
   │ lastname       │            │ role (N:1 → Role)  │
   │ region (N:1)   │            │                    │
   └────────────────┘            │ implements         │
                                 │ UserDetails        │
                                 └────────────────────┘

   ┌────────────────┐    N:N    ┌────────────────────┐
   │      Role      │◄────────►│    Capability       │
   │────────────────│           │────────────────────│
   │ id   (PK)      │           │ id   (PK)          │
   │ name (UK)      │           │ name (UK)          │
   │ users  (1:N)   │           │ description        │
   │ capabilities   │           │ roles (N:N)        │
   └────────────────┘           └────────────────────┘
```

- **AbstractEntity** — `@MappedSuperclass` που παρέχει auditing fields (`createdAt`, `updatedAt`) μέσω Spring Data JPA Auditing και **soft delete** (`deleted`, `deletedAt`).
- **Teacher** — Αναγνωρίζεται μέσω UUID (αυτό-generated via `@PrePersist`) και VAT. Ανήκει σε Region.
- **User** — Implements `UserDetails`. Ο λογαριασμός είναι ενεργός μόνο αν `deleted == false` (`isEnabled()`).
- **Role ↔ Capability** — Many-to-Many μέσω join table `roles_capabilities`.

---

## 🔐 Ασφάλεια — Roles & Capabilities

Η εφαρμογή χρησιμοποιεί **role-based** και **capability-based** authorization:

| Role | Capabilities |
|---|---|
| **ADMIN** | `INSERT_TEACHER`, `VIEW_TEACHERS`, `EDIT_TEACHER`, `DELETE_TEACHER` |
| **EMPLOYEE** | `VIEW_TEACHERS` |

### Endpoint Authorization Matrix

| Endpoint | Μέθοδος | Δικαίωμα |
|---|---|---|
| `/`, `/index.html`, `/login` | GET | Public |
| `/users/register`, `/users/success` | GET/POST | Public |
| `/teachers` | GET | `VIEW_TEACHERS` (ADMIN, EMPLOYEE) |
| `/teachers/insert` | GET/POST | `INSERT_TEACHER` (ADMIN) |
| `/teachers/edit/{uuid}` | GET/POST | `EDIT_TEACHER` (ADMIN) |
| `/teachers/delete/{uuid}` | POST | `DELETE_TEACHER` (ADMIN) |
| `/css/**`, `/img/**`, `/error` | GET | Public |

- Login: custom login page στο `/login`
- Password hashing: **BCrypt** (strength 12)
- Logout: invalidates session, deletes `JSESSIONID` cookie

---

## ✅ Απαιτήσεις

- **Java 21** — Το Gradle Toolchain μπορεί να κάνει auto-provision Amazon Corretto 21
- **MySQL 8+** — τρέχον και προσβάσιμο
- **Port 8080** — (default, αλλάζει μέσω `server.port`)

---

## 🚀 Εγκατάσταση & Εκτέλεση

### 1. Clone

```bash
git clone https://github.com/blancberger/schoolapp9ssr.git
cd schoolapp9ssr
```

### 2. MySQL Setup

Δημιουργήστε τη βάση δεδομένων (το Flyway θα τρέξει αυτόματα τα migrations):

```sql
CREATE DATABASE IF NOT EXISTS school9ssr
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;

CREATE USER IF NOT EXISTS 'user9'@'localhost' IDENTIFIED BY '12345';
GRANT ALL PRIVILEGES ON school9ssr.* TO 'user9'@'localhost';
FLUSH PRIVILEGES;
```

**Ή με Docker:**

```bash
docker run --name mysql8 \
  -e MYSQL_ROOT_PASSWORD=root \
  -e MYSQL_DATABASE=school9ssr \
  -e MYSQL_USER=user9 \
  -e MYSQL_PASSWORD=12345 \
  -p 3306:3306 -d mysql:8
```

### 3. Εκτέλεση

**Windows:**
```powershell
.\gradlew.bat bootRun
```

**macOS/Linux:**
```bash
./gradlew bootRun
```

Η εφαρμογή εκκινεί στο **http://localhost:8080/**.

### 4. Build Executable JAR

```bash
# Build
./gradlew clean bootJar

# Το artifact δημιουργείται στο: build/libs/schoolapp.jar

# Εκτέλεση με συγκεκριμένο profile
java -jar build/libs/schoolapp.jar --spring.profiles.active=prod
```

---

## ⚙ Environment Variables

| Μεταβλητή | Περιγραφή | Default (dev) |
|---|---|---|
| `SPRING_PROFILES_ACTIVE` | Ενεργό profile | `dev` |
| `MYSQL_HOST` | MySQL host | `localhost` |
| `MYSQL_PORT` | MySQL port | `3306` |
| `MYSQL_DB` | Όνομα βάσης | `school9ssr` |
| `MYSQL_USER` | Database user | `user9` |
| `MYSQL_PASSWORD` | Database password | `12345` |

---

## 🔀 Spring Profiles

| Profile | Περιγραφή |
|---|---|
| **`dev`** | Default. Flyway enabled, Thymeleaf cache disabled, `ddl-auto=validate`, `open-in-view=false` |
| **`staging`** | Placeholder για staging περιβάλλον |
| **`prod`** | Placeholder για production (προσθέστε DB credentials & tuning) |

Ενεργοποίηση profile:
```bash
# μέσω JVM argument
java -jar schoolapp.jar -Dspring.profiles.active=prod

# μέσω Spring argument
./gradlew bootRun --args="--spring.profiles.active=staging"

# μέσω environment variable
export SPRING_PROFILES_ACTIVE=prod
```

---

## 🗃 Database Migrations (Flyway)

Τα migration scripts βρίσκονται στο `src/main/resources/db/migration/` και εκτελούνται αυτόματα κατά το startup:

| Migration | Περιγραφή |
|---|---|
| `V1__initial_schema.sql` | Δημιουργία πινάκων `regions` και `teachers` |
| `V2__insert_regions.sql` | Εισαγωγή δεδομένων στο `regions` |
| `V3__alter_teachers_add_soft_delete_columns.sql` | Προσθήκη `deleted`, `deleted_at` στους teachers |
| `V4__teachers_soft_delete_indexes.sql` | Indexes για soft delete queries |
| `V5__create_users_roles_capabilities_indexes.sql` | Δημιουργία πινάκων `users`, `roles`, `capabilities`, `roles_capabilities` |
| `V6__insert_roles_capabilites.sql` | Εισαγωγή ρόλων (ADMIN, EMPLOYEE) και capabilities |

> **Σημαντικό:** Το `spring.jpa.hibernate.ddl-auto` είναι `validate` — το Hibernate **δεν** τροποποιεί το schema. Μόνο το Flyway διαχειρίζεται migrations.

---

## 📝 Logging

Η διαμόρφωση γίνεται μέσω `logback-spring.xml`. Τα logs αποθηκεύονται στο `./logs/` με **time-based rolling** (30 ημέρες retention):

| Αρχείο | Περιεχόμενο | Level |
|---|---|---|
| `logs/all.log` | Γενικά application logs | INFO+ |
| `logs/error.log` | Μόνο errors | ERROR |
| `logs/sql.log` | Hibernate SQL queries | DEBUG |
| `logs/tomcat.log` | Tomcat/Catalina logs | DEBUG |
| `logs/hikari.log` | HikariCP connection pool | INFO |

Στο console εμφανίζονται μόνο `INFO+` logs.

---

## 🌐 Internationalization (i18n)

| Αρχείο | Γλώσσα |
|---|---|
| `messages.properties` | Default (fallback) |
| `messages_el.properties` | Ελληνικά |

Τα μηνύματα καλύπτουν:
- Bean Validation errors (field-specific & generic)
- Custom validator messages (π.χ. duplicate VAT/username)
- Authentication error messages

---

## 📁 Δομή Project

```
src/main/java/gr/aueb/cf/schoolapp/
├── SchoolappApplication.java              # Entry point
├── authentication/
│   ├── SecurityConfig.java                # Security filter chain, BCrypt, rules
│   ├── CustomAuthenticationSuccessHandler.java
│   ├── CustomAuthenticationFailureHandler.java
│   └── CustomUserDetailsService.java      # Loads User entity for Spring Security
├── controller/
│   ├── LoginController.java               # GET /, GET /login
│   ├── TeacherController.java             # CRUD καθηγητών (paginated)
│   └── UserController.java                # Εγγραφή χρηστών
├── core/exceptions/
│   ├── EntityAlreadyExistsException.java
│   ├── EntityInvalidArgumentException.java
│   └── EntityNotFoundException.java
├── dto/
│   ├── TeacherInsertDTO.java              # Insert form binding
│   ├── TeacherEditDTO.java                # Edit form binding
│   ├── TeacherReadOnlyDTO.java            # Read-only projection
│   ├── UserInsertDTO.java
│   ├── UserReadOnlyDTO.java
│   ├── RegionReadOnlyDTO.java
│   └── RoleReadOnlyDTO.java
├── mapper/
│   └── Mapper.java                        # DTO ↔ Entity conversions
├── model/
│   ├── AbstractEntity.java                # Audit fields + soft delete
│   ├── Teacher.java
│   ├── User.java                          # implements UserDetails
│   ├── Role.java
│   ├── Capability.java
│   └── static_data/
│       └── Region.java
├── repository/
│   ├── TeacherRepository.java
│   ├── UserRepository.java
│   ├── RoleRepository.java
│   ├── RegionRepository.java
│   └── CapabilityRepository.java
├── service/
│   ├── ITeacherService.java               # Interface
│   ├── TeacherService.java                # Implementation + @PreAuthorize
│   ├── IUserService.java
│   ├── UserService.java
│   ├── IRegionService.java
│   ├── RegionServiceImpl.java
│   ├── IRoleService.java
│   └── RoleServiceImpl.java
└── validator/
    ├── TeacherInsertValidator.java         # Custom duplicate VAT check
    └── TeacherEditValidator.java

src/main/resources/
├── application.properties                 # Base config (driver, active profile)
├── application-dev.properties             # Dev: DB, Flyway, Thymeleaf cache off
├── application-staging.properties         # Staging placeholder
├── application-prod.properties            # Production placeholder
├── logback-spring.xml                     # Logging configuration
├── messages.properties                    # Default i18n messages
├── messages_el.properties                 # Greek messages
├── db/migration/                          # Flyway SQL scripts (V1–V6)
├── templates/
│   ├── index.html                         # Landing page
│   ├── login.html                         # Login form
│   ├── teachers.html                      # Paginated teacher list
│   ├── teacher-insert.html                # Insert teacher form
│   ├── teacher-edit.html                  # Edit teacher form
│   ├── teacher-success.html               # Insert success
│   ├── update-teacher-success.html        # Update success
│   ├── delete-teacher-success.html        # Delete success
│   ├── user-form.html                     # User registration form
│   ├── user-success.html                  # Registration success
│   ├── error.html                         # Global error page
│   └── fragments/
│       ├── header.html                    # Shared header fragment
│       └── footer.html                    # Shared footer fragment
└── static/
    ├── css/
    │   ├── main.css
    │   ├── header.css
    │   ├── footer.css
    │   ├── teachers.css
    │   └── trinsert.css
    └── img/
        └── gov_header_logo.svg
```

---

## 🔧 Gradle Tasks

| Task | Περιγραφή |
|---|---|
| `./gradlew bootRun` | Εκτέλεση εφαρμογής με hot-reload (DevTools) |
| `./gradlew test` | Εκτέλεση unit tests (JUnit Platform) |
| `./gradlew build` | Build + tests |
| `./gradlew clean` | Καθαρισμός build outputs |
| `./gradlew bootJar` | Build executable JAR → `build/libs/schoolapp.jar` |

> **Σημείωση (Windows):** Αντικαταστήστε `./gradlew` με `.\gradlew.bat`

---

## 🧪 Tests

- **Location:** `src/test/java/gr/aueb/cf/schoolapp/`
- **Framework:** JUnit 5 (JUnit Platform)
- **Run:** `./gradlew test`

---

## 🐛 Troubleshooting

| Πρόβλημα | Λύση |
|---|---|
| `Communications link failure` | Βεβαιωθείτε ότι η MySQL τρέχει και τα credentials είναι σωστά |
| `Schema validation failed` | Η βάση δεν είναι συγχρονισμένη με τα entities — ελέγξτε ότι το Flyway έτρεξε σωστά (`spring.flyway.enabled=true`) |
| Port already in use | Αλλάξτε port μέσω `server.port` στο properties ή `--server.port=8081` |
| `Access Denied` στα endpoints | Ελέγξτε τον ρόλο του χρήστη — μόνο ADMIN έχει πλήρη πρόσβαση |
| `LazyInitializationException` | Το `open-in-view=false` απαιτεί `JOIN FETCH` στα queries — ελέγξτε τα repository methods |

---

## 📄 License

Δεν περιλαμβάνεται αρχείο license στο αποθετήριο. Όλα τα δικαιώματα ανήκουν στον δημιουργό εκτός αν καθοριστεί διαφορετικά.
