# Implementation Plan — Admin Frontend Application

## Контекст проекта

**Приложение:** "Дорожная карта" — система пошагового руководства иностранных граждан через миграционные процедуры в РФ.

**Текущий стек (мигрант-фронтенд):**
- **Backend:** Java 17, Spring Boot 3.5.5, Maven, Spring Data JPA, H2 (in-memory), layered architecture (Controller → Service → Repository + DTO), пакет `ru.savka.demo`, порт 8081
- **Frontend (мигрант):** React 19, TypeScript, Vite 7, React Router 6, TanStack React Query 5, Zustand 5, Bootstrap 5, порт 5173

**Новое приложение:** Admin-панель для управления конфигурацией системы. Расположение: `/admin/`. Будет работать на порту 5174 (или другом, чтобы не конфликтовать с мигрант-фронтендом).

**Администратор:** фиксированные учётные данные — логин: `admin`, пароль: `admin`.

**Предполагается,** что задачи 1-7 из `implementation-steps.md` (backend: новые сущности ValidationRule, StepCondition, AppMessage, расширение Step, CRUD эндпоинты, Config endpoint) уже выполнены или выполняются параллельно. Данный план включает задачи по доработке бекенда, необходимые именно для админ-панели.

---

## Текущие таблицы БД (после выполнения задач 1-4 из main plan):

- `users` (username PK, password)
- `countries` (code PK, name)
- `forms` (id PK, username FK, entryDate, citizenshipCountry FK, visitPurpose, relocationProgramStatus, hqsStatus)
- `steps` (stepName PK, stepDescription, stepOrder, enabled, deadlineDays)
- `form_steps` (formId FK, stepName FK, completed)
- `validation_rules` (id PK, countryCode, visitPurpose, relocationProgramStatus, hqsStatus, maxDays)
- `step_conditions` (id PK, stepName FK, countryCode, visitPurpose, relocationProgramStatus, hqsStatus)
- `app_messages` (id PK, messageKey unique, messageText, messageTextEn, category)

---

## План реализации

План разбит на 10 задач. Задачи A1-A3 — доработки бекенда. Задачи F1-F7 — фронтенд админ-панели.

---

### Задача A1. Backend: Эндпоинт аутентификации администратора

**Контекст:** Администратор входит под фиксированными учётными данными (admin/admin). Нужен отдельный эндпоинт для админ-логина. В будущем можно заменить на полноценную систему ролей, но сейчас — простая проверка.

**Что сделать:**

**Создать `AdminAuthController`** (`/api/admin/auth`):
- `POST /login` — принимает `{ username: string, password: string }`, возвращает `{ username: string, role: string, token: string }` или 401

**Логика:**
- Если `username === "admin"` и `password === "admin"` — вернуть объект `{ username: "admin", role: "ADMIN", token: "admin-token-static" }`
- Иначе — вернуть HTTP 401 Unauthorized с телом `{ message: "Invalid admin credentials" }`

**DTO:**
- `AdminLoginRequest` — `username`, `password`
- `AdminLoginResponse` — `username`, `role`, `token`

**CORS:** Обновить `CorsConfig` чтобы разрешить запросы с порта админ-фронтенда (например `http://localhost:5174`).

**Важно:** Это временное решение. В будущем нужно добавить:
- Хеширование паролей (BCrypt)
- JWT-токены с expiration
- Таблицу admin_users в БД

---

### Задача A2. Backend: CRUD эндпоинты для администрирования (полный набор)

**Контекст:** Админ-панели нужен полный CRUD для управления всеми конфигурационными сущностями. Эндпоинты должны быть под префиксом `/api/admin/`. Предполагается, что сущности (ValidationRule, StepCondition, AppMessage, расширенный Step) уже созданы по задачам 1-4 из main plan.

**Что сделать:**

Создать контроллеры для CRUD операций. Все эндпоинты под префиксом `/api/admin/`.

#### AdminStepController (`/api/admin/steps`)

| Method | Endpoint | Request Body | Response | Description |
|---|---|---|---|---|
| GET | `/` | — | `List<StepDto>` | Все шаги, отсортированные по stepOrder |
| GET | `/{stepName}` | — | `StepDto` | Конкретный шаг с условиями |
| POST | `/` | `StepCreateDto` | `StepDto` (201) | Создать новый шаг |
| PUT | `/{stepName}` | `StepUpdateDto` | `StepDto` | Обновить шаг |
| DELETE | `/{stepName}` | — | 204 | Удалить шаг + каскадно StepCondition |

**StepCreateDto:** `stepName`, `stepDescription`, `stepOrder`, `enabled`, `deadlineDays`
**StepUpdateDto:** `stepDescription`, `stepOrder`, `enabled`, `deadlineDays` (stepName не меняется)

**Важно:** При создании шага автоматически создавать записи `FormStep` для всех существующих форм (если шаг enabled). Или оставить это на ответственность администратора — не создавать для существующих форм, только для новых.

#### AdminStepConditionController (`/api/admin/steps/{stepName}/conditions`)

| Method | Endpoint | Request Body | Response | Description |
|---|---|---|---|---|
| GET | `/` | — | `List<StepConditionDto>` | Все условия для шага |
| POST | `/` | `StepConditionDto` | `StepConditionDto` (201) | Создать условие |
| PUT | `/{id}` | `StepConditionDto` | `StepConditionDto` | Обновить условие |
| DELETE | `/{id}` | — | 204 | Удалить условие |

**StepConditionDto:** `id`, `countryCode`, `visitPurpose`, `relocationProgramStatus`, `hqsStatus`

#### AdminValidationRuleController (`/api/admin/validation-rules`)

| Method | Endpoint | Request Body | Response | Description |
|---|---|---|---|---|
| GET | `/` | — | `List<ValidationRuleDto>` | Все правила |
| POST | `/` | `ValidationRuleDto` | `ValidationRuleDto` (201) | Создать правило |
| PUT | `/{id}` | `ValidationRuleDto` | `ValidationRuleDto` | Обновить правило |
| DELETE | `/{id}` | — | 204 | Удалить правило |

**ValidationRuleDto:** `id`, `countryCode`, `visitPurpose`, `relocationProgramStatus`, `hqsStatus`, `maxDays`

#### AdminAppMessageController (`/api/admin/messages`)

| Method | Endpoint | Request Body | Response | Description |
|---|---|---|---|---|
| GET | `/` | — | `List<AppMessageDto>` | Все сообщения |
| GET | `/category/{category}` | — | `List<AppMessageDto>` | Сообщения по категории |
| POST | `/` | `AppMessageDto` | `AppMessageDto` (201) | Создать сообщение |
| PUT | `/{id}` | `AppMessageDto` | `AppMessageDto` | Обновить сообщение |
| DELETE | `/{id}` | — | 204 | Удалить сообщение |

**AppMessageDto:** `id`, `messageKey`, `messageText`, `messageTextEn`, `category`

#### AdminCountryController (`/api/admin/countries`)

| Method | Endpoint | Request Body | Response | Description |
|---|---|---|---|---|
| GET | `/` | — | `List<Country>` | Все страны |
| POST | `/` | `{ code, name }` | `Country` (201) | Добавить страну |
| PUT | `/{code}` | `{ name }` | `Country` | Обновить страну |
| DELETE | `/{code}` | — | 204 | Удалить страну |

**Важно:** При удалении страны — проверить, не используется ли она в `forms` или `validation_rules` / `step_conditions`. Если используется — вернуть 409 Conflict.

---

### Задача A3. Backend: Промежуточный фильтр (AdminAuthInterceptor)

**Контекст:** Все эндпоинты `/api/admin/*` должны быть защищены. Сейчас используется простая проверка admin/admin, но нужно чтобы каждый запрос к админ-эндпоинтам проверял авторизацию.

**Что сделать:**

**Создать `AdminAuthInterceptor`** (или `AdminAuthFilter`):
- Реализовать `HandlerInterceptor` (Spring MVC)
- В `preHandle()`: проверять заголовок `Authorization` или cookie/session
- Для текущей упрощённой схемы: проверять что в request body/login был успешный админ-логин и токен сохранён в session/header
- Простейший вариант: проверять заголовок `X-Admin-Token: admin-token-static`

**Регистрация:**
- Зарегистрировать interceptor только для путей `/api/admin/**`
- Исключить `/api/admin/auth/login` из проверки

**Альтернатива (проще для начала):** Пока не реализовывать interceptor. Админ-эндпоинты будут открыты, а защита будет добавлена позже когда будет полноценная JWT-аутентификация. Админ-фронтенд просто хранит токен и отправляет его с каждым запросом.

**Рекомендация:** Начать без interceptor (открытые эндпоинты), добавить защиту в отдельной задаче после реализации JWT.

---

### Задача F1. Admin Frontend: Инициализация проекта

**Контекст:** Необходимо создать новое React-приложение в папке `/admin/` с тем же стеком, что и основной фронтенд.

**Что сделать:**

**Создать проект:**
```bash
cd /home/andy/Documents/learn/s6/PP/final
npm create vite@latest admin -- --template react-ts
cd admin
npm install
```

**Установить зависимости:**
```bash
npm install react-router-dom @tanstack/react-query zustand bootstrap
```

**Настроить `vite.config.ts`:**
```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5174,
    proxy: {
      '/api': {
        target: 'http://localhost:8081',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
    },
  },
})
```

**Настроить `tsconfig.json`:**
- Target: ES2022
- Strict mode
- `noUnusedLocals`, `noUnusedParameters` включены
- JSX: `react-jsx`

**Настроить `src/main.tsx`:**
- Подключить Bootstrap CSS: `import 'bootstrap/dist/css/bootstrap.min.css'`
- Обернуть в `BrowserRouter` и `QueryClientProvider`

**Настроить `index.html`:**
- Title: "Дорожная карта — Администрирование"

**Настроить ESLint** аналогично основному фронтенду.

---

### Задача F2. Admin Frontend: Типы и API сервис

**Контекст:** Админ-панели нужны TypeScript типы для всех сущностей и сервис для работы с API.

**Что сделать:**

**Создать `src/types/index.ts`:**

```ts
export interface AdminCredentials {
  username: string;
  password: string;
}

export interface AdminAuthResponse {
  username: string;
  role: string;
  token: string;
}

export interface Country {
  code: string;
  name: string;
}

export interface StepCondition {
  id?: number;
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

export interface StepCreateDto {
  stepName: string;
  stepDescription: string;
  stepOrder: number;
  enabled: boolean;
  deadlineDays: number | null;
}

export interface StepUpdateDto {
  stepDescription: string;
  stepOrder: number;
  enabled: boolean;
  deadlineDays: number | null;
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

**Создать `src/services/api.ts`:**

```ts
const API_BASE = '/api';

const apiFetch = async <T>(url: string, options?: RequestInit): Promise<T | null> => {
  const response = await fetch(`${API_BASE}${url}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
  });

  if (!response.ok) {
    const error = await response.json().catch(() => ({ message: 'Unknown error' }));
    throw new Error(error.message || `HTTP ${response.status}`);
  }

  if (response.status === 204) return null;

  const contentType = response.headers.get('content-type');
  if (contentType?.includes('application/json')) {
    return response.json();
  }
  return null;
};

// Auth
export const adminLogin = (credentials: AdminCredentials) =>
  apiFetch<AdminAuthResponse>('/admin/auth/login', {
    method: 'POST',
    body: JSON.stringify(credentials),
  });

// Steps
export const getSteps = () => apiFetch<Step[]>('/admin/steps');
export const getStep = (stepName: string) => apiFetch<Step>(`/admin/steps/${stepName}`);
export const createStep = (data: StepCreateDto) =>
  apiFetch<Step>('/admin/steps', { method: 'POST', body: JSON.stringify(data) });
export const updateStep = (stepName: string, data: StepUpdateDto) =>
  apiFetch<Step>(`/admin/steps/${stepName}`, { method: 'PUT', body: JSON.stringify(data) });
export const deleteStep = (stepName: string) =>
  apiFetch(`/admin/steps/${stepName}`, { method: 'DELETE' });

// Step Conditions
export const getStepConditions = (stepName: string) =>
  apiFetch<StepCondition[]>(`/admin/steps/${stepName}/conditions`);
export const createStepCondition = (stepName: string, data: StepCondition) =>
  apiFetch<StepCondition>(`/admin/steps/${stepName}/conditions`, {
    method: 'POST',
    body: JSON.stringify(data),
  });
export const updateStepCondition = (stepName: string, id: number, data: StepCondition) =>
  apiFetch<StepCondition>(`/admin/steps/${stepName}/conditions/${id}`, {
    method: 'PUT',
    body: JSON.stringify(data),
  });
export const deleteStepCondition = (stepName: string, id: number) =>
  apiFetch(`/admin/steps/${stepName}/conditions/${id}`, { method: 'DELETE' });

// Validation Rules
export const getValidationRules = () => apiFetch<ValidationRule[]>('/admin/validation-rules');
export const createValidationRule = (data: ValidationRule) =>
  apiFetch<ValidationRule>('/admin/validation-rules', { method: 'POST', body: JSON.stringify(data) });
export const updateValidationRule = (id: number, data: ValidationRule) =>
  apiFetch<ValidationRule>(`/admin/validation-rules/${id}`, { method: 'PUT', body: JSON.stringify(data) });
export const deleteValidationRule = (id: number) =>
  apiFetch(`/admin/validation-rules/${id}`, { method: 'DELETE' });

// App Messages
export const getAppMessages = () => apiFetch<AppMessage[]>('/admin/messages');
export const getAppMessagesByCategory = (category: string) =>
  apiFetch<AppMessage[]>(`/admin/messages/category/${category}`);
export const createAppMessage = (data: AppMessage) =>
  apiFetch<AppMessage>('/admin/messages', { method: 'POST', body: JSON.stringify(data) });
export const updateAppMessage = (id: number, data: AppMessage) =>
  apiFetch<AppMessage>(`/admin/messages/${id}`, { method: 'PUT', body: JSON.stringify(data) });
export const deleteAppMessage = (id: number) =>
  apiFetch(`/admin/messages/${id}`, { method: 'DELETE' });

// Countries
export const getCountries = () => apiFetch<Country[]>('/admin/countries');
export const createCountry = (data: { code: string; name: string }) =>
  apiFetch<Country>('/admin/countries', { method: 'POST', body: JSON.stringify(data) });
export const updateCountry = (code: string, data: { name: string }) =>
  apiFetch<Country>(`/admin/countries/${code}`, { method: 'PUT', body: JSON.stringify(data) });
export const deleteCountry = (code: string) =>
  apiFetch(`/admin/countries/${code}`, { method: 'DELETE' });
```

---

### Задача F3. Admin Frontend: Store авторизации и Layout

**Контекст:** Нужен Zustand store для хранения состояния администратора и общий Layout с навигацией.

**Что сделать:**

**Создать `src/store/adminStore.ts`:**

```ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface AdminState {
  isAuthenticated: boolean;
  token: string | null;
  login: (token: string) => void;
  logout: () => void;
}

const useAdminStore = create<AdminState>()(
  persist(
    (set) => ({
      isAuthenticated: false,
      token: null,
      login: (token) => set({ isAuthenticated: true, token }),
      logout: () => set({ isAuthenticated: false, token: null }),
    }),
    { name: 'admin-auth' }
  )
);

export default useAdminStore;
```

**Создать `src/components/AdminLayout.tsx`:**

- Верхняя навигация (navbar) с:
  - Брендом: "Дорожная карта — Администрирование"
  - Навигационными ссылками: Шаги, Правила валидации, Сообщения, Страны
  - Кнопкой "Выход" (справа)
- Основной контент через `<Outlet />`
- Если пользователь не авторизован — редирект на `/login`

**Создать `src/components/ProtectedRoute.tsx`:**

```tsx
import { Navigate, Outlet } from 'react-router-dom';
import useAdminStore from '../store/adminStore';

const ProtectedRoute = () => {
  const isAuthenticated = useAdminStore((s) => s.isAuthenticated);
  return isAuthenticated ? <Outlet /> : <Navigate to="/login" replace />;
};

export default ProtectedRoute;
```

---

### Задача F4. Admin Frontend: Страница логина

**Контекст:** Администратор входит с фиксированными учётными данными (admin/admin).

**Что сделать:**

**Создать `src/pages/AdminLoginPage.tsx`:**

- Форма с полями: username, password
- Валидация: поля не пустые
- При submit: вызов `adminLogin({ username, password })` через `useMutation`
- При успехе: сохранить токен в Zustand store, редирект на `/dashboard`
- При ошибке: показать alert "Неверные учётные данные"
- Стилизация: Bootstrap, центрированная карточка, аналогично LoginPage мигрант-фронтенда

---

### Задача F5. Admin Frontend: Dashboard (главная страница)

**Контекст:** После входа администратор попадает на dashboard — обзор текущей конфигурации.

**Что сделать:**

**Создать `src/pages/DashboardPage.tsx`:**

- Заголовок: "Панель администрирования"
- Карточки-сводки (Bootstrap cards):
  - **Шаги:** количество шагов (запрос `GET /admin/steps`), кнопка "Управление" → `/steps`
  - **Правила валидации:** количество правил (запрос `GET /admin/validation-rules`), кнопка "Управление" → `/validation-rules`
  - **Сообщения:** количество сообщений (запрос `GET /admin/messages`), кнопка "Управление" → `/messages`
  - **Страны:** количество стран (запрос `GET /admin/countries`), кнопка "Управление" → `/countries`
- Использовать React Query для загрузки счётчиков
- Стилизация: сетка Bootstrap (row + col)

---

### Задача F6. Admin Frontend: Управление шагами (Steps Management)

**Контекст:** Самая сложная страница админки. Нужно управлять шагами дорожной карты и их условиями применимости.

**Что сделать:**

**Создать `src/pages/StepsPage.tsx`:**

- Заголовок: "Управление шагами"
- Таблица (Bootstrap table) со столбцами:
  - Порядок (stepOrder)
  - Название (stepName)
  - Описание (stepDescription)
  - Срок (deadlineDays)
  - Включён (enabled — badge/checkbox)
  - Условия (количество условий)
  - Действия: редактировать, удалить
- Кнопка "Добавить шаг" → открывает модальное окно или форму

**Создать `src/pages/StepFormPage.tsx` (создание/редактирование шага):**

- Если `stepName` в URL — режим редактирования, иначе — создания
- Поля формы:
  - **Название** (stepName) — text input (только при создании, disabled при редактировании)
  - **Описание** (stepDescription) — textarea
  - **Порядок** (stepOrder) — number input
  - **Срок (дней)** (deadlineDays) — number input, nullable
  - **Включён** (enabled) — checkbox
- Секция **"Условия применимости"**:
  - Таблица существующих условий для этого шага
  - Столбцы: Страна, Цель визита, Переселение, HQS, действия (удалить)
  - Кнопка "Добавить условие" → inline-форма или модальное окно
  - Каждое условие — 4 select-поля:
    - Страна: список стран + опция "Любая" (`*`)
    - Цель визита: `work`, `recreation`, `*`
    - Переселение: `yes`, `no`, `family`, `*`
    - HQS: `yes`, `no`, `family`, `*`
- Кнопки: "Сохранить", "Отмена" (назад к списку)

**Создать `src/components/StepConditionForm.tsx`:**

- Компонент для создания/редактирования одного условия
- 4 select-поля с опцией `*` = "Любое значение"
- Кнопка "Добавить" / "Сохранить"

**Логика удаления шага:**
- При нажатии "Удалить" — подтверждение (confirm dialog): "Удалить шаг и все связанные условия?"
- Вызов `DELETE /admin/steps/{stepName}`
- Инвалидация кеша React Query

---

### Задача F7. Admin Frontend: Управление правилами валидации (Validation Rules)

**Контекст:** Страница для CRUD правил, определяющих максимальный срок пребывания в зависимости от параметров пользователя.

**Что сделать:**

**Создать `src/pages/ValidationRulesPage.tsx`:**

- Заголовок: "Правила валидации сроков"
- Таблица со столбцами:
  - Страна (countryCode — отображать название страны или `*`)
  - Цель визита (visitPurpose — отображать текст или `*`)
  - Переселение (relocationProgramStatus — текст или `*`)
  - HQS (hqsStatus — текст или `*`)
  - Макс. дней (maxDays)
  - Specificity (вычисляемое: количество не-`*` полей)
  - Действия: редактировать, удалить
- Кнопка "Добавить правило"

**Создать `src/pages/ValidationRuleFormPage.tsx`:**

- Поля формы:
  - **Страна** — select: список стран + "Любая" (`*`)
  - **Цель визита** — select: `work` → "Работа", `recreation` → "Отдых", `*` → "Любая"
  - **Программа переселения** — select: `yes` → "Да", `no` → "Нет", `family` → "Член семьи", `*` → "Любой"
  - **HQS** — select: `yes` → "Да", `no` → "Нет", `family` → "Член семьи", `*` → "Любой"
  - **Макс. дней** — number input
- Кнопки: "Сохранить", "Отмена"
- Валидация: maxDays > 0

**Логика:**
- При создании/обновлении — POST/PUT к `/admin/validation-rules`
- После сохранения — редирект на список
- При удалении — confirm dialog

---

### Задача F8. Admin Frontend: Управление сообщениями (App Messages)

**Контекст:** Страница для CRUD текстовых сообщений. Сообщений много (~35+), поэтому нужна группировка по категориям и удобный поиск.

**Что сделать:**

**Создать `src/pages/MessagesPage.tsx`:**

- Заголовок: "Управление сообщениями"
- Фильтр по категории — select: Все, navbar, home, login, form, steps, common
- Поиск по ключу (messageKey) — text input с фильтрацией на клиенте
- Таблица со столбцами:
  - Ключ (messageKey)
  - Категория (category)
  - Текст (messageText) — обрезанный до 80 символов с `...`
  - Действия: редактировать, удалить
- Кнопка "Добавить сообщение"

**Создать `src/pages/MessageFormPage.tsx`:**

- Поля формы:
  - **Ключ** (messageKey) — text input (только при создании, disabled при редактировании)
  - **Категория** (category) — select: navbar, home, login, form, steps, common или text input для произвольной
  - **Текст (RU)** (messageText) — textarea
  - **Текст (EN)** (messageTextEn) — textarea, nullable
- Кнопки: "Сохранить", "Отмена"

**Создать `src/components/MessagePreview.tsx`:**

- Компонент для предпросмотра полного текста сообщения в таблице (popover или expandable row)

---

### Задача F9. Admin Frontend: Управление странами (Countries)

**Контекст:** Страница для CRUD справочника стран. Самая простая страница.

**Что сделать:**

**Создать `src/pages/CountriesPage.tsx`:**

- Заголовок: "Управление странами"
- Таблица со столбцами:
  - Код (code)
  - Название (name)
  - Действия: редактировать, удалить
- Кнопка "Добавить страну"

**Создать `src/pages/CountryFormPage.tsx`:**

- Поля формы:
  - **Код** (code) — text input, uppercase, только при создании (disabled при редактировании)
  - **Название** (name) — text input
- Кнопки: "Сохранить", "Отмена"

**Логика удаления:**
- При удалении — проверить ошибку от бекенда (если страна используется в forms/validation_rules/step_conditions — вернуть 409)
- Показать сообщение об ошибке если удаление невозможно

---

### Задача F10. Admin Frontend: Роутинг и сборка

**Контекст:** Связать все страницы в единое приложение с правильным роутингом.

**Что сделать:**

**Создать `src/App.tsx`:**

```tsx
import { Routes, Route } from 'react-router-dom';
import AdminLayout from './components/AdminLayout';
import ProtectedRoute from './components/ProtectedRoute';
import AdminLoginPage from './pages/AdminLoginPage';
import DashboardPage from './pages/DashboardPage';
import StepsPage from './pages/StepsPage';
import StepFormPage from './pages/StepFormPage';
import ValidationRulesPage from './pages/ValidationRulesPage';
import ValidationRuleFormPage from './pages/ValidationRuleFormPage';
import MessagesPage from './pages/MessagesPage';
import MessageFormPage from './pages/MessageFormPage';
import CountriesPage from './pages/CountriesPage';
import CountryFormPage from './pages/CountryFormPage';

function App() {
  return (
    <Routes>
      <Route path="/login" element={<AdminLoginPage />} />
      <Route element={<ProtectedRoute />}>
        <Route path="/" element={<AdminLayout />}>
          <Route index element={<DashboardPage />} />
          <Route path="steps" element={<StepsPage />} />
          <Route path="steps/new" element={<StepFormPage />} />
          <Route path="steps/:stepName/edit" element={<StepFormPage />} />
          <Route path="validation-rules" element={<ValidationRulesPage />} />
          <Route path="validation-rules/new" element={<ValidationRuleFormPage />} />
          <Route path="validation-rules/:id/edit" element={<ValidationRuleFormPage />} />
          <Route path="messages" element={<MessagesPage />} />
          <Route path="messages/new" element={<MessageFormPage />} />
          <Route path="messages/:id/edit" element={<MessageFormPage />} />
          <Route path="countries" element={<CountriesPage />} />
          <Route path="countries/new" element={<CountryFormPage />} />
          <Route path="countries/:code/edit" element={<CountryFormPage />} />
        </Route>
      </Route>
    </Routes>
  );
}

export default App;
```

**Обновить `src/main.tsx`:**

```tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import 'bootstrap/dist/css/bootstrap.min.css';
import App from './App';

const queryClient = new QueryClient();

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <BrowserRouter>
      <QueryClientProvider client={queryClient}>
        <App />
      </QueryClientProvider>
    </BrowserRouter>
  </StrictMode>
);
```

**Обновить `vite.config.ts`:** — порт 5174, proxy на backend.

**Добавить `package.json` scripts:**
- `dev` — `vite`
- `build` — `tsc -b && vite build`
- `lint` — `eslint .`
- `preview` — `vite preview`

---

## Итоговая структура Admin-приложения

```
admin/
├── index.html
├── package.json
├── vite.config.ts
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.node.json
├── eslint.config.js
└── src/
    ├── main.tsx
    ├── App.tsx
    ├── types/
    │   └── index.ts
    ├── services/
    │   └── api.ts
    ├── store/
    │   └── adminStore.ts
    ├── components/
    │   ├── AdminLayout.tsx
    │   ├── ProtectedRoute.tsx
    │   └── StepConditionForm.tsx
    └── pages/
        ├── AdminLoginPage.tsx
        ├── DashboardPage.tsx
        ├── StepsPage.tsx
        ├── StepFormPage.tsx
        ├── ValidationRulesPage.tsx
        ├── ValidationRuleFormPage.tsx
        ├── MessagesPage.tsx
        ├── MessageFormPage.tsx
        ├── CountriesPage.tsx
        └── CountryFormPage.tsx
```

---

## Backend — новые/изменённые файлы для админки

| Файл | Действие | Описание |
|---|---|---|
| `controller/AdminAuthController.java` | **Новый** | POST `/admin/auth/login` |
| `dto/AdminLoginRequest.java` | **Новый** | DTO для админ-логина |
| `dto/AdminLoginResponse.java` | **Новый** | DTO для админ-логина |
| `controller/AdminStepController.java` | **Новый** | CRUD `/admin/steps` |
| `controller/AdminStepConditionController.java` | **Новый** | CRUD `/admin/steps/{stepName}/conditions` |
| `controller/AdminValidationRuleController.java` | **Новый** | CRUD `/admin/validation-rules` |
| `controller/AdminAppMessageController.java` | **Новый** | CRUD `/admin/messages` |
| `controller/AdminCountryController.java` | **Новый** | CRUD `/admin/countries` |
| `dto/StepCreateDto.java` | **Новый** | DTO для создания шага |
| `dto/StepUpdateDto.java` | **Новый** | DTO для обновления шага |
| `config/CorsConfig.java` | **Изменить** | Добавить `http://localhost:5174` |
| `config/AdminAuthInterceptor.java` | **Новый** (опционально) | Защита `/admin/**` |
| `Webinar2Application.java` | **Изменить** (опционально) | Регистрация interceptor |

---

## Порядок выполнения

```
Задача A1 (Admin auth endpoint)          — Backend
Задача A2 (Admin CRUD endpoints)          — Backend
Задача A3 (Admin interceptor, optional)   — Backend
    ↓
Задача F1 (Init admin project)            — Frontend
Задача F2 (Types + API service)           — Frontend
Задача F3 (Store + Layout)                — Frontend
Задача F4 (Login page)                    — Frontend
    ↓
Задача F10 (Routing + wiring)             — Frontend
    ↓
Задача F5 (Dashboard)                     — Frontend
Задача F9 (Countries — simplest)          — Frontend
Задача F7 (Validation Rules)              — Frontend
Задача F8 (Messages)                      — Frontend
Задача F6 (Steps — most complex)          — Frontend
```

Рекомендуемый порядок: backend сначала (A1-A2), затем frontend (F1-F4, F10, F5, F9, F7, F8, F6). Задача A3 (interceptor) опциональна и может быть отложена.

---

## Сводная таблица страниц админ-панели

| Страница | Путь | Описание |
|---|---|---|
| Login | `/login` | Вход администратора |
| Dashboard | `/` | Обзор конфигурации |
| Steps | `/steps` | Список шагов |
| Step Form | `/steps/new`, `/steps/:stepName/edit` | Создание/редактирование шага + условия |
| Validation Rules | `/validation-rules` | Список правил валидации |
| Validation Rule Form | `/validation-rules/new`, `/validation-rules/:id/edit` | Создание/редактирование правила |
| Messages | `/messages` | Список сообщений с фильтром |
| Message Form | `/messages/new`, `/messages/:id/edit` | Создание/редактирование сообщения |
| Countries | `/countries` | Список стран |
| Country Form | `/countries/new`, `/countries/:code/edit` | Создание/редактирование страны |
