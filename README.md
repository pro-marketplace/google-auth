# Google Auth Extension

SSO авторизация через Google. **1 функция** с роутингом по action.

> ⚠️ **Авторизация через Google не работает в редакторе!**
>
> Google блокирует работу в iframe. Для проверки авторизации откройте сайт **в отдельной вкладке браузера**.

---

## Для ассистента: перед интеграцией

1. **Проверь таблицу users** — должен быть столбец `google_id`:
   ```sql
   ALTER TABLE users ADD COLUMN IF NOT EXISTS google_id VARCHAR(50);
   ALTER TABLE users ADD COLUMN IF NOT EXISTS avatar_url TEXT;
   CREATE INDEX IF NOT EXISTS idx_users_google_id ON users(google_id);
   ```

2. **ОБЯЗАТЕЛЬНО добавь ВСЕ секреты через тулу добавления секретов:**
   ```
   GOOGLE_CLIENT_ID=<Client ID>
   GOOGLE_CLIENT_SECRET=<Client Secret>
   GOOGLE_REDIRECT_URI=<Authorized redirect URI>
   JWT_SECRET=<сгенерируй: openssl rand -hex 32>
   ```

   **Покажи пользователю в чате вызов тулы для каждого секрета!**

3. **Создай страницу callback** — на неё Google перенаправит после авторизации

4. **После успешной авторизации ОБЯЗАТЕЛЬНО перенаправь на страницу профиля** — покажи пользователю его данные (имя, email, аватар)

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
4. Нажми **"Save and Continue"**
5. На странице Scopes нажми **"Add or Remove Scopes"**:
   - Выбери `openid`, `email`, `profile`
6. Нажми **"Save and Continue"**
7. На странице Test users добавь тестовых пользователей (пока приложение в режиме Testing)

### Шаг 3: Создание OAuth credentials

1. Перейди в **"APIs & Services"** → **"Credentials"**
2. Нажми **"+ Create Credentials"** → **"OAuth client ID"**
3. Выбери **"Web application"**
4. Введи название
5. **Authorized JavaScript origins** — добавь ОБА домена:

| Среда | Origin |
|-------|--------|
| **Разработка** | `https://preview--{project}.poehali.dev` |
| **Продакшн** | `https://your-domain.com` или `https://{project}--preview.poehali.dev` |

6. **Authorized redirect URIs** — добавь ОБА:

| Среда | Redirect URI |
|-------|--------------|
| **Разработка** | `https://preview--{project}.poehali.dev/auth/google/callback` |
| **Продакшн** | `https://your-domain.com/auth/google/callback` |

7. Нажми **"Create"**
8. Скопируй **Client ID** и **Client Secret**

### Шаг 4: Публикация приложения (опционально)

Пока приложение в режиме **Testing**, только добавленные тестовые пользователи могут входить.

Для публикации:
1. Перейди в **"OAuth consent screen"**
2. Нажми **"Publish App"**
3. Пройди верификацию Google (может занять время)

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
| `GOOGLE_CLIENT_ID` | Client ID | Google Cloud Console → Credentials |
| `GOOGLE_CLIENT_SECRET` | Client Secret | Google Cloud Console → Credentials |
| `GOOGLE_REDIRECT_URI` | Redirect URI | Должен совпадать с Authorized redirect URI |
| `JWT_SECRET` | Секрет для токенов (мин. 32 символа) | `openssl rand -hex 32` |
| `ALLOWED_ORIGINS` | (опционально) Разрешённые домены через запятую | `https://example.com,https://app.example.com` |

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
const AUTH_URL = "https://functions.poehali.dev/xxx";

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

Создай страницу `/auth/google/callback` которая:
1. Вызывает `auth.handleCallback()` — автоматически извлечёт code и state из URL
2. Редиректит на страницу профиля при успехе

```tsx
// pages/auth/google/callback.tsx или app/auth/google/callback/page.tsx
"use client";

import { useEffect } from "react";
import { useRouter } from "next/navigation";
import { useGoogleAuth } from "@/hooks/useGoogleAuth";

export default function GoogleCallbackPage() {
  const router = useRouter();
  const auth = useGoogleAuth({
    apiUrls: {
      authUrl: `${process.env.NEXT_PUBLIC_GOOGLE_AUTH_URL}?action=auth-url`,
      callback: `${process.env.NEXT_PUBLIC_GOOGLE_AUTH_URL}?action=callback`,
      refresh: `${process.env.NEXT_PUBLIC_GOOGLE_AUTH_URL}?action=refresh`,
      logout: `${process.env.NEXT_PUBLIC_GOOGLE_AUTH_URL}?action=logout`,
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
