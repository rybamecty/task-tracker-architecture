# Task Tracker - Архитектура проекта

**Автор:** Виктория Алексеенко  
**Дата:** 23 октября 2025

**Описание:** Архитектурное решение для системы управления задачами с календарём

---

## 1. Структура данных

### 1.1 Основные сущности

#### Таблица users

Хранит информацию о пользователях системы.

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_users_email ON users(email);
```

**Поля:**
- `id` — уникальный идентификатор пользователя
- `email` — email для авторизации (уникальный)
- `password_hash` — хэш пароля (bcrypt)
- `full_name` — имя пользователя для отображения
- `created_at` — дата регистрации
- `updated_at` — дата последнего обновления профиля
- `is_active` — статус активности аккаунта

#### Таблица tasks

Хранит задачи пользователей с привязкой к дате и времени.

```sql
CREATE TABLE tasks (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    due_date DATE NOT NULL,
    due_time TIME,
    status VARCHAR(50) DEFAULT 'pending',
    priority VARCHAR(20) DEFAULT 'medium',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    
    CONSTRAINT check_due_time CHECK (
        due_time IS NULL OR due_date IS NOT NULL
    )
);

-- Составной индекс для оптимизации запросов календаря
CREATE INDEX idx_tasks_user_due_date ON tasks(user_id, due_date);
CREATE INDEX idx_tasks_status ON tasks(status);

-- Индекс для полнотекстового поиска по названию (требует pg_trgm)
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_tasks_title_search ON tasks USING gin (to_tsvector('russian', title));
```

**Поля:**
- `id` — уникальный идентификатор задачи
- `user_id` — связь с пользователем (внешний ключ)
- `title` — название задачи
- `description` — подробное описание (опционально)
- `due_date` — дата выполнения задачи
- `due_time` — время выполнения (опционально)
- `status` — статус задачи: `pending`, `in_progress`, `completed`, `cancelled`
- `priority` — приоритет: `low`, `medium`, `high`, `urgent`
- `created_at` — дата создания задачи
- `updated_at` — дата последнего изменения
- `completed_at` — дата фактического завершения

### 1.2 Связи между сущностями

**Тип связи:** One-to-Many

Один пользователь может иметь множество задач. Каждая задача принадлежит только одному пользователю.

```
users (1) ────────< (N) tasks
```

**Обеспечение изоляции данных:**
- Каждый запрос к задачам фильтруется по `user_id` текущего пользователя
- При удалении пользователя все его задачи удаляются каскадно (`ON DELETE CASCADE`)
- Добавлен `CONSTRAINT` на уровне БД для валидации `due_time`

---

## 2. Архитектура проекта

### 2.1 Общая структура

Проект построен по архитектуре **REST API + SPA** (Single Page Application).

```
┌─────────────────┐         ┌─────────────────┐
│   Frontend      │ ◄─────► │    Backend      │
│   (React)       │   REST  │    (Django)     │
│                 │   API   │                 │
└─────────────────┘         └────────┬────────┘
                                     │
                            ┌────────▼────────┐
                            │   PostgreSQL    │
                            │    Database     │
                            └─────────────────┘
```

### 2.2 Backend (API)

**Технологии:**
- Фреймворк: Django 5.0 + Django REST Framework (DRF)
- База данных: PostgreSQL 15
- Аутентификация: JWT (JSON Web Tokens) через `djangorestframework-simplejwt`
- ORM: Django ORM
- Валидация: DRF Serializers + Model-level validation
- Документация API: `drf-spectacular` (OpenAPI/Swagger)
- Контейнеризация: Docker + Docker Compose

**Структура проекта:**

```
task_tracker_backend/
├── config/                 # Настройки проекта
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── apps/                   # Приложения
│   ├── users/              # Приложение для пользователей
│   │   ├── models.py       # Модель User
│   │   ├── serializers.py
│   │   ├── views.py        # API views для аутентификации
│   │   └── urls.py
│   └── tasks/              # Приложение для задач
│       ├── models.py       # Модель Task с валидацией
│       ├── serializers.py
│       ├── views.py        # API views для CRUD задач
│       ├── permissions.py
│       └── urls.py
├── Dockerfile              # Для деплоя
├── docker-compose.yml      # Для локальной разработки
├── requirements.txt
├── .env                    # Переменные окружения
└── manage.py
```

### 2.3 Frontend (SPA)

**Технологии:**
- Фреймворк: React 18 + TypeScript
- UI библиотека: Material-UI (MUI) или Ant Design
- Календарь: FullCalendar или React Big Calendar
- State management: React Context API + React Query (TanStack Query)
- HTTP клиент: Axios
- Роутинг: React Router v6

**Структура проекта:**

```
task_tracker_frontend/
├── src/
│   ├── components/         # Переиспользуемые компоненты
│   │   ├── Calendar/       # Компонент календаря
│   │   ├── TaskForm/       # Форма создания/редактирования
│   │   └── TaskList/       # Список задач
│   ├── pages/              # Страницы
│   │   ├── LoginPage.tsx
│   │   ├── RegisterPage.tsx
│   │   └── DashboardPage.tsx
│   ├── services/           # API клиент
│   │   └── api.ts          # Axios instance + endpoints
│   ├── hooks/              # React Query hooks
│   │   └── useTasks.ts     # Кастомные хуки для работы с API
│   ├── contexts/           # Контексты
│   │   └── AuthContext.tsx
│   ├── utils/              # Вспомогательные функции
│   │   └── tokenStorage.ts
│   └── App.tsx
├── Dockerfile
├── package.json
└── .env
```

---

## 3. Логика авторизации

### 3.1 Процесс регистрации

**Шаги:**

1. Пользователь отправляет `email` + `password`
   ```
   POST /api/auth/register/
   ```

2. Backend:
   - Валидирует данные (email формат, пароль >= 8 символов)
   - Проверяет уникальность email
   - Хэширует пароль (bcrypt)
   - Создаёт запись в БД
   - Возвращает success message

3. Frontend перенаправляет на страницу логина

**Пример запроса:**

```json
POST /api/auth/register/
{
    "email": "user@example.com",
    "password": "SecurePass123",
    "full_name": "Иван Иванов"
}
```

**Ответ:**

```json
{
    "id": 1,
    "email": "user@example.com",
    "full_name": "Иван Иванов",
    "message": "Регистрация успешна"
}
```

### 3.2 Процесс аутентификации (Login)

**Шаги:**

1. Пользователь отправляет `email` + `password`
   ```
   POST /api/auth/login/
   ```

2. Backend:
   - Проверяет существование пользователя
   - Сверяет хэш пароля
   - Генерирует пару токенов (access + refresh)
   - Возвращает токены + информацию о пользователе

3. Frontend:
   - Сохраняет `access_token` в памяти (state)
   - Сохраняет `refresh_token` в httpOnly cookies (рекомендуется для production)
   - Устанавливает `access_token` в заголовки Axios
   - Перенаправляет на Dashboard

**Пример запроса:**

```json
POST /api/auth/login/
{
    "email": "user@example.com",
    "password": "SecurePass123"
}
```

**Ответ:**

```json
{
    "access": "eyJ0eXAiOiJKV1QiLCJhbGc...",
    "refresh": "eyJ0eXAiOiJKV1QiLCJhbGc...",
    "user": {
        "id": 1,
        "email": "user@example.com",
        "full_name": "Иван Иванов"
    }
}
```

### 3.3 Защита эндпоинтов

**Backend (Django):**

```python
from rest_framework.permissions import IsAuthenticated

class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]
    
    def get_queryset(self):
        # Пользователь видит только свои задачи
        return Task.objects.filter(user=self.request.user)
```

**Frontend (Axios interceptor):**

```typescript
// Axios interceptor добавляет JWT в каждый запрос
axios.interceptors.request.use(config => {
    const token = getAccessToken(); // Из state, не из localStorage
    if (token) {
        config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
});
```

**Улучшенное хранение токенов (Production):**

```typescript
// Вместо localStorage - использовать state для access token
const [accessToken, setAccessToken] = useState<string | null>(null);

// Refresh token хранится в httpOnly cookies (настраивается на backend)
// Backend устанавливает cookie при логине:
// response.set_cookie('refresh_token', refresh_token, httponly=True, secure=True, samesite='Strict')
```

### 3.4 Обновление токенов (Refresh)

**Процесс:**

1. Access токен истёк → получаем `401 Unauthorized`
2. Frontend автоматически отправляет refresh токен
   ```
   POST /api/auth/token/refresh/
   ```
3. Backend возвращает новый access токен
4. Повторяем изначальный запрос с новым токеном

**Реализация:**

```typescript
// Axios interceptor для автоматического refresh
axios.interceptors.response.use(
    response => response,
    async error => {
        const originalRequest = error.config;
        
        if (error.response?.status === 401 && !originalRequest._retry) {
            originalRequest._retry = true;
            
            try {
                // Refresh token отправляется автоматически из httpOnly cookie
                const { data } = await axios.post('/api/auth/token/refresh/');
                setAccessToken(data.access);
                
                originalRequest.headers.Authorization = `Bearer ${data.access}`;
                return axios(originalRequest);
            } catch (refreshError) {
                // Редирект на логин
                redirectToLogin();
                return Promise.reject(refreshError);
            }
        }
        
        return Promise.reject(error);
    }
);
```

---

## 4. Логика работы с задачами

### 4.1 Модель Task с валидацией

```python
# apps/tasks/models.py
from django.db import models
from django.contrib.auth.models import User
from django.core.exceptions import ValidationError
from django.utils import timezone


class Task(models.Model):
    STATUS_CHOICES = [
        ('pending', 'Ожидает'),
        ('in_progress', 'В процессе'),
        ('completed', 'Завершена'),
        ('cancelled', 'Отменена'),
    ]
    
    PRIORITY_CHOICES = [
        ('low', 'Низкий'),
        ('medium', 'Средний'),
        ('high', 'Высокий'),
        ('urgent', 'Срочный'),
    ]
    
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='tasks')
    title = models.CharField(max_length=255)
    description = models.TextField(blank=True)
    due_date = models.DateField()
    due_time = models.TimeField(null=True, blank=True)
    status = models.CharField(max_length=50, choices=STATUS_CHOICES, default='pending')
    priority = models.CharField(max_length=20, choices=PRIORITY_CHOICES, default='medium')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    completed_at = models.DateTimeField(null=True, blank=True)
    
    class Meta:
        ordering = ['due_date', 'due_time']
        indexes = [
            models.Index(fields=['user', 'due_date'], name='idx_user_due_date'),
            models.Index(fields=['status'], name='idx_status'),
        ]
    
    def clean(self):
        """Валидация на уровне модели"""
        if self.due_time and not self.due_date:
            raise ValidationError({
                'due_time': 'Время выполнения не может быть установлено без даты.'
            })
    
    def save(self, *args, **kwargs):
        """Автоматическое управление completed_at"""
        self.full_clean()  # Запуск валидации
        
        # Автоматически проставляем completed_at
        if self.status == 'completed' and not self.completed_at:
            self.completed_at = timezone.now()
        elif self.status != 'completed':
            self.completed_at = None
            
        super().save(*args, **kwargs)
    
    def __str__(self):
        return f"{self.title} ({self.get_status_display()})"
```

### 4.2 Создание задачи

**Процесс:**

1. Пользователь заполняет форму:
   - Название задачи
   - Описание (опционально)
   - Дата и время
   - Приоритет

2. `POST /api/tasks/`  
   `Authorization: Bearer {access_token}`

3. Backend:
   - Валидирует данные
   - Автоматически привязывает task к `user_id` из токена
   - Создаёт запись в БД
   - Возвращает созданную задачу

4. Frontend:
   - Обновляет состояние (добавляет задачу в список)
   - Отображает задачу в календаре

**Пример запроса:**

```json
POST /api/tasks/
{
    "title": "Встреча с клиентом",
    "description": "Обсудить Q4 планы",
    "due_date": "2025-10-25",
    "due_time": "14:00:00",
    "priority": "high"
}
```

**Ответ:**

```json
{
    "id": 1,
    "title": "Встреча с клиентом",
    "description": "Обсудить Q4 планы",
    "due_date": "2025-10-25",
    "due_time": "14:00:00",
    "status": "pending",
    "priority": "high",
    "created_at": "2025-10-23T10:00:00Z",
    "updated_at": "2025-10-23T10:00:00Z",
    "completed_at": null
}
```

### 4.3 Получение задач пользователя

Backend автоматически фильтрует по `user_id` из токена.

**Примеры запросов с фильтрами:**

```
GET /api/tasks/?date=2025-10-25          # Задачи на конкретную дату
GET /api/tasks/?month=2025-10             # Задачи за месяц
GET /api/tasks/?status=pending            # По статусу
GET /api/tasks/?priority=high             # По приоритету
GET /api/tasks/?ordering=due_date         # Сортировка
GET /api/tasks/?search=встреча            # Поиск по названию
GET /api/tasks/?page=1&page_size=50       # Пагинация
```

**Ответ:**

```json
{
    "count": 2,
    "next": null,
    "previous": null,
    "results": [
        {
            "id": 1,
            "title": "Встреча с клиентом",
            "description": "Обсудить Q4 планы",
            "due_date": "2025-10-25",
            "due_time": "14:00:00",
            "status": "pending",
            "priority": "high",
            "created_at": "2025-10-23T10:00:00Z"
        },
        {
            "id": 2,
            "title": "Code review",
            "due_date": "2025-10-25",
            "due_time": null,
            "status": "pending",
            "priority": "medium"
        }
    ]
}
```

### 4.4 Обновление и удаление задач

```
PATCH /api/tasks/{id}/    # Частичное обновление
PUT /api/tasks/{id}/      # Полное обновление
DELETE /api/tasks/{id}/   # Удаление
```

Backend проверяет, что задача принадлежит текущему пользователю:

```python
def get_queryset(self):
    return Task.objects.filter(user=self.request.user)
```

Если пользователь пытается изменить чужую задачу → `404 Not Found`.

### 4.5 Отображение в календаре

**Frontend логика:**

1. При загрузке календаря запрашиваются задачи за месяц:
   ```typescript
   const { data: tasks } = useTasks({ month: '2025-10', page: 1, pageSize: 50 });
   ```

2. Задачи группируются по датам и отображаются в календаре

3. При клике на дату:
   - Показываются все задачи на эту дату
   - Есть возможность создать новую задачу на выбранную дату

4. При клике на задачу:
   - Открывается модальное окно с деталями
   - Можно редактировать или удалить

**React Query hook с пагинацией:**

```typescript
// hooks/useTasks.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

export const useTasks = (filters?: TaskFilters) => {
    const queryClient = useQueryClient();
    
    const { data, isLoading, error } = useQuery({
        queryKey: ['tasks', filters],
        queryFn: async () => {
            const response = await api.get('/tasks/', { params: filters });
            return response.data;
        },
        staleTime: 30000, // 30 секунд
    });
    
    const createMutation = useMutation({
        mutationFn: (task: Partial<Task>) => api.post('/tasks/', task),
        onSuccess: () => {
            queryClient.invalidateQueries({ queryKey: ['tasks'] });
        },
    });
    
    return {
        tasks: data?.results || [],
        count: data?.count || 0,
        isLoading,
        error,
        createTask: createMutation.mutate,
    };
};
```

---

## 5. Структура API

### 5.1 Эндпоинты аутентификации

| Метод | Endpoint | Описание |
|-------|----------|----------|
| POST | `/api/auth/register/` | Регистрация нового пользователя |
| POST | `/api/auth/login/` | Вход (получение JWT токенов) |
| POST | `/api/auth/token/refresh/` | Обновление access токена |
| POST | `/api/auth/logout/` | Выход (опционально, blacklist токена) |
| GET | `/api/auth/me/` | Получить информацию о текущем пользователе |

### 5.2 Эндпоинты задач

| Метод | Endpoint | Описание |
|-------|----------|----------|
| GET | `/api/tasks/` | Список всех задач пользователя |
| POST | `/api/tasks/` | Создать новую задачу |
| GET | `/api/tasks/{id}/` | Получить задачу по ID |
| PATCH | `/api/tasks/{id}/` | Обновить задачу |
| DELETE | `/api/tasks/{id}/` | Удалить задачу |
| PATCH | `/api/tasks/{id}/complete/` | Отметить задачу выполненной |

### 5.3 Фильтры и параметры

| Параметр | Тип | Описание | Пример |
|----------|-----|----------|--------|
| `date` | string | Фильтр по конкретной дате | `?date=2025-10-25` |
| `month` | string | Фильтр по месяцу | `?month=2025-10` |
| `status` | string | Фильтр по статусу | `?status=pending` |
| `priority` | string | Фильтр по приоритету | `?priority=high` |
| `search` | string | Поиск по названию | `?search=встреча` |
| `ordering` | string | Сортировка | `?ordering=due_date` |
| `page` | integer | Номер страницы | `?page=1` |
| `page_size` | integer | Размер страницы | `?page_size=50` |

---

## 6. Безопасность

### 6.1 Защита данных

1. **Хэширование паролей:** bcrypt (Django `make_password`)
2. **JWT токены:** короткое время жизни access token (15-30 мин)
3. **HTTPS:** обязательно для production
4. **CORS:** настроен только для фронтенд-домена
5. **Rate limiting:** ограничение количества запросов с одного IP
6. **Хранение токенов:**
   - Access token: в памяти (React state)
   - Refresh token: httpOnly cookies (защита от XSS)

### 6.2 Изоляция данных пользователей

- Все запросы к задачам фильтруются по `user_id` из JWT токена
- На уровне БД: внешний ключ `user_id NOT NULL`
- Попытка доступа к чужой задаче возвращает `404` (не `403`, чтобы не раскрывать существование)
- Cascading delete: при удалении пользователя все его задачи удаляются автоматически

### 6.3 Настройки Django для production

```python
# config/settings.py

# Security
SECRET_KEY = os.getenv('SECRET_KEY')
DEBUG = False
ALLOWED_HOSTS = os.getenv('ALLOWED_HOSTS', '').split(',')

# HTTPS
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'

# CORS
CORS_ALLOWED_ORIGINS = os.getenv('CORS_ALLOWED_ORIGINS', '').split(',')
CORS_ALLOW_CREDENTIALS = True

# JWT
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=30),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=1),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
}
```

---

## 7. Контейнеризация (Docker)

### 7.1 Backend Dockerfile

```dockerfile
FROM python:3.11-slim

ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

WORKDIR /app

# Установка системных зависимостей
RUN apt-get update && apt-get install -y \
    postgresql-client \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Копирование requirements и установка зависимостей
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Копирование кода приложения
COPY . .

# Создание непривилегированного пользователя
RUN useradd -m -u 1000 django && chown -R django:django /app
USER django

# Порт приложения
EXPOSE 8000

# Команда запуска
CMD ["sh", "-c", "python manage.py migrate && gunicorn config.wsgi:application --bind 0.0.0.0:8000 --workers 3"]
```

### 7.2 Docker Compose

```yaml
version: '3.8'

services:
  db:
    image: postgres:15-alpine
    container_name: tasktracker_db
    environment:
      POSTGRES_DB: tasktracker
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  backend:
    build: .
    container_name: tasktracker_backend
    command: sh -c "python manage.py migrate && python manage.py runserver 0.0.0.0:8000"
    volumes:
      - .:/app
      - static_volume:/app/staticfiles
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/tasktracker
      SECRET_KEY: dev-secret-key-change-in-production
      DEBUG: "True"
      ALLOWED_HOSTS: localhost,127.0.0.1
      CORS_ALLOWED_ORIGINS: http://localhost:3000

volumes:
  postgres_data:
  static_volume:
```

### 7.3 .dockerignore

```
__pycache__
*.pyc
*.pyo
*.pyd
.Python
env/
venv/
.venv
.env
db.sqlite3
*.log
.git
.gitignore
README.md
*.md
.DS_Store
node_modules/
```

### 7.4 Frontend Dockerfile

```dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine

COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

---

## 8. Технические детали деплоя

### 8.1 Backend

**Хостинг:** Railway, Render, или DigitalOcean App Platform

**База данных:** Managed PostgreSQL (с автоматическими бэкапами)

**Переменные окружения (.env):**

```bash
# Django Settings
SECRET_KEY=your-secret-key-here-change-in-production
DEBUG=False
ALLOWED_HOSTS=yourdomain.com,www.yourdomain.com

# Database
DATABASE_URL=postgresql://user:password@host:5432/database

# CORS
CORS_ALLOWED_ORIGINS=https://yourdomain.com,https://www.yourdomain.com

# JWT Settings
JWT_ACCESS_TOKEN_LIFETIME=30  # minutes
JWT_REFRESH_TOKEN_LIFETIME=1  # days
```

**Миграции:** автоматически через Django migrations

**Статика:** `collectstatic` + WhiteNoise или S3

**WSGI Server:** Gunicorn (не встроенный сервер Django)

```bash
# Установка
pip install gunicorn

# Запуск
gunicorn config.wsgi:application --bind 0.0.0.0:8000 --workers 3
```

### 8.2 Frontend

**Хостинг:** Vercel, Netlify, или Cloudflare Pages

**Сборка:** `npm run build` (создает оптимизированную production сборку)

**Переменные (.env.production):**

```bash
REACT_APP_API_URL=https://api.yourdomain.com
```

**CDN:** автоматически через хостинг

**SSL:** автоматически через хостинг

### 8.3 Checklist для production

- [ ] Изменить `SECRET_KEY`
- [ ] Установить `DEBUG=False`
- [ ] Настроить `ALLOWED_HOSTS`
- [ ] Настроить HTTPS
- [ ] Настроить CORS
- [ ] Настроить database backups
- [ ] Настроить мониторинг (Sentry)
- [ ] Настроить логирование
- [ ] Провести security audit
- [ ] Написать тесты

---

## 9. Возможные расширения

В будущем систему можно расширить:

1. **Повторяющиеся задачи:** поле `recurrence_rule` (daily, weekly, monthly)

2. **Теги/категории:** таблица `tags` с many-to-many связью

3. **Уведомления:** email/push за N минут до задачи (Celery + Redis)

4. **Совместные задачи:** таблица `task_shares` для расшаривания задач между пользователями

5. **Вложения:** таблица `attachments` с файлами к задачам (S3/CloudStorage)

6. **Подзадачи:** самосвязь в таблице tasks (`parent_task_id`)

7. **История изменений:** audit log для отслеживания изменений задач

8. **Полнотекстовый поиск:** PostgreSQL Full-Text Search или Elasticsearch

9. **Аналитика:** dashboard с метриками (выполненные задачи, продуктивность)

10. **Мобильное приложение:** React Native с тем же API

---

## 10. Requirements.txt

```txt
Django==5.0
djangorestframework==3.14.0
djangorestframework-simplejwt==5.3.0
django-cors-headers==4.3.0
django-filter==23.5
drf-spectacular==0.27.0
psycopg2-binary==2.9.9
gunicorn==21.2.0
python-decouple==3.8
whitenoise==6.6.0
```

## 11. Frontend package.json

```json
{
  "name": "task-tracker-frontend",
  "version": "1.0.0",
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.20.0",
    "@tanstack/react-query": "^5.12.0",
    "@tanstack/react-query-devtools": "^5.12.0",
    "@mui/material": "^5.14.20",
    "@mui/icons-material": "^5.14.19",
    "@emotion/react": "^11.11.1",
    "@emotion/styled": "^11.11.0",
    "axios": "^1.6.2",
    "react-big-calendar": "^1.8.5",
    "date-fns": "^2.30.0",
    "react-hot-toast": "^2.4.1"
  },
  "devDependencies": {
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "typescript": "^5.0.0",
    "vite": "^5.0.0"
  },
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  }
}
```

---

## Заключение

Данная архитектура обеспечивает:

- ✅ **Безопасность:** Надежное хранение и изоляцию данных пользователей, JWT аутентификацию с httpOnly cookies, многоуровневую валидацию
- ✅ **Производительность:** Оптимизированные индексы БД (составной индекс user_id + due_date), эффективное кэширование через React Query, пагинация
- ✅ **Масштабируемость:** Легко добавить новые функции благодаря модульной структуре, раздельная разработка frontend/backend
- ✅ **Удобство разработки:** Docker для консистентности окружения, современный tech stack с хорошей документацией, четкое разделение ответственности
- ✅ **Production-Ready:** Все рекомендации по безопасности применены, готовность к деплою, возможность CI/CD интеграции

**Время реализации MVP:** 2-3 недели для одного full-stack разработчика.

---

**Документ подготовлен:** Виктория Алексеенко | 23 октября 2025
