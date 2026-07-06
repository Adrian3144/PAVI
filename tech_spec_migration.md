# Техническая спецификация миграции: ФитТреккер → Telegram Mini App

**Версия:** 1.0
**Дата:** 2026-07-04
**Источник:** прототип из Figma Make (Vite + React 18 + TypeScript + Tailwind v4 + shadcn/ui)
**Цель:** миграция с localStorage на серверное хранение с авторизацией через Telegram, интеграция Telegram SDK

---

## 0. Контекст и допущения

- Текущий фронтенд (`App.tsx`, компоненты `WorkoutCard`, `SetRow`, `ExerciseSearch`, `ProgressChart`, `VoiceRecorder`) переиспользуется без переписывания внутренней логики рендеринга — меняется только источник данных (localStorage → API).
- Бэкенд — существующая Supabase Edge Function на Hono (`supabase/functions/server/index.tsx`), сейчас содержит только health-check. Дописывается с нуля по этой спецификации.
- `AuthScreen.tsx` (email/password через Supabase Auth) — не используется, заменяется полностью логикой Telegram initData.
- Окружения: два отдельных Supabase-проекта — `prod` и `dev`, разные БД, разные переменные окружения.
- Нет TODO-заглушек — каждый эндпоинт описан с полным телом запроса/ответа и кодами ошибок.

---

## Модуль 1: Telegram-авторизация

### 1.1 User Stories
- Как пользователь, я открываю бота в Telegram и Mini App авторизует меня автоматически, без ввода логина/пароля.
- Как пользователь, открывший Mini App повторно (в том числе с другого устройства), я вижу свои же данные, потому что личность подтверждена через мой Telegram-аккаунт.
- Как разработчик, я хочу быть уверен, что запрос действительно пришёл из Telegram, а не подделан произвольным HTTP-клиентом.
- Edge case: пользователь открыл Mini App вне Telegram (напрямую по URL в браузере) → `initData` отсутствует → приложение показывает экран "Откройте через Telegram", доступ к данным не выдаётся.
- Edge case: `initData` есть, но подпись не совпадает (истёк срок действия, подделка) → 401, приложение показывает ошибку авторизации с кнопкой "Повторить".

### 1.2 Модель данных

```sql
create table public.users (
  id uuid primary key default gen_random_uuid(),
  telegram_id bigint unique not null,
  username text,
  first_name text,
  last_name text,
  photo_url text,
  language_code text,
  is_premium boolean default false,      -- задел под платные тарифы (не Telegram Premium)
  created_at timestamptz default now(),
  last_seen_at timestamptz default now()
);

create index idx_users_telegram_id on public.users(telegram_id);
```

**RLS-политики:**
```sql
alter table public.users enable row level security;

-- Пользователь видит и обновляет только свою запись.
-- auth.uid() сопоставляется с users.id через кастомный JWT, выданный после валидации initData (см. 1.3).
create policy "users_select_own" on public.users
  for select using (auth.uid() = id);

create policy "users_update_own" on public.users
  for update using (auth.uid() = id);

-- INSERT выполняется только сервисной ролью (Edge Function), не напрямую с клиента
```

### 1.3 API

**`POST /auth/telegram`**
Вызывается фронтендом сразу при старте приложения.

Тело запроса:
```json
{ "initData": "query_id=AAH...&user=%7B%22id%22...&auth_date=1751... &hash=abc123..." }
```

Логика на сервере:
1. Распарсить `initData` как query-string.
2. Извлечь поле `hash`, удалить его из набора данных.
3. Отсортировать оставшиеся пары `key=value` по алфавиту, склеить через `\n`.
4. Вычислить `secret_key = HMAC-SHA256("WebAppData", BOT_TOKEN)`.
5. Вычислить `computed_hash = HMAC-SHA256(secret_key, data_check_string)`.
6. Сравнить `computed_hash` с `hash` из запроса. Не совпало → 401.
7. Проверить `auth_date` — не старше 24 часов (защита от replay-атак). Просрочено → 401.
8. Найти или создать пользователя по `telegram_id` из поля `user`.
9. Обновить `last_seen_at`.
10. Выдать JWT (через `supabase.auth.admin.generateLink` или кастомный подписанный токен) с `sub = users.id`.

Ответ 200:
```json
{
  "access_token": "eyJhbGciOi...",
  "user": {
    "id": "b3f1...-uuid",
    "telegram_id": 123456789,
    "username": "ivan_petrov",
    "first_name": "Иван",
    "is_premium": false
  }
}
```

Ответ 401:
```json
{ "error": "invalid_signature" }
```
или
```json
{ "error": "auth_date_expired" }
```

### 1.4 Экраны/компоненты
- Новый компонент `TelegramAuthGate.tsx`: обёртка вокруг `App.tsx`. Состояния: `loading` (спиннер, длится долю секунды), `error` (initData отсутствует или не прошла валидацию — текст + кнопка "Повторить"), `ready` (рендерит `App.tsx`, передаёт `access_token` и `user` через контекст).
- `AuthScreen.tsx` — удаляется из дерева компонентов (файл можно оставить в репозитории неиспользуемым или удалить полностью).

### 1.5 Бизнес-логика
- Токен хранится в памяти (React Context/state), не в localStorage — при перезапуске Mini App происходит повторная валидация `initData` (она у Telegram всегда свежая при каждом открытии).
- `is_premium` в таблице `users` — это будущий флаг платной подписки внутри приложения, не путать с Telegram Premium.

### 1.6 Крайние случаи
| Ситуация | Ожидаемое поведение |
|---|---|
| `initData` пустая строка | Показать экран "Откройте через Telegram" |
| Подпись не совпадает | 401 → экран ошибки, retry |
| `auth_date` старше 24ч | 401 → фронт автоматически запрашивает свежий `initData` через `window.Telegram.WebApp.initData` и повторяет запрос |
| Пользователь заблокировал бота, потом снова открыл | `telegram_id` уже существует → просто обновляется `last_seen_at`, данные тренировок сохранены |

---

## Модуль 2: Хранение тренировок (замена localStorage)

### 2.1 User Stories
- Как пользователь, я создаю тренировку, и она сохраняется на сервере, а не только в браузере.
- Как пользователь, я удаляю Telegram и переустанавливаю — открыв Mini App заново, вижу все свои тренировки.
- Как пользователь, я добавляю подход (вес, повторения) в реальном времени во время тренировки.
- Edge case: потеря сети во время сохранения подхода → показать индикатор ошибки на конкретной строке, не блокировать весь экран, дать retry.
- Edge case: два устройства одновременно редактируют одну тренировку → последняя запись побеждает (last-write-wins), без сложного merge на MVP-этапе.

### 2.2 Модель данных

```sql
create table public.workouts (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references public.users(id) on delete cascade,
  name text not null,
  status text not null default 'in-progress', -- 'in-progress' | 'completed'
  started_at timestamptz default now(),
  completed_at timestamptz,
  duration_seconds integer default 0,
  created_at timestamptz default now()
);

create table public.exercises (
  id uuid primary key default gen_random_uuid(),
  workout_id uuid not null references public.workouts(id) on delete cascade,
  name text not null,
  position integer not null,          -- порядок в тренировке (drag & drop)
  parent_exercise_id uuid references public.exercises(id) on delete cascade, -- для суперсетов (subExercises)
  created_at timestamptz default now()
);

create table public.sets (
  id uuid primary key default gen_random_uuid(),
  exercise_id uuid not null references public.exercises(id) on delete cascade,
  position integer not null,
  weight numeric,
  reps integer,
  completed boolean default false,
  created_at timestamptz default now()
);

create index idx_workouts_user_id on public.workouts(user_id);
create index idx_exercises_workout_id on public.exercises(workout_id);
create index idx_sets_exercise_id on public.sets(exercise_id);
```

**RLS-политики** (одинаковый паттерн для всех трёх таблиц, пример для `workouts`):
```sql
alter table public.workouts enable row level security;

create policy "workouts_select_own" on public.workouts
  for select using (auth.uid() = user_id);

create policy "workouts_insert_own" on public.workouts
  for insert with check (auth.uid() = user_id);

create policy "workouts_update_own" on public.workouts
  for update using (auth.uid() = user_id);

create policy "workouts_delete_own" on public.workouts
  for delete using (auth.uid() = user_id);

-- Для exercises и sets — RLS через join на workouts.user_id (exists-подзапрос),
-- т.к. у них нет прямого user_id
```

### 2.3 API

| Метод + путь | Тело | Ответ | Ошибки |
|---|---|---|---|
| `GET /workouts` | — | `200 [{id, name, status, started_at, exercises: [...]}]` | 401 |
| `POST /workouts` | `{name}` | `201 {id, name, status: 'in-progress', started_at}` | 400 (пустое имя), 401 |
| `PATCH /workouts/:id` | `{name?, status?, completed_at?, duration_seconds?}` | `200 {...обновлённая тренировка}` | 404, 401, 403 (чужая тренировка) |
| `DELETE /workouts/:id` | — | `204` | 404, 401, 403 |
| `POST /workouts/:id/exercises` | `{name, position, parent_exercise_id?}` | `201 {id, name, position}` | 400, 404, 401 |
| `PATCH /exercises/:id` | `{name?, position?}` | `200 {...}` | 404, 401, 403 |
| `DELETE /exercises/:id` | — | `204` | 404, 401, 403 |
| `POST /exercises/:id/sets` | `{position, weight?, reps?}` | `201 {id, weight, reps, completed: false}` | 400, 404, 401 |
| `PATCH /sets/:id` | `{weight?, reps?, completed?}` | `200 {...}` | 404, 401, 403 |
| `DELETE /sets/:id` | — | `204` | 404, 401, 403 |

Все запросы требуют заголовок `Authorization: Bearer <access_token>` из Модуля 1.

### 2.4 Экраны/компоненты
- `WorkoutCard.tsx`, `SetRow.tsx`, `ExerciseSearch.tsx` — логика рендера не меняется, меняются только колбэки: вместо прямого вызова `localStorage.setItem` — вызов соответствующего API-метода через новый слой `src/lib/api.ts`.
- Добавить состояния `isLoading` / `isSaving` / `error` на уровне `App.tsx` — сейчас их нет, т.к. localStorage синхронный.
- Добавить optimistic update: изменение сразу отражается в UI, запрос уходит в фоне; при ошибке — откат + toast через `sonner` (уже используется в проекте).

### 2.5 Бизнес-логика
- `duration_seconds` пересчитывается на клиенте каждую секунду (как сейчас через `setInterval`), но сохраняется на сервер только при явном действии (завершение тренировки или переход на другой экран), а не каждую секунду — чтобы не заваливать API запросами.
- `position` для exercises/sets пересчитывается полностью при каждом drag & drop и отправляется батчем (`PATCH /workouts/:id/reorder` — опционально, если понадобится оптимизация).

### 2.6 Крайние случаи
| Ситуация | Ожидаемое поведение |
|---|---|
| `GET /workouts` вернул пустой список | Показать текущий пустой стейт (уже есть в `renderDashboard`) |
| Сеть недоступна при создании тренировки | Toast с ошибкой, кнопка "Создать тренировку" не блокируется навсегда — retry вручную |
| Попытка изменить чужую тренировку (чужой `user_id`) | 403 от RLS/API → toast "Нет доступа" |
| Одновременное редактирование с двух вкладок | Последний `PATCH` побеждает, без конфликт-резолюшна на MVP |

---

## Модуль 3: Интеграция Telegram SDK

### 3.1 User Stories
- Как пользователь, я разворачиваю приложение на весь экран без лишних действий.
- Как пользователь, я нажимаю системную кнопку "Назад" Telegram и попадаю на предыдущий экран приложения (а не закрываю Mini App).
- Как пользователь, интерфейс подстраивается под мою тему Telegram (светлая/тёмная).

### 3.2 Модель данных
Не требуется (только фронтенд).

### 3.3 API
Не требуется (используется клиентский Telegram WebApp SDK, не HTTP).

### 3.4 Экраны/компоненты
- `src/main.tsx`: вызов `WebApp.ready()` и `WebApp.expand()` при старте.
- Новый хук `useTelegramBackButton(screen, setScreen)`: показывает `BackButton` когда `screen !== 'dashboard'`, скрывает на `dashboard`; `onClick` вызывает `setScreen('dashboard')` либо предыдущий экран по стеку.
- Новый модуль `src/lib/telegramTheme.ts`: подписка на `themeParams`, проброс в CSS-переменные (`--tg-theme-bg-color` и т.д.), используемые вместо хардкода `bg-black`/`bg-zinc-900` в Tailwind-классах.

### 3.5 Бизнес-логика
- Если приложение открыто вне Telegram (`window.Telegram?.WebApp` отсутствует) — SDK-вызовы оборачиваются в проверку наличия объекта, чтобы не падать в обычном браузере при локальной разработке.

### 3.6 Крайние случаи
| Ситуация | Ожидаемое поведение |
|---|---|
| Открыто в обычном браузере (разработка) | SDK-вызовы — no-op, приложение работает как обычный сайт |
| Пользователь на Android с вырезом экрана | Использовать `viewportStableHeight` вместо `100dvh` там, где сейчас захардкожено |
| Смена темы Telegram во время сессии | Слушатель `themeChanged` обновляет CSS-переменные на лету |

---

## Модуль 4: Базовая аналитика (задел)

### 4.1 User Stories
- Как продакт, я хочу видеть, сколько пользователей создают тренировку в первый день после открытия бота.
- Как продакт, я хочу видеть частоту создания тренировок на пользователя.

### 4.2 Модель данных
Не требуется — используется внешний сервис (PostHog), события отправляются напрямую с фронтенда через SDK.

### 4.3 API
Не требуется на этом этапе — задел, не полная реализация.

### 4.4 События для трекинга (минимальный набор)
| Событие | Когда отправляется | Свойства |
|---|---|---|
| `app_opened` | При успешной авторизации через Модуль 1 | `telegram_id` |
| `workout_created` | При `POST /workouts` успешен | `workout_id` |
| `workout_completed` | При смене статуса на `completed` | `workout_id`, `duration_seconds` |
| `set_logged` | При `POST /exercises/:id/sets` успешен | `exercise_id`, `weight`, `reps` |

### 4.5 Крайние случаи
Не применимо на этапе задела — полноценная обработка ошибок аналитики (retry, батчинг) выносится за рамки MVP-миграции.

---

## Приоритет и зависимости

1. **Модуль 1 (авторизация)** — блокирует всё остальное, делается первым.
2. **Модуль 2 (хранение тренировок)** — зависит от Модуля 1 (нужен `user_id`).
3. **Модуль 3 (Telegram SDK)** — независим, можно делать параллельно с Модулем 2.
4. **Модуль 4 (аналитика)** — низкий приоритет, добавляется последним, не блокирует релиз MVP.

---

## Готовность к передаче в Claude Code

- [x] Каждый модуль: user stories + модель данных + API + экраны + бизнес-логика + edge cases
- [x] Типы полей как в SQL (uuid, text, timestamptz, numeric, boolean)
- [x] API: метод + путь + тело + ответ + коды ошибок
- [x] RLS-политики для каждой таблицы
- [x] Нет TODO-заглушек
- [ ] CLAUDE.md и субагенты — следующий шаг после утверждения этой спецификации
