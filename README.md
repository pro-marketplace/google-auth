# Google Auth Extension

SSO авторизация через Google. **1 функция** с роутингом по action.

> ⚠️ **Авторизация через Google не работает в редакторе!**
>
> Google блокирует работу в iframe. Для проверки авторизации откройте сайт **в отдельной вкладке браузера**.

---

## Для ассистента: чеклист интеграции

### Шаг 1: Подготовка базы данных

Проверь/добавь столбец `google_id` в таблицу users:
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
>
> **Примечание:** Пока приложение в режиме Testing, войти смогут только пользователи добавленные в Test users (Audience → Add users).

### Шаг 3: Добавление секретов

**ОБЯЗАТЕЛЬНО добавь ВСЕ секреты через тулу. Покажи пользователю вызов тулы для каждого!**

```
GOOGLE_CLIENT_ID=<Client ID от пользователя>
GOOGLE_CLIENT_SECRET=<Client Secret от пользователя>
GOOGLE_REDIRECT_URI=https://preview--{project}.poehali.dev/auth/google/callback
JWT_SECRET=<сгенерируй сам: 64 случайных символа>
```

**JWT_SECRET генерируй сам**, не спрашивай у пользователя!

### Шаг 4: Создание страниц

1. **Страница с кнопкой входа** — добавь `GoogleLoginButton`
2. **Страница callback** `/auth/google/callback` — обработка редиректа от Google
3. **Страница профиля** — показать данные пользователя после входа

### Шаг 5: После установки скажи пользователю

> ✅ Авторизация через Google настроена!
>
> **Важно:**
> - Проверяйте авторизацию **в отдельной вкладке**, не в редакторе
> - Пока приложение в режиме Testing, войти могут только пользователи из списка Test Users
> - Для публичного доступа нужно опубликовать приложение в Google Cloud Console (OAuth consent screen → Publish App)

---

## Создание приложения в Google Cloud Console

### Шаг 1: Создание проекта

1. Перейди в [Google Cloud Console](https://console.cloud.google.com/)
2. Нажми на селектор проектов → **"New Project"**
3. Введи название проекта
4. Нажми **"Create"**

### Шаг 2: Настройка OAuth consent screen

1. В меню выбери **"APIs & Services"** → **"OAuth consent screen"**
2. Выбери **"External"** (для публичных приложений)
3. Заполни обязательные поля:
   - **App name** — название приложения
   - **User support email** — email поддержки
   - **Developer contact email** — email разработчика
4. Нажми **"Create"** (или "Save and Continue")
5. Ты окажешься на странице **OAuth Overview**

### Шаг 3: Создание OAuth credentials

1. На странице **OAuth Overview** нажми **"Create OAuth client"**
2. Выбери **"Web application"**
3. Введи название
4. **Authorized JavaScript origins** — добавь домен(ы):
   - `https://your-domain.com`
5. **Authorized redirect URIs** — добавь (должен ТОЧНО совпадать с `GOOGLE_REDIRECT_URI`):
   - `https://your-domain.com/auth/google/callback`
6. Нажми **"Create"**

### Шаг 4: Получение Client ID и Client Secret

После создания появится модалка с **Client ID** — скопируй его (иконка копирования справа).

**Для получения Client Secret:**
1. Закрой модалку (нажми OK)
2. В левом меню перейди в **Clients**
3. Найди созданный клиент и нажми на него
4. На странице клиента будет **Client ID** и **Client secret** — скопируй оба

### Шаг 5: Test users (пока в режиме Testing)

Пока приложение в режиме **Testing**, войти могут только добавленные пользователи:

1. Перейди в **Audience** (в левом меню)
2. Нажми **"Add users"**
3. Добавь email пользователей для тестирования

### Шаг 6: Публикация (когда готов к продакшну)

Для публичного доступа:
1. Перейди в **OAuth consent screen**
2. Нажми **"Publish App"**
3. Подтверди публикацию

---

## Установка

### 1. База данных

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    google_id VARCHAR(50) UNIQUE,
    email VARCHAR(255),
    name VARCHAR(255),
    avatar_url TEXT,
    email_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login_at TIMESTAMP
);

CREATE TABLE refresh_tokens (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(64) NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_google_id ON users(google_id);
CREATE INDEX idx_refresh_tokens_hash ON refresh_tokens(token_hash);
```

### 2. Переменные окружения

| Переменная | Описание | Где взять |
|------------|----------|-----------|
| `GOOGLE_CLIENT_ID` | Client ID | Пользователь даст после создания OAuth credentials |
| `GOOGLE_CLIENT_SECRET` | Client Secret | Пользователь даст после создания OAuth credentials |
| `GOOGLE_REDIRECT_URI` | Redirect URI | `https://preview--{project}.poehali.dev/auth/google/callback` |
| `JWT_SECRET` | Секрет для токенов (мин. 32 символа) | **Сгенерируй сам!** |
| `ALLOWED_ORIGINS` | (опционально) Разрешённые домены | `https://example.com,https://app.example.com` |

---

## API

```
GET  ?action=auth-url  — получить URL для авторизации Google
POST ?action=callback  — обмен code на токены (body: { code })
POST ?action=refresh   — обновление токена (body: { refresh_token })
POST ?action=logout    — выход (body: { refresh_token })
```

---

## Frontend

| Файл | Описание |
|------|----------|
| `useGoogleAuth.ts` | Хук авторизации |
| `GoogleLoginButton.tsx` | Кнопка "Войти через Google" |
| `UserProfile.tsx` | Профиль пользователя |

```tsx
// Получи URL функции после деплоя
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

Создай страницу `/auth/google/callback`:

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
        router.push("/profile"); // Редирект на страницу профиля
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
6. Frontend извлекает code из URL
7. Frontend → POST ?action=callback { code }
8. Backend обменивает code на токены через Google API
9. Редирект на страницу профиля
```

---

## Безопасность

- JWT access tokens (15 мин)
- Refresh tokens хешируются (SHA256) перед сохранением в БД
- Автоочистка протухших токенов при каждом запросе
- CSRF protection через state параметр
- Параметризованные SQL-запросы (защита от SQL injection)
- Валидация JWT_SECRET (минимум 32 символа)
- CORS ограничение через `ALLOWED_ORIGINS`
- Скрытие внутренних ошибок от клиента
