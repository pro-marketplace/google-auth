# Google Auth Extension

SSO авторизация через Google. **1 функция** с роутингом по action.

> ⚠️ **Авторизация через Google не работает в редакторе!**
>
> Google блокирует работу в iframe. Для проверки авторизации откройте сайт **в отдельной вкладке браузера**.

---

# [AUTH] Общее для виджетов авторизации

## Логика привязки аккаунтов

Функция автоматически связывает аккаунты по email:

1. **Поиск по provider_id** (google_id/vk_id) → если найден, логиним
2. **Поиск по email** → если найден, привязываем провайдера к существующему аккаунту
3. **Новый пользователь** → создаём запись

Это позволяет пользователю войти через любой провайдер, если email совпадает.

## Требования к базе данных

Функция работает с полями таблицы `users`:

| Поле | Тип | Требования |
|------|-----|------------|
| `google_id` / `vk_id` | VARCHAR(50) | Может быть NULL |
| `email` | VARCHAR(255) | Может быть NULL |
| `name` | VARCHAR(255) | Может быть NULL |
| `avatar_url` | TEXT | Может быть NULL |
| `email_verified` | BOOLEAN | DEFAULT FALSE/TRUE |
| `created_at` | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP |
| `updated_at` | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP |
| `last_login_at` | TIMESTAMP | Может быть NULL |

**⚠️ Если столбец имеет `NOT NULL` без `DEFAULT` — функция упадёт при вставке!**

### Проверка структуры БД перед установкой

**ОБЯЗАТЕЛЬНО перед установкой расширения:**

1. Выполни `SELECT column_name, is_nullable, column_default FROM information_schema.columns WHERE table_name = 'users';`
2. Найди все столбцы с `is_nullable = 'NO'` и `column_default = NULL`
3. Такие столбцы вызовут ошибку при INSERT, если мы не передаём для них значение
4. **Типичные проблемы:**
   - `password_hash NOT NULL` — передай пустую строку `''`
   - `role NOT NULL` — передай дефолтное значение (например `'user'`)
   - Кастомные поля проекта — уточни у пользователя

**Если нашёл NOT NULL поля без DEFAULT — добавь их в INSERT с дефолтными значениями!**

## Добавление секретов

**ОБЯЗАТЕЛЬНО вызови тулу `put_secret` для КАЖДОГО секрета!**

```python
put_secret("PROVIDER_CLIENT_ID", "<от пользователя>")
put_secret("PROVIDER_CLIENT_SECRET", "<от пользователя>")
put_secret("PROVIDER_REDIRECT_URI", "https://{домен}/auth/{provider}/callback")
put_secret("JWT_SECRET", "<сгенерируй сам>")
```

- Вызывай `put_secret` для КАЖДОГО секрета отдельно
- **JWT_SECRET генерируй сам** (64 hex символа), не спрашивай у пользователя
- Покажи пользователю в чате каждый вызов тулы

## После установки

**ОБЯЗАТЕЛЬНО:** Последним шагом добавь кнопку входа на фронтенд, чтобы пользователь сразу видел результат!

Используй компоненты из `/frontend`:
- `GoogleLoginButton.tsx` — кнопка входа
- `useGoogleAuth.ts` — хук авторизации

**Скажи пользователю:**

> ✅ Авторизация настроена!
>
> **Важно:**
> - Проверяйте авторизацию **в отдельной вкладке**, не в редакторе
> - Если email уже использовался — аккаунт автоматически привяжется

## API (общий для всех провайдеров)

```
GET  ?action=auth-url  — получить URL для авторизации
POST ?action=callback  — обмен code на токены (body: { code, ... })
POST ?action=refresh   — обновление токена (body: { refresh_token })
POST ?action=logout    — выход (body: { refresh_token })
```

## Безопасность

- JWT access tokens (15 мин)
- Refresh tokens хешируются (SHA256) перед сохранением
- Автоочистка протухших токенов при каждом запросе
- CSRF protection через state параметр
- Параметризованные SQL-запросы
- Валидация JWT_SECRET (минимум 32 символа)
- CORS ограничение через `ALLOWED_ORIGINS`
- Скрытие внутренних ошибок от клиента

---

# [GOOGLE] Специфичное для Google Auth

## Чеклист интеграции

### Шаг 1: Подготовка базы данных

```sql
ALTER TABLE users ADD COLUMN IF NOT EXISTS google_id VARCHAR(50);
ALTER TABLE users ADD COLUMN IF NOT EXISTS avatar_url TEXT;
CREATE INDEX IF NOT EXISTS idx_users_google_id ON users(google_id);
```

### Шаг 2: Сопровождение пользователя в Google Cloud Console

**Скажи пользователю:**

> Для авторизации через Google нужно создать приложение в Google Cloud Console. Я помогу пошагово:
>
> 1. Откройте [Google Cloud Console](https://console.cloud.google.com/)
> 2. Создайте новый проект (или выберите существующий)
> 3. Перейдите в **APIs & Services** → **OAuth consent screen**
> 4. Выберите **External** и заполните:
>    - App name: название вашего приложения
>    - User support email: ваш email
>    - Developer contact: ваш email
> 5. После создания вы окажетесь на странице **OAuth Overview** — нажмите **"Create OAuth client"**
> 6. Выберите **Web application**
> 7. Добавьте в **Authorized JavaScript origins**:
>    - `https://{ваш-домен}` (например: `https://coder.arnld.ai`)
> 8. Добавьте в **Authorized redirect URIs**:
>    - `https://{ваш-домен}/auth/google/callback` (например: `https://coder.arnld.ai/auth/google/callback`)
> 9. Нажмите **Create**
> 10. В появившейся модалке скопируйте **Client ID** (иконка копирования справа)
> 11. Нажмите **OK**, затем в левом меню перейдите в **Clients**
> 12. Нажмите на созданный клиент — там будет **Client secret**
>
> Пришлите мне **Client ID** и **Client Secret** когда будут готовы!

### Шаг 3: Добавление секретов

Когда пользователь пришлёт Client ID и Client Secret:

```python
put_secret("GOOGLE_CLIENT_ID", "<Client ID от пользователя>")
put_secret("GOOGLE_CLIENT_SECRET", "<Client Secret от пользователя>")
put_secret("GOOGLE_REDIRECT_URI", "https://{домен-пользователя}/auth/google/callback")
put_secret("JWT_SECRET", "<сгенерируй: 64 hex символа>")
```

### Шаг 4: Создание страниц

1. **Страница с кнопкой входа** — добавь `GoogleLoginButton`
2. **Страница callback** `/auth/google/callback` — обработка редиректа
3. **Страница профиля** — показать данные пользователя после входа

---

## Создание приложения в Google Cloud Console (детально)

### Шаг 1: Создание проекта

1. Перейди в [Google Cloud Console](https://console.cloud.google.com/)
2. Нажми на селектор проектов → **"New Project"**
3. Введи название проекта
4. Нажми **"Create"**

### Шаг 2: Настройка OAuth consent screen

1. В меню выбери **"APIs & Services"** → **"OAuth consent screen"**
2. Выбери **"External"**
3. Заполни обязательные поля:
   - **App name** — название приложения
   - **User support email** — email поддержки
   - **Developer contact email** — email разработчика
4. Нажми **"Create"**
5. Ты окажешься на странице **OAuth Overview**

### Шаг 3: Создание OAuth credentials

1. Нажми **"Create OAuth client"**
2. Выбери **"Web application"**
3. **Authorized JavaScript origins**: `https://your-domain.com`
4. **Authorized redirect URIs**: `https://your-domain.com/auth/google/callback`
5. Нажми **"Create"**

### Шаг 4: Получение Client ID и Client Secret

1. В модалке скопируй **Client ID** (иконка справа)
2. Нажми **OK**
3. Перейди в **Clients** (левое меню)
4. Нажми на созданный клиент
5. Скопируй **Client secret**

---

## Frontend компоненты

| Файл | Описание |
|------|----------|
| `useGoogleAuth.ts` | Хук авторизации |
| `GoogleLoginButton.tsx` | Кнопка "Войти через Google" |
| `UserProfile.tsx` | Профиль пользователя |

### Пример использования

```tsx
const AUTH_URL = "https://functions.poehali.dev/xxx-google-auth";

const auth = useGoogleAuth({
  apiUrls: {
    authUrl: `${AUTH_URL}?action=auth-url`,
    callback: `${AUTH_URL}?action=callback`,
    refresh: `${AUTH_URL}?action=refresh`,
    logout: `${AUTH_URL}?action=logout`,
  },
});

// Кнопка входа
<GoogleLoginButton onClick={auth.login} isLoading={auth.isLoading} />

// После авторизации
if (auth.isAuthenticated && auth.user) {
  return <UserProfile user={auth.user} onLogout={auth.logout} />;
}
```

### Страница callback

```tsx
// app/auth/google/callback/page.tsx
"use client";

import { useEffect } from "react";
import { useRouter } from "next/navigation";
import { useGoogleAuth } from "@/hooks/useGoogleAuth";

const AUTH_URL = "https://functions.poehali.dev/xxx-google-auth";

export default function GoogleCallbackPage() {
  const router = useRouter();
  const auth = useGoogleAuth({
    apiUrls: {
      authUrl: `${AUTH_URL}?action=auth-url`,
      callback: `${AUTH_URL}?action=callback`,
      refresh: `${AUTH_URL}?action=refresh`,
      logout: `${AUTH_URL}?action=logout`,
    },
  });

  useEffect(() => {
    auth.handleCallback().then((success) => {
      if (success) {
        router.push("/profile");
      }
    });
  }, []);

  return (
    <div className="flex items-center justify-center min-h-screen">
      <p>Авторизация...</p>
    </div>
  );
}
```

---

## Поток авторизации

```
1. Пользователь нажимает "Войти через Google"
2. Frontend → GET ?action=auth-url → получает auth_url + state
3. Frontend сохраняет state в sessionStorage
4. Редирект на Google для авторизации
5. Google → редирект на callback с ?code=...&state=...
6. Frontend → POST ?action=callback { code }
7. Backend обменивает code на токены
8. Backend проверяет google_id → email → создаёт/привязывает пользователя
9. Редирект на страницу профиля
```
