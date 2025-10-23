# Task Tracker - Архитектура проекта

Архитектурное решение для системы управления задачами с календарём.

## 📋 Описание

Проект представляет собой полноценную архитектуру веб-приложения Task Tracker, включающую:
- REST API (Django + PostgreSQL)
- Single Page Application (React + TypeScript)
- Систему авторизации (JWT)
- Календарь для управления задачами

## 📂 Документация

[**📖 Полная архитектура проекта**](task_tracker_architecture.md)

## 🛠 Технологии

### Backend
- Django 5.0 + Django REST Framework
- PostgreSQL 15
- JWT аутентификация
- Docker

### Frontend
- React 18 + TypeScript
- Material-UI
- React Query (TanStack Query)
- React Big Calendar

## 🚀 Ключевые особенности

- ✅ Оптимизированные индексы БД (составной индекс user_id + due_date)
- ✅ Валидация на уровне модели Django
- ✅ React Query для эффективного управления состоянием
- ✅ HttpOnly cookies для безопасности
- ✅ Docker для простого деплоя
- ✅ Production-ready настройки

## 📊 Структура данных

- **users** - таблица пользователей с JWT аутентификацией
- **tasks** - таблица задач с привязкой к дате и времени

## ⏱ Время реализации

MVP: 2-3 недели для одного full-stack разработчика

---

**Автор:** Виктория Алексеенко
**Дата:** 23 октября 2025
