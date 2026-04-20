# Full Project Description — "Дорожная карта" (Roadmap)

## 1. Project Overview

This is a **roadmap/checklist application for foreign citizens in Russia**. It guides users through the immigration process by:

1. Collecting personal/immigration data via a questionnaire (form/anketa)
2. Validating the form against configurable rules to determine if the user has overstayed their permitted entry period
3. Presenting a checklist of required steps (e.g., photos, tax registration) that must be completed before formal application
4. Tracking step completion; when all steps are done, the user is informed they can proceed with their application

---

## 2. Backend

### 2.1. Technology Stack

| Technology | Version / Details |
|---|---|
| **Language** | Java 17 |
| **Framework** | Spring Boot 3.5.5 |
| **Build Tool** | Maven |
| **Persistence** | Spring Data JPA + Hibernate |
| **Database** | H2 (in-memory, `jdbc:h2:mem:testdb`) |
| **CORS** | Configured for `http://localhost:5173` |
| **Group/Artifact** | `com.example:demo:0.0.1-SNAPSHOT` ("webinar-2") |
| **Package** | `ru.savka.demo` |

### 2.2. Dependencies (pom.xml)

- `spring-boot-starter-web` — REST API
- `spring-boot-starter-data-jpa` — JPA/Hibernate
- `h2` (runtime) — in-memory database
- `lombok` (optional)
- `spring-boot-starter-test` (test)

### 2.3. Configuration (application.properties)

```
server.port=8081
spring.h2.console.enabled=true
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=password
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.hibernate.ddl-auto=update

form.validation.rules=\
  * * * * 30,\
  by * * * 90,\
  by work no yes 180,\
  by work no family 180,\
  * * yes * 120,\
  * * family * 90
```

**Validation Rules Format:** Each rule is 5 space-separated values: `country visitPurpose relocationProgramStatus hqsStatus maxDays`. The `*` character acts as a wildcard. Rules are comma-separated.

**Current Rules:**

| Country | Purpose | Resettlement | HQS | Max Days | Meaning |
|---|---|---|---|---|---|
| `*` | `*` | `*` | `*` | 30 | Default for everyone |
| `by` | `*` | `*` | `*` | 90 | Citizens of Belarus |
| `by` | `work` | `no` | `yes` | 180 | From Belarus, HQS specialist, work purpose |
| `by` | `work` | `no` | `family` | 180 | From Belarus, HQS family member, work purpose |
| `*` | `*` | `yes` | `*` | 120 | Any country, resettlement program participant |
| `*` | `*` | `family` | `*` | 90 | Any country, resettlement program family member |

### 2.4. Architecture

**Layered architecture:** `Controller → Service → Repository` with DTOs for API data transfer.

```
ru.savka.demo/
├── Webinar2Application.java          # Spring Boot entry point
├── DataLoader.java                   # CommandLineRunner — seeds DB on startup
├── config/
│   └── CorsConfig.java               # CORS configuration
├── controller/
│   ├── UserController.java           # POST /users, POST /users/login
│   ├── CountryController.java        # GET /countries
│   ├── FormController.java           # CRUD + validation for forms
│   └── FormStepController.java       # GET/PUT form steps
├── dto/
│   ├── UserDto.java                  # username, password
│   ├── FormDto.java                  # username, entryDate, citizenshipCountryCode, visitPurpose, relocationProgramStatus, hqsStatus
│   ├── FormStepDto.java              # formId, stepName, completed
│   └── ValidationResultDto.java      # overdue (boolean), overdueDays (int)
├── entity/
│   ├── User.java                     # PK = username
│   ├── Country.java                  # PK = code
│   ├── Form.java                     # PK = id (auto-generated)
│   ├── Step.java                     # PK = stepName
│   ├── FormStep.java                 # Composite PK (FormStepId)
│   └── FormStepId.java               # Embeddable: formId + stepName
├── exception/
│   └── UserNotFoundException.java    # @ResponseStatus(404)
├── repository/
│   ├── UserRepository.java           # JpaRepository<User, String>
│   ├── CountryRepository.java        # JpaRepository<Country, String>, findAllByOrderByNameAsc()
│   ├── FormRepository.java           # JpaRepository<Form, Long>, findByUser_Username(String)
│   ├── StepRepository.java           # JpaRepository<Step, String>
│   └── FormStepRepository.java       # JpaRepository<FormStep, FormStepId>, findByForm_Id(Long)
└── service/
    ├── UserService.java              # makeNewUser, loginUser (plaintext password comparison)
    ├── CountryService.java           # getAllCountries
    ├── FormService.java              # getUserForm, saveNewForm, updateForm, getFormById
    ├── FormStepService.java          # getFormSteps, updateStepStatus
    └── ValidationService.java        # Parses rules, calculates overdue days
```

### 2.5. Entity Details

#### User (`users` table)
| Field | Type | Constraints |
|---|---|---|
| `username` | String | `@Id`, PK |
| `password` | String | — |

#### Country (`countries` table)
| Field | Type | Constraints |
|---|---|---|
| `code` | String | `@Id`, PK |
| `name` | String | — |

#### Form (`forms` table)
| Field | Type | Constraints |
|---|---|---|
| `id` | Long | `@Id`, `@GeneratedValue` |
| `user` | User | `@OneToOne`, FK → `users.username` |
| `entryDate` | LocalDate | — |
| `citizenshipCountry` | Country | `@ManyToOne`, FK → `countries.code` |
| `visitPurpose` | String | `"work"` or `"recreation"` |
| `relocationProgramStatus` | String | `"yes"`, `"no"`, `"family"` |
| `hqsStatus` | String | `"yes"`, `"no"`, `"family"` |

#### Step (`steps` table)
| Field | Type | Constraints |
|---|---|---|
| `stepName` | String | `@Id`, PK |
| `stepDescription` | String | — |

#### FormStep (`form_steps` table) — Composite PK
| Field | Type | Constraints |
|---|---|---|
| `id` | FormStepId | `@EmbeddedId` |
| `form` | Form | `@ManyToOne(LAZY)`, `@MapsId("formId")` |
| `step` | Step | `@ManyToOne(EAGER)`, `@MapsId("stepName")` |
| `completed` | int | 0 or 1 |

#### FormStepId (embeddable)
| Field | Type |
|---|---|
| `formId` | Long |
| `stepName` | String |

### 2.6. REST API Endpoints

#### UserController (`/users`)
| Method | Endpoint | Request Body | Response | Description |
|---|---|---|---|---|
| POST | `/users` | `UserDto` (username, password) | `User` | Register a new user |
| POST | `/users/login` | `UserDto` (username, password) | `User` or 404 | Login; plaintext password comparison |

#### CountryController (`/countries`)
| Method | Endpoint | Response | Description |
|---|---|---|---|
| GET | `/countries` | `List<Country>` | All countries, alphabetically ordered by name |

#### FormController (`/forms`)
| Method | Endpoint | Request/Path | Response | Description |
|---|---|---|---|---|
| GET | `/forms/{username}` | path: username | `Form` or 404 | Get user's form by username |
| POST | `/forms` | body: `FormDto` | `Form` | Create new form; auto-creates FormStep entries for all Steps (completed=0). Fails if form already exists. |
| PUT | `/forms/{formId}` | path: formId, body: `FormDto` | `Form` or 404 | Update existing form |
| GET | `/forms/{formId}/validate` | path: formId | `ValidationResultDto` | Validate form against rules; returns overdue status and days |

#### FormStepController (`/form-steps`)
| Method | Endpoint | Request/Path | Response | Description |
|---|---|---|---|---|
| GET | `/form-steps/{formId}` | path: formId | `List<FormStep>` | Get all steps for a form (includes nested Step entity) |
| PUT | `/form-steps` | body: `FormStepDto` | `FormStep` or 404 | Update step completion status |

### 2.7. Service Logic Details

#### ValidationService
- Parses `form.validation.rules` from `application.properties` using Spring `@Value` with SpEL expression
- Each rule becomes a `ValidationRule` record: `(country, purpose, resettlementStatus, hqsStatus, days)`
- **Matching algorithm:** Iterates all rules, checks if each field matches the form's values (wildcard `*` matches anything)
- **Specificity scoring:** Counts non-wildcard fields; the rule with the highest score wins
- **Overdue calculation:** `daysSinceEntry = now - entryDate`; if `daysSinceEntry > rule.days`, returns the difference; otherwise 0

#### FormService.saveNewForm
1. Looks up User by username (throws `UserNotFoundException` if not found)
2. Checks if form already exists (throws `IllegalStateException` if yes)
3. Looks up Country by code
4. Creates and saves Form
5. Fetches all Steps from DB and creates `FormStep` entries for each (completed=0)
6. Saves all FormSteps

#### DataLoader (DB seeding)
Runs on startup if tables are empty:
- **Countries:** RU (Россия), US (США), CN (Китай), BY (Беларусь), DZ (Алжир), MA (Марокко), KG (Киргизия)
- **Steps:** `photo` ("Сделать фотографии"), `tax` ("Подать заявление в налоговую")

### 2.8. Known Limitations / Current State

- **Only 2 steps** are seeded: `photo` and `tax`. The full roadmap (from `roadmap_table.md`) includes 9+ steps with dependencies and conditions.
- **No token-based authentication** — login returns the User object directly; no JWT/session tokens.
- **Plaintext passwords** — no hashing.
- **H2 in-memory DB** — data is lost on restart; no PostgreSQL yet.
- **No Swagger/OpenAPI** integration despite being listed in requirements.
- **Form validation** only checks overdue status; no field-level validation (e.g., required fields, date ranges).
- **No migration framework** (Flyway/Liquibase); DB schema is auto-generated by Hibernate (`ddl-auto=update`).

---

## 3. Frontend

### 3.1. Technology Stack

| Technology | Version |
|---|---|
| **React** | 19.2.0 |
| **TypeScript** | ~5.9.3 |
| **Vite** | 7.2.4 |
| **React Router DOM** | 6.23.1 |
| **TanStack React Query** | 5.40.0 |
| **Zustand** | 5.0.9 |
| **Bootstrap** | 5.3.3 |
| **ESLint** | 9.39.1 |

### 3.2. Build & Dev Configuration

#### vite.config.ts
```ts
export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8081',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
    },
  },
});
```
All `/api/*` requests are proxied to the backend at `localhost:8081`, with the `/api` prefix stripped.

#### TypeScript Config
- Target: ES2022
- Strict mode enabled
- `noUnusedLocals` and `noUnusedParameters` enabled
- JSX: `react-jsx`

### 3.3. Architecture

```
src/
├── main.tsx                    # Entry: BrowserRouter + QueryClientProvider + Bootstrap CSS
├── App.tsx                     # Routes definition
├── App.css                     # Empty
├── index.css                   # Empty
├── components/
│   ├── Layout.tsx              # Navbar with "Дорожная карта" brand + Outlet
│   └── FormSteps.tsx           # Checkbox list of form steps with toggle mutation
├── pages/
│   ├── HomePage.tsx            # Landing page with "Получить дорожную карту" button
│   ├── LoginPage.tsx           # Login form; auto-registers if user not found
│   ├── FormPage.tsx            # Form create/edit with country/purpose/status selects
│   └── FormDetailPage.tsx      # Validation result + FormSteps component
├── services/
│   └── apiService.ts           # All API call functions (fetch wrapper)
└── store/
    └── userStore.ts            # Zustand store: user state
```

### 3.4. Routing

| Route | Component | Description |
|---|---|---|
| `/` | `HomePage` | Landing page with CTA button |
| `/login` | `LoginPage` | User login / auto-registration |
| `/form` | `FormPage` | Create or edit questionnaire |
| `/form-details/:formId` | `FormDetailPage` | Validation results + step checklist |

All routes are nested under `Layout` (navbar + container).

### 3.5. Page Details

#### HomePage
- Single centered button: "Получить дорожную карту" → navigates to `/login`

#### LoginPage
- Form with username and password fields
- Validation: fields must not be empty
- On submit: calls `POST /api/users/login`
  - If 404 (user not found): auto-registers via `POST /api/users`
  - On success: stores user in Zustand store, navigates to `/form`
- Uses TanStack Query `useMutation`

#### FormPage
- **Guard:** Redirects to `/login` if no user in store
- Fetches countries via `GET /api/countries`
- Fetches existing form via `GET /api/forms/{username}`
- **Create mode** (no existing form):
  - Empty form fields
  - Submit → `POST /api/forms`
  - Button text: "Сохранить анкету"
- **Edit mode** (existing form):
  - Pre-fills form with existing data
  - Submit → `PUT /api/forms/{formId}`
  - Button text: "Обновить анкету"
  - Additional "Просмотр анкеты" button → navigates to `/form-details/{formId}`
- Fields:
  - **Дата въезда** — date input
  - **Страна гражданства** — select from countries (value = country code)
  - **Цель визита** — select: `work` → "Работа", `recreation` → "Отдых"
  - **Программа переселения** — select: `yes` → "Да", `no` → "Нет", `family` → "Член семьи"
  - **Высококвалифицированный специалист** — select: same options as above

#### FormDetailPage
- Fetches validation result via `GET /api/forms/{formId}/validate`
- **If overdue:** Shows warning: "Срок подачи заявления просрочен на X дней. Вам необходимо выехать из страны."
- **If not overdue:** Shows success message + renders `FormSteps` component

#### FormSteps (Component)
- Fetches steps via `GET /api/form-steps/{formId}`
- Renders each step as a checkbox with the step description
- On checkbox toggle: calls `PUT /api/form-steps` with `{formId, stepName, completed: 0|1}`
- Invalidates `formSteps` query on success (refetches)
- When all steps have `completed === 1`: shows success alert: "Все предварительные условия выполнены. Вы можете приступить к подаче заявления"

### 3.6. API Service Layer (`apiService.ts`)

All API calls go through a central `apiFetch` helper:
- Prefixes URLs with `/api`
- On 404: returns `null` (instead of throwing)
- On other errors: parses JSON error body and throws
- On success: returns parsed JSON (if content-type is `application/json`)

**Exported functions:**
| Function | Method | Endpoint | Description |
|---|---|---|---|
| `getCountries()` | GET | `/countries` | Get all countries |
| `getUserForm(username)` | GET | `/forms/{username}` | Get user's form |
| `saveUserForm(formData)` | POST | `/forms` | Create new form |
| `updateUserForm(formId, formData)` | PUT | `/forms/{formId}` | Update form |
| `validateForm(formId)` | GET | `/forms/{formId}/validate` | Validate form |
| `getFormSteps(formId)` | GET | `/form-steps/{formId}` | Get form steps |
| `updateFormStep(stepData)` | PUT | `/form-steps` | Update step status |

### 3.7. State Management

**Zustand User Store (`userStore.ts`):**
```ts
interface User { username: string }
interface UserState { user: User | null; setUser: (user: User | null) => void }
```
- Stores only the username after successful login
- No persistence (lost on page refresh)
- Used by `FormPage` for guard and by `LoginPage` to store authenticated user

**TanStack Query:**
- Used for all server-state management (countries, forms, steps, validation)
- Query keys: `['countries']`, `['form', username]`, `['formValidation', formId]`, `['formSteps', formId]`
- Mutations invalidate relevant queries on success

### 3.8. Known Limitations / Current State

- **No TypeScript types** for API responses — all `apiService.ts` functions use implicit `any` types
- **No form validation** on the frontend (e.g., required fields before submit)
- **No error boundaries** or graceful error handling beyond basic alert messages
- **No persistent auth** — Zustand store is in-memory; page refresh loses login state
- **No loading states** on some pages (e.g., FormPage doesn't show spinner while fetching countries/form)
- **Only 2 steps** displayed (matching backend seed data)
- **No token-based auth** — relies on frontend-only user state; no Authorization header sent to backend
- **No input sanitization** or XSS protection beyond React's default escaping

---

## 4. Database Schema (H2, auto-generated by Hibernate)

```
users
├── username (VARCHAR, PK)
└── password (VARCHAR)

countries
├── code (VARCHAR, PK)
└── name (VARCHAR)

forms
├── id (BIGINT, PK, AUTO_INCREMENT)
├── username (VARCHAR, FK → users.username)
├── entry_date (DATE)
├── citizenship_country_code (VARCHAR, FK → countries.code)
├── visit_purpose (VARCHAR)
├── relocation_program_status (VARCHAR)
└── hqs_status (VARCHAR)

steps
├── step_name (VARCHAR, PK)
└── step_description (TEXT)

form_steps
├── form_id (BIGINT, PK, FK → forms.id)
├── step_name (VARCHAR, PK, FK → steps.step_name)
└── completed (INT, 0 or 1)
```

---

## 5. User Flow

```
[HomePage] → Click "Получить дорожную карту"
    ↓
[LoginPage] → Enter username/password
    → If user exists: login
    → If user not found: auto-register then login
    → Store user in Zustand, navigate to /form
    ↓
[FormPage] → Fill out questionnaire (entry date, country, purpose, statuses)
    → If first time: POST /forms (creates form + steps)
    → If editing: PUT /forms/{formId}
    → After save, click "Просмотр анкеты" → navigate to /form-details/{formId}
    ↓
[FormDetailPage] → GET /forms/{formId}/validate
    → If overdue: show warning, must leave country
    → If valid: show success + step checklist
    ↓
[FormSteps] → Check/uncheck steps
    → PUT /form-steps to update completion
    → When all completed: show "You can proceed with application"
```

---

## 6. Project Files Reference

| File | Purpose |
|---|---|
| `/project-context.md` | Project structure overview and file descriptions |
| `/database.md` | Database schema documentation |
| `/description.md` | Technology stack requirements |
| `/roadmap_table.md` | Reference table of all possible roadmap steps (9+ steps) |
| `/ext/task-be.md` | Backend development prompts (tasks 1-8) |
| `/ext/task-fe.md` | Frontend development prompts (tasks 1-7) |
| `/Дорожная карта.docx` | Roadmap document (Word) |
| `/Лаба1_v8.docx` | Lab assignment document |
| `/Диаграммы.drawio` | UML/use case diagrams |

---

## 7. Gap Analysis: Current vs. Intended (from roadmap_table.md)

The `roadmap_table.md` describes a much richer set of steps:

| # | Step | Current? |
|---|---|---|
| 1 | Миграционный учет (notify owner via Gosuslugi, 7-90 days) | ❌ |
| 2 | Дактилоскопия и фотографирование (МВД/ПВС МВД, 30/90 days) | ❌ |
| 3 | Медицинское освидетельствование (30/90 days) | ❌ |
| 4 | Медицинская страховка (before employment) | ❌ |
| 5 | Патент (МВД/ПВС МВД, 30 days) | ❌ |
| 6.1 | Сертификат знания языка (exam) | ❌ |
| 6.2 | Оплата НДФЛ (monthly) | ❌ |
| 6.3 | Фотографии 3x4 | ✅ (as `photo`) |
| 6.4 | ИНН (tax office) | ❌ |
| 6.5 | Трудовой договор (2 months after patent) | ❌ |
| 6.6 | Уведомление МВД (2 months after employment) | ❌ |
| 7 | Участие в программе переселения | ❌ |
| 8 | Меры поддержки | ❌ |
| 9 | Компенсация жилья | ❌ |

Currently only **2 steps** (`photo`, `tax`) are implemented. The full roadmap requires expanding the step catalog with conditional logic based on user attributes (citizenship, purpose, statuses).
