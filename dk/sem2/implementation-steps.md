# Implementation Plan — Полная конфигурируемость приложения

## Контекст проекта

**Приложение:** "Дорожная карта" — система пошагового руководства иностранных граждан через миграционные процедуры в РФ.

**Текущий стек:**
- **Backend:** Java 17, Spring Boot 3.5.5, Maven, Spring Data JPA, H2 (in-memory), layered architecture (Controller → Service → Repository + DTO)
- **Frontend:** React 19, TypeScript, Vite 7, React Router 6, TanStack React Query 5, Zustand 5, Bootstrap 5
- **Пакет:** `ru.savka.demo`
- **Backend порт:** 8081
- **Frontend dev порт:** 5173 (проксирует `/api/*` → `http://localhost:8081/*` с удалением префикса `/api`)

**Текущая архитектура бекенда:**
```
ru.savka.demo/
├── Webinar2Application.java
├── DataLoader.java
├── config/CorsConfig.java
├── controller/   [UserController, CountryController, FormController, FormStepController]
├── dto/          [UserDto, FormDto, FormStepDto, ValidationResultDto]
├── entity/       [User, Country, Form, Step, FormStep, FormStepId]
├── exception/    [UserNotFoundException]
├── repository/   [UserRepository, CountryRepository, FormRepository, StepRepository, FormStepRepository]
└── service/      [UserService, CountryService, FormService, FormStepService, ValidationService]
```

**Текущая архитектура фронтенда:**
```
src/
├── main.tsx, App.tsx
├── components/  [Layout.tsx, FormSteps.tsx]
├── pages/       [HomePage, LoginPage, FormPage, FormDetailPage]
├── services/    [apiService.ts]
└── store/       [userStore.ts]
```

**Текущие таблицы БД:**
- `users` (username PK, password)
- `countries` (code PK, name)
- `forms` (id PK, username FK, entryDate, citizenshipCountry FK, visitPurpose, relocationProgramStatus, hqsStatus)
- `steps` (stepName PK, stepDescription)
- `form_steps` (formId FK, stepName FK, completed) — composite PK

**Что уже реализовано:**
- Аутентификация пользователя (логин/пароль, без хеширования, без токенов)
- CRUD анкеты (Form) с полями: дата въезда, страна гражданства, цель визита, статусы переселения и HQS
- Справочник шагов (Step) — сейчас 2 шага: photo, tax
- Привязка шагов к анкете (FormStep) с флагом выполнения
- Валидация сроков пребывания — правила заданы в `application.properties` в формате строки, парсятся через SpEL, матчатся по specificity scoring
- Фронтенд: страницы Home, Login, Form (create/edit), FormDetail (валидация + чеклист шагов)

**Что нужно сделать (цели задачи):**
1. Конфигурация шагов дорожной карты через БД (частично уже есть — Step, но нужно расширить)
2. Конфигурация статусов мигранта и предельных сроков через БД (сейчас в `application.properties` — нужно перенести в БД)
3. Конфигурация текстовых сообщений на фронтенде через БД (кастомизация приложения)
4. Бекенд должен предоставить полный набор эндпоинтов для загрузки конфигурации
5. Фронтенд должен работать с динамической конфигурацией
6. В дальнейшем будет разработан админ-фронтенд для управления конфигурацией (логин: admin/admin)

---

## План реализации

План разбит на 12 задач. Каждая задача оформлена как промпт, готовый к выполнению.

---

### Задача 1. Backend: Модель данных для правил валидации сроков (ValidationRule)

**Контекст:** Сейчас правила валидации сроков хранения задаются в `application.properties` в виде строки формата `"country purpose resettlementStatus hqsStatus maxDays"`, парсятся через SpEL в `ValidationService`. Необходимо перенести эту логику в базу данных.

**Что сделать:**

Создать новую сущность `ValidationRule` для хранения правил валидации сроков в БД.

**Модель `ValidationRule`** (таблица `validation_rules`):
- `id` — Long, `@Id`, `@GeneratedValue` — первичный ключ
- `countryCode` — String, nullable — код страны (из таблицы countries), `*` или null означает "любая страна"
- `visitPurpose` — String, nullable — цель визита (`work`, `recreation`), `*` или null означает "любая цель"
- `relocationProgramStatus` — String, nullable — статус программы переселения (`yes`, `no`, `family`), `*` или null означает "любой статус"
- `hqsStatus` — String, nullable — статус высококвалифицированного специалиста (`yes`, `no`, `family`), `*` или null означает "любой статус"
- `maxDays` — int — максимальное количество дней пребывания до подачи заявления

**Репозиторий:** `ValidationRuleRepository extends JpaRepository<ValidationRule, Long>` с методом `findAll()`.

**Сервис:** `ValidationRuleService` — метод `getAllRules()` возвращает все правила.

**Контроллер:** `ValidationRuleController` (`/api/validation-rules`):
- `GET /` — получить все правила валидации

**DTO:** `ValidationRuleDto` с полями: `id`, `countryCode`, `visitPurpose`, `relocationProgramStatus`, `hqsStatus`, `maxDays`.

**DataLoader:** Добавить seeding начальных правил из текущих значений `application.properties`:
1. `*, *, *, *, 30` — по умолчанию 30 дней
2. `BY, *, *, *, 90` — из Беларуси 90 дней
3. `BY, work, no, yes, 180` — из Беларуси HQS специалист по работе 180 дней
4. `BY, work, no, family, 180` — из Беларуси член семьи HQS 180 дней
5. `*, *, yes, *, 120` — программа переселения 120 дней
6. `*, *, family, *, 90` — член семьи по программе переселения 90 дней

**Важно:** Не удалять пока `ValidationService` и `application.properties` — это будет сделано в задаче 3.

---

### Задача 2. Backend: Расширение модели Step — условия применимости шагов

**Контекст:** Сейчас модель `Step` содержит только `stepName` (PK) и `stepDescription`. Для конфигурируемости необходимо, чтобы шаги могли применяться selectively в зависимости от параметров пользователя (страна, цель визита, статусы). Также нужно добавить порядковый номер для сортировки.

**Что сделать:**

Расширить модель `Step` и создать модель для условий применимости шагов.

**Изменения в `Step`:**
- Добавить поле `stepOrder` — int — порядковый номер шага для сортировки
- Добавить поле `enabled` — boolean, default true — включён ли шаг
- Добавить поле `deadlineDays` — int, nullable — срок выполнения шага в днях от въезда (если применимо)

**Новая модель `StepCondition`** (таблица `step_conditions`):
- `id` — Long, `@Id`, `@GeneratedValue`
- `step` — `@ManyToOne` → `Step` (по stepName)
- `countryCode` — String, nullable — код страны, `*` = любая
- `visitPurpose` — String, nullable — цель визита, `*` = любая
- `relocationProgramStatus` — String, nullable — статус переселения, `*` = любой
- `hqsStatus` — String, nullable — статус HQS, `*` = любой

**Логика:** Шаг применим к пользователю, если существует хотя бы одно `StepCondition` для этого шага, которое матчит параметры пользователя (или если у шага нет условий — он применим всегда).

**Репозиторий:** `StepConditionRepository extends JpaRepository<StepCondition, Long>` с методом `findByStep_StepName(String stepName)`.

**DTO:** `StepDto` (для ответа) с полями: `stepName`, `stepDescription`, `stepOrder`, `enabled`, `deadlineDays`, `conditions` (список `StepConditionDto`).

**DTO:** `StepConditionDto` с полями: `id`, `stepName`, `countryCode`, `visitPurpose`, `relocationProgramStatus`, `hqsStatus`.

**Контроллер:** Обновить `StepController` (или добавить в существующий):
- `GET /steps` — получить все шаги с условиями, отсортированные по `stepOrder`
- `GET /steps/{stepName}` — получить конкретный шаг

**DataLoader:** Обновить seed данных — добавить `stepOrder` к существующим шагам.

---

### Задача 3. Backend: Перенос логики валидации из application.properties в БД

**Контекст:** После выполнения задачи 1 правила хранятся в БД. Теперь нужно обновить `ValidationService` чтобы он читал правила из БД вместо `application.properties`, и удалить конфигурацию из `application.properties`.

**Что сделать:**

**Изменения в `ValidationService`:**
- Удалить `@Value("#{'${form.validation.rules}'.split(',')}")` и парсинг строки
- Внедрить `ValidationRuleRepository` (или `ValidationRuleService`)
- В методе `calculateOverdueDays(Form form)` загружать правила из БД через репозиторий
- Оставить логику matching и specificity scoring без изменений

**Удалить из `application.properties`:** свойство `form.validation.rules`.

**DataLoader:** Убедиться, что правила сидятся при старте если таблица пуста.

---

### Задача 4. Backend: Модель и API для текстовых сообщений (AppMessage)

**Контекст:** Фронтенд содержит захардкоженные текстовые сообщения (заголовки, кнопки, предупреждения, подсказки). Необходимо вынести все тексты в БД для возможности кастомизации приложения под любые типы дорожной карты.

**Что сделать:**

Создать сущность `AppMessage` для хранения текстовых сообщений.

**Модель `AppMessage`** (таблица `app_messages`):
- `id` — Long, `@Id`, `@GeneratedValue`
- `messageKey` — String, unique — уникальный ключ сообщения (например: `home.page.title`, `home.page.button.text`, `login.page.title`, `form.page.title`, `form.save.button`, `form.update.button`, `form.view.button`, `form.validation.overdue.message`, `form.validation.ok.message`, `form.steps.all.completed.message`, `form.steps.title`, `navbar.brand`)
- `messageText` — String, nullable — текст сообщения на русском языке
- `messageTextEn` — String, nullable — текст на английском (опционально, для будущего расширения)
- `category` — String, nullable — категория для группировки (`home`, `login`, `form`, `steps`, `navbar`, `common`)

**Репозиторий:** `AppMessageRepository extends JpaRepository<AppMessage, Long>` с методами:
- `findAll()`
- `findByCategory(String category)`
- `findByMessageKey(String key)`

**Сервис:** `AppMessageService` — методы:
- `getAllMessages()` — все сообщения
- `getMessagesByCategory(String category)`
- `getMessageByKey(String key)`

**DTO:** `AppMessageDto` с полями: `id`, `messageKey`, `messageText`, `messageTextEn`, `category`.

**Контроллер:** `AppMessageController` (`/api/messages`):
- `GET /` — получить все сообщения
- `GET /category/{category}` — получить сообщения по категории
- `GET /key/{key}` — получить сообщение по ключу

**DataLoader:** Добавить seed начальных сообщений, соответствующих текущим текстам на фронтенде:

| messageKey | messageText | category |
|---|---|---|
| `navbar.brand` | Дорожная карта | navbar |
| `home.page.button.text` | Получить дорожную карту | home |
| `login.page.title` | Вход | login |
| `login.username.label` | Имя пользователя | login |
| `login.password.label` | Пароль | login |
| `login.button.text` | Войти | login |
| `login.button.pending` | Вход... | login |
| `login.validation.error` | Имя пользователя и пароль обязательны. | login |
| `form.page.title` | Анкета | form |
| `form.entry.date.label` | Дата въезда | form |
| `form.country.label` | Страна гражданства | form |
| `form.purpose.label` | Цель визита | form |
| `form.relocation.label` | Статус: «Программа переселения» | form |
| `form.hqs.label` | Статус: «Высококвалифицированный специалист» | form |
| `form.save.button` | Сохранить анкету | form |
| `form.save.pending` | Сохранение... | form |
| `form.update.button` | Обновить анкету | form |
| `form.view.button` | Просмотр анкеты | form |
| `form.select.country.placeholder` | Выберите страну | form |
| `form.detail.page.title` | Результат проверки анкеты | form |
| `form.validation.checking` | Проверка формы... | form |
| `form.validation.error` | Ошибка при проверке формы | form |
| `form.validation.overdue.message` | Срок подачи заявления просрочен на {days} дней. Вам необходимо выехать из страны. | form |
| `form.validation.ok.message` | Условия для подачи заявления выполнены. | form |
| `form.steps.title` | Дальнейшие шаги | steps |
| `form.steps.loading` | Загрузка шагов... | steps |
| `form.steps.error` | Не удалось загрузить шаги | steps |
| `form.steps.all.completed.message` | Все предварительные условия выполнены. Вы можете приступить к подаче заявления | steps |
| `purpose.work` | Работа | common |
| `purpose.recreation` | Отдых | common |
| `status.yes` | Да | common |
| `status.no` | Нет | common |
| `status.family` | Член семьи | common |

---

### Задача 5. Backend: CRUD эндпоинты для администрирования конфигурации

**Контекст:** В дальнейшем будет разработан админ-фронтенд для управления конфигурацией. Бекенд должен предоставить полный CRUD для всех конфигурационных сущностей. Администратор будет заходить под фиксированными учётными данными: admin/admin.

**Что сделать:**

Добавить CRUD эндпоинты для управления конфигурацией. Пока без авторизации (админ-панель будет добавлена позже), но с зарезервированными путями.

**Steps CRUD** (`/api/admin/steps`):
- `GET /` — получить все шаги (дублирует публичный GET, но для админки)
- `POST /` — создать новый шаг (принимает `StepDto`: stepName, stepDescription, stepOrder, enabled, deadlineDays)
- `PUT /{stepName}` — обновить существующий шаг
- `DELETE /{stepName}` — удалить шаг (каскадно удалить связанные StepCondition)

**StepConditions CRUD** (`/api/admin/step-conditions`):
- `POST /` — создать условие для шага
- `PUT /{id}` — обновить условие
- `DELETE /{id}` — удалить условие

**ValidationRules CRUD** (`/api/admin/validation-rules`):
- `POST /` — создать правило валидации
- `PUT /{id}` — обновить правило
- `DELETE /{id}` — удалить правило

**AppMessages CRUD** (`/api/admin/messages`):
- `POST /` — создать сообщение
- `PUT /{id}` — обновить сообщение
- `DELETE /{id}` — удалить сообщение

**Countries CRUD** (`/api/admin/countries`):
- `POST /` — добавить страну
- `PUT /{code}` — обновить страну
- `DELETE /{code}` — удалить страну

**Важно:** Для всех операций добавить валидацию уникальности ключевых полей. При удалении шага — каскадное удаление связанных `StepCondition` и `FormStep`.

---

### Задача 6. Backend: Эндпоинт для получения полной конфигурации (config endpoint)

**Контекст:** Фронтенду нужно загрузить всю конфигурацию за один или несколько запросов. Для эффективности создадим единый endpoint, возвращающий полную конфигурацию.

**Что сделать:**

Создать контроллер `ConfigController` (`/api/config`) с одним эндпоинтом:

- `GET /` — возвращает полный объект конфигурации:

```json
{
  "countries": [...],
  "steps": [...],
  "validationRules": [...],
  "messages": {...}
}
```

Где:
- `countries` — список всех стран (code, name)
- `steps` — список всех включённых шагов (stepName, stepDescription, stepOrder, deadlineDays, conditions[])
- `validationRules` — список всех правил валидации
- `messages` — объект типа `{ key: text }` (map messageKey → messageText) для удобного использования на фронтенде

**DTO:** `AppConfigDto` с полями: `countries`, `steps`, `validationRules`, `messages`.

**Сервис:** `ConfigService` — собирает данные из всех соответствующих сервисов и формирует `AppConfigDto`.

---

### Задача 7. Backend: Обновить FormController — фильтрация шагов по условиям

**Контекст:** Сейчас при создании формы (`FormService.saveNewForm`) создаются FormStep записи для ВСЕХ шагов из таблицы `steps`. Необходимо изменить логику: создавать только те шаги, которые применимы к данному пользователю на основе его параметров (country, visitPurpose, relocationProgramStatus, hqsStatus).

**Что сделать:**

**Изменения в `FormService.saveNewForm`:**
1. После создания Form, загрузить все шаги из БД
2. Для каждого шага проверить его `StepCondition` записи:
   - Если у шага нет условий — шаг применим всегда
   - Если есть условия — проверить хотя бы одно условие на соответствие параметрам Form
3. Создать `FormStep` только для применимых шагов

**Метод matching:** Аналогичен логике в `ValidationService` — `*` или null в условии матчит любое значение.

**Обновить `FormController.validateForm`:**
- В `ValidationResultDto` добавить поле `applicableSteps` — список шагов, которые применимы к данной форме (для информации)
- Оставить поля `overdue` и `overdueDays` без изменений

---

### Задача 8. Frontend: Типизация API и создание типов TypeScript

**Контекст:** Сейчас фронтенд не использует TypeScript типы для API ответов — все функции в `apiService.ts` работают с `any`. Необходимо добавить типы.

**Что сделать:**

Создать файл `src/types/index.ts` с типами:

```ts
export interface User {
  username: string;
}

export interface Country {
  code: string;
  name: string;
}

export interface StepCondition {
  id?: number;
  stepName: string;
  countryCode: string | null;
  visitPurpose: string | null;
  relocationProgramStatus: string | null;
  hqsStatus: string | null;
}

export interface Step {
  stepName: string;
  stepDescription: string;
  stepOrder: number;
  enabled: boolean;
  deadlineDays: number | null;
  conditions: StepCondition[];
}

export interface FormStep {
  formId: number;
  step: Step;
  completed: number; // 0 or 1
}

export interface Form {
  id: number;
  user: { username: string };
  entryDate: string;
  citizenshipCountry: { code: string; name: string };
  visitPurpose: string;
  relocationProgramStatus: string;
  hqsStatus: string;
}

export interface ValidationResult {
  overdue: boolean;
  overdueDays: number;
}

export interface ValidationRule {
  id: number;
  countryCode: string | null;
  visitPurpose: string | null;
  relocationProgramStatus: string | null;
  hqsStatus: string | null;
  maxDays: number;
}

export interface AppMessage {
  id: number;
  messageKey: string;
  messageText: string;
  messageTextEn: string | null;
  category: string | null;
}

export interface AppConfig {
  countries: Country[];
  steps: Step[];
  validationRules: ValidationRule[];
  messages: Record<string, string>;
}
```

Обновить `apiService.ts` чтобы все функции использовали эти типы.

---

### Задача 9. Frontend: Сервис конфигурации и хук useAppConfig

**Контекст:** Фронтенд должен загружать динамическую конфигурацию с бекенда и использовать её во всех компонентах.

**Что сделать:**

**Создать `src/services/configService.ts`:**
- Функция `loadConfig()` — GET `/api/config`, возвращает `AppConfig`
- Функция `loadMessages()` — GET `/api/messages`, возвращает `AppMessage[]`
- Функция `loadSteps()` — GET `/api/steps`, возвращает `Step[]`

**Создать `src/hooks/useAppConfig.ts`:**
- Custom hook, который загружает конфигурацию через React Query
- Ключ кеша: `['appConfig']`
- Возвращает: `{ config, isLoading, error, t }` где `t` — функция локализации: `t(key: string) => string`
- Функция `t(key)` берёт текст из `config.messages[key]`, если ключ не найден — возвращает сам ключ как fallback

**Создать `src/context/AppConfigContext.tsx`:**
- React Context для предоставления конфигурации всем компонентам
- Provider оборачивает приложение в `main.tsx`

---

### Задача 10. Frontend: Обновление всех страниц для работы с динамической конфигурацией

**Контекст:** Все страницы сейчас содержат захардкоженные тексты. Необходимо заменить их на использование `t()` из `useAppConfig`.

**Что сделать:**

**Layout.tsx:**
- Заменить захардкоженный "Дорожная карта" на `t('navbar.brand')`

**HomePage.tsx:**
- Заменить текст кнопки "Получить дорожную карту" на `t('home.page.button.text')`

**LoginPage.tsx:**
- Все лейблы, заголовки, тексты кнопок, сообщения валидации — через `t()`
- Использовать ключи: `login.page.title`, `login.username.label`, `login.password.label`, `login.button.text`, `login.button.pending`, `login.validation.error`

**FormPage.tsx:**
- Все лейблы полей формы через `t()`
- Тексты кнопок: `form.save.button`, `form.update.button`, `form.view.button`, `form.save.pending`
- Опции select-ов (work/recreation, yes/no/family) через `t('purpose.work')`, `t('status.yes')` и т.д.
- Placeholder для выбора страны: `t('form.select.country.placeholder')`

**FormDetailPage.tsx:**
- Заголовок: `t('form.detail.page.title')`
- Загрузка: `t('form.validation.checking')`
- Ошибка: `t('form.validation.error')`
- Сообщение о просрочке: `t('form.validation.overdue.message')` с подстановкой `{days}` → `text.replace('{days}', String(overdueDays))`
- Сообщение об успехе: `t('form.validation.ok.message')`

**FormSteps.tsx:**
- Заголовок: `t('form.steps.title')`
- Загрузка: `t('form.steps.loading')`
- Ошибка: `t('form.steps.error')`
- Все шаги completed: `t('form.steps.all.completed.message')`
- Описания шагов брать из `step.stepDescription` (приходят с бекенда)

---

### Задача 11. Frontend: Обновление FormPage и FormDetailPage для работы с динамическими шагами

**Контекст:** Сейчас фронтенд работает с фиксированным набором шагов. После выполнения задач 2 и 7 бекенд будет возвращать только применимые шаги. Фронтенд должен корректно отображать динамический список шагов.

**Что сделать:**

**FormPage.tsx:**
- При создании/обновлении формы — больше не нужно ничего менять, шаги создаются на бекенде автоматически (задача 7)

**FormDetailPage.tsx:**
- Компонент `FormSteps` уже работает с динамическим списком шагов — без изменений
- Убедиться что шаги отображаются в правильном порядке (по `stepOrder`)
- Если у шага есть `deadlineDays` — отображать срок рядом с описанием шага (например: "Сделать фотографии (до 30 дней)")

**FormSteps.tsx:**
- Сортировать шаги по `stepOrder` перед отображением
- Для каждого шага отображать `stepDescription` + если `deadlineDays` не null: ` (до {deadlineDays} дней)`

---

### Задача 12. Frontend: Обновление userStore — сохранение пользователя в localStorage

**Контекст:** Сейчас Zustand store хранит пользователя только в памяти. При перезагрузке страницы пользователь теряет "авторизацию". Необходимо добавить персистентность.

**Что сделать:**

**Обновить `src/store/userStore.ts`:**
- Добавить persist middleware от Zustand (или вручную сохранять/загружать из `localStorage`)
- Ключ в localStorage: `roadmap_user`
- При логине — сохранять пользователя
- При логауте (если будет добавлен) — очищать

**Добавить функцию логаута:**
- `logout()` — очищает user state и localStorage
- Понадобится для админ-панели в будущем

---

## Итоговая структура после реализации

### Backend — новые/изменённые файлы:

| Файл | Действие |
|---|---|
| `entity/ValidationRule.java` | **Новый** |
| `entity/StepCondition.java` | **Новый** |
| `entity/AppMessage.java` | **Новый** |
| `dto/ValidationRuleDto.java` | **Новый** |
| `dto/StepConditionDto.java` | **Новый** |
| `dto/AppMessageDto.java` | **Новый** |
| `dto/StepDto.java` | **Новый** (расширенный) |
| `dto/AppConfigDto.java` | **Новый** |
| `repository/ValidationRuleRepository.java` | **Новый** |
| `repository/StepConditionRepository.java` | **Новый** |
| `repository/AppMessageRepository.java` | **Новый** |
| `service/ValidationRuleService.java` | **Новый** |
| `service/AppMessageService.java` | **Новый** |
| `service/ConfigService.java` | **Новый** |
| `controller/ValidationRuleController.java` | **Новый** |
| `controller/StepController.java` | **Новый** (или обновить существующий) |
| `controller/AppMessageController.java` | **Новый** |
| `controller/ConfigController.java` | **Новый** |
| `controller/AdminStepController.java` | **Новый** |
| `controller/AdminStepConditionController.java` | **Новый** |
| `controller/AdminValidationRuleController.java` | **Новый** |
| `controller/AdminAppMessageController.java` | **Новый** |
| `controller/AdminCountryController.java` | **Новый** |
| `service/ValidationService.java` | **Изменить** (читать из БД) |
| `service/FormService.java` | **Изменить** (фильтрация шагов) |
| `DataLoader.java` | **Изменить** (seed новых сущностей) |
| `application.properties` | **Изменить** (удалить form.validation.rules) |

### Frontend — новые/изменённые файлы:

| Файл | Действие |
|---|---|
| `src/types/index.ts` | **Новый** |
| `src/services/configService.ts` | **Новый** |
| `src/hooks/useAppConfig.ts` | **Новый** |
| `src/context/AppConfigContext.tsx` | **Новый** |
| `src/services/apiService.ts` | **Изменить** (типизация) |
| `src/components/Layout.tsx` | **Изменить** (динамические тексты) |
| `src/pages/HomePage.tsx` | **Изменить** |
| `src/pages/LoginPage.tsx` | **Изменить** |
| `src/pages/FormPage.tsx` | **Изменить** |
| `src/pages/FormDetailPage.tsx` | **Изменить** |
| `src/components/FormSteps.tsx` | **Изменить** (сортировка, deadlineDays) |
| `src/store/userStore.ts` | **Изменить** (localStorage persist) |
| `src/main.tsx` | **Изменить** (добавить AppConfigProvider) |

---

## Порядок выполнения

Рекомендуемый порядок выполнения задач (зависимости):

```
Задача 1 (ValidationRule entity + API)
    ↓
Задача 3 (перенос ValidationService на БД)
    ↓
Задача 2 (StepCondition + расширение Step)
    ↓
Задача 7 (фильтрация шагов в FormService)
    ↓
Задача 4 (AppMessage entity + API)
    ↓
Задача 5 (CRUD для админки)
    ↓
Задача 6 (Config endpoint)
    ↓
Задача 8 (TypeScript типы)
    ↓
Задача 9 (Config сервис + хук)
    ↓
Задача 10 (обновление страниц с t())
    ↓
Задача 11 (динамические шаги)
    ↓
Задача 12 (userStore persist)
```

Задачи 1-7 — backend. Задачи 8-12 — frontend. Задача 5 (CRUD) может выполняться параллельно с 4-6.
