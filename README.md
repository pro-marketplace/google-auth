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

> Для авторизации через Google нужно создать приложение в Google Cloud Console. Я помогу вам пошагово.
>
> 1. Откройте [Google Cloud Console](https://console.cloud.google.com/)
> 2. Создайте новый проект (или выберите существующий)
> 3. Перейдите в **APIs & Services** → **OAuth consent screen**
> 4. Выберите **External** и заполните:
>    - App name: название вашего приложения
>    - User support email: ваш email
>    - Developer contact: ваш email
> 5. На странице Scopes добавьте: `openid`, `email`, `profile`
> 6. **ВАЖНО:** На странице Test users добавьте свой email (пока приложение в режиме Testing, только эти пользователи могут входить)
> 7. Перейдите в **Credentials** → **Create Credentials** → **OAuth client ID**
> 8. Выберите **Web application**
> 9. Добавьте в **Authorized JavaScript origins**:
>    - `https://preview--{project}.poehali.dev` (для разработки)
>    - ваш продакшн домен (если есть)
> 10. Добавьте в **Authorized redirect URIs**:
>     - `https://preview--{project}.poehali.dev/auth/google/callback`
>     - продакшн redirect URI (если есть)
> 11. Нажмите Create и скопируйте **Client ID** и **Client Secret**
>
> Пришлите мне Client ID и Client Secret когда будут готовы.

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
4. Нажми **"Save and Continue"**
5. На странице Scopes нажми **"Add or Remove Scopes"**:
   - Выбери `openid`, `email`, `profile`
6. Нажми **"Save and Continue"**
7. **ВАЖНО:** На странице Test users добавь свой email — пока приложение в Testing, только эти пользователи могут входить!

### Шаг 3: Создание OAuth credentials

1. Перейди в **"APIs & Services"** → **"Credentials"**
2. Нажми **"+ Create Credentials"** → **"OAuth client ID"**
3. Выбери **"Web application"**
4. Введи название
5. **Authorized JavaScript origins** — добавь ОБА домена:

| Среда | Origin |
|-------|--------|
| **Разработка** | `https://preview--{project}.poehali.dev` |
| **Продакшн** | `https://your-domain.com` или `https://{project}.poehali.dev` |

6. **Authorized redirect URIs** — добавь ОБА (должны ТОЧНО совпадать с `GOOGLE_REDIRECT_URI`):

| Среда | Redirect URI |
|-------|--------------|
| **Разработка** | `https://preview--{project}.poehali.dev/auth/google/callback` |
| **Продакшн** | `https://your-domain.com/auth/google/callback` |

7. Нажми **"Create"**
8. Скопируй **Client ID** и **Client Secret**

### Шаг 4: Публикация приложения (когда готов к продакшну)

Пока приложение в режиме **Testing**, только добавленные Test users могут входить.

Для публикации:
1. Перейди в **"OAuth consent screen"**
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
