University: ITMO University

Faculty: FICT

Course: Cloud-platforms

Year: 2025/2026

Group: U4225

Author: KOROBKOVA EKATERINA ANDREEVNA

Lab: Lab4

Date of create: 30.11.2025

Date of finished: 01.12.2025

Приложение - чат-бот с RAG (Retrieval-Augmented Generation), отвечающий пользователям на основе базы знаний компании.

**Компоненты RAG, влияющие на инфраструктуру**
- Vector Store (хранилище эмбеддингов) — Pinecone / Vertex AI Vector Search / Elasticsearch.
- Embeddings API — получение эмбеддингов для документов и пользовательских запросов.
- Документное хранилище — Cloud Storage + метаданные в Firestore / Cloud SQL.
- ETL-пайплайн — загрузка новых документов, их разбиение (chunking), расчёт эмбеддингов.
- RAG pipeline: Пользовательский запрос → Embedding → Vector Search → Retrieve Top-K → Prompt Assembly → LLM ответ.

**1. Метрики**

Количество запросов:

- MVP: 20 000 / мес
- Pilot: 200 000 / мес
- Prod: 2 000 000 / мес

Объём корпоративных документов:

- MVP: 500 документов (~50 MB)
- Pilot: 5 000 документов (~500 MB)
- Prod: 50 000 документов (~5 GB)
  
Размер векторного индекса:

- MVP: ~100k embeddings
- Pilot: ~1M embeddings
- Prod: ~10M embeddings
  
**2. Компоненты архитектуры RAG (Google Cloud)**

**Компоненты RAG, влияющие на инфраструктуру**
- Vector Store (хранилище эмбеддингов) — Pinecone / Vertex AI Vector Search / Elasticsearch.
- Embeddings API — получение эмбеддингов для документов и пользовательских запросов.
- Документное хранилище — Cloud Storage + метаданные в Firestore / Cloud SQL.
- ETL-пайплайн — загрузка новых документов, их разбиение (chunking), расчёт эмбеддингов.
- RAG pipeline: Пользовательский запрос → Embedding → Vector Search → Retrieve Top-K → Prompt Assembly → LLM ответ.
   
**Схема компонентов RAG в GCP**

- Cloud CDN - отдача фронта
- Cloud Load Balancer — распределение запросов
- Cloud Run / GKE Autopilot — backend чат-бота
- Identity Platform — авторизация

**RAG-слой**

- Vertex AI Vector Search — векторный поиск
- Vertex AI Embeddings API — генерация эмбеддингов
- Vertex AI Generative AI / Gemini API — генерация ответа (LLM)
- Cloud Storage — хранение корпоративных документов
- Firestore / Cloud SQL — метаданные документов, версии, статусы
- Cloud Functions / Cloud Run jobs — ETL пайплайн: загрузка → разбиение → эмбеддинги → индексация

**Кеширование и ускорение**
- Memorystore (Redis) — кеш ответов / кеш retrieved context

**Мониторинг**
- Cloud Monitoring + Cloud Logging

**3. Архитектурные решения по стадиям и обоснование**

3.1 Начальная (MVP)

Цель: минимум усилий, минимальные затраты, быстрый запуск.

Серверless / контейнеры с минимальным развёртыванием (Lambda или ECS Fargate / small EC2)

Использовать внешние LLM-API (OpenAI/Google) чтобы избежать капитальных затрат на GPU

S3 для хранения файлов

Managed NoSQL/Serverless DB (DynamoDB / Firebase) для упрощения эксплуатации

CDN необязательно, но можно подключить при наличии статики

**MVP диаграмма**

ML-блок называется: “Inference: External API”

Это означает:
- нет своего ML-хостинга
- вызовы отправляются в сторонний/внешний API (OpenAI/Anthropic и т.п.)
- минимальная стоимость
- минимальная инфраструктура

![alt text](screenshots/1.png)

Обоснование: нет смысла держать GPU-инфраструктуру при низкой нагрузке; управление сервисом проще и быстрее.

3.2 Тестирование партнерами (pilot)

Цель: стабильность, мониторинг, начать оптимизировать расходы и латентность.

Доработка:
- Перенос backend в контейнерный кластер (ECS/EKS) с автоскейлингом
- Хранение модели/векторные индексы в managed inference (SageMaker/Vertex) или edge-caching
- Добавление Redis-кеш для уменьшения вызовов LLM-API
- Полноценный мониторинг и инструменты CI/CD.

  
Обоснование: рост нагрузки делает выгодным контейнерный подход и кеширование; позволяется постепенно оплачивать GPU только для тестов.
3.3 Прод
Цель: отказоустойчивость, масштабирование, контроль задержки и стоимости на большом трафике. Рекомендация:
Многоуровневая архитектура: CDN, regional load balancers, autoscaling backend
Выделенные GPU-инстансы в кластере для онлайн-инференса (или managed inference endpoints) + очереди/батчи
Горизонтальное масштабирование DB, multi-AZ/region репликация
Включить кеши, rate-limits, оптимизацию вызовов LLM (prompt engineering, caching, shorter outputs)
Оптимизации: spot/RTX instances для batch, reserved instances или saved commitments для predictable load
Обоснование: при больших объёмах API-вызывов использование собственного инференса (или выгодных контрактов с провайдерами) экономически оправдано и уменьшает задержки.
