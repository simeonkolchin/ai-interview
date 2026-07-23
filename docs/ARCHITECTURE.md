# Архитектура

Как устроена платформа и почему именно так. Дополняет обзор в [README](../README.md) деталями жизненного цикла запросов, асинхронных пайплайнов и безопасности.

---

## Сервисы

| Сервис | Порт | Наружу | Роль |
|---|---|---|---|
| `agent-hr-front` | 8080 | через nginx | React-SPA: кабинет рекрутера и кандидатский сценарий |
| `agent-hr-gateway` | 8002 | да | Единая точка входа. Авторизация, CSRF, same-origin медиа/PDF |
| `data-service` | 8001 | нет | Ядро: сущности, бизнес-логика, S3, очереди, воркеры |
| `llm-service` | 8000 | нет | Stateless AI: генерация вопросов, оценка, распознавание речи |
| `gaze-service` | 8003 | нет | Детекция взгляда: batch-анализ видео и real-time landmarks |
| `postgres` · `rabbitmq` · `minio` | — | нет | Хранилище, очереди, объектное хранилище |

Все контейнеры — во внутренней Docker-сети `agent-hr-network`. Наружу открыты только gateway и фронтенд. Корневой репозиторий — meta-repo: инфраструктура и оркестрация в нём, а сами сервисы подключены как **git submodules**, каждый с собственной историей и CI.

## Жизненный цикл запроса

Браузерный запрос за результатом интервью проходит строго через gateway; data-service из браузера не виден.

```mermaid
sequenceDiagram
    autonumber
    participant B as 🌐 Браузер
    participant G as Gateway :8002
    participant D as Data Service :8001
    participant P as 🐘 PostgreSQL

    B->>G: GET /interviews/{id}<br/>Cookie: access_token
    G->>G: JWTAuthMiddleware — cookie-сессия
    G->>G: CSRF (для mutating) · security headers
    G->>D: проксирование + X-Internal-Service-Token
    D->>D: проверка tenant + владельца + visibility
    D->>P: SELECT интервью, вопросы, ответы, оценки
    P-->>D: строки
    D-->>G: JSON интервью
    G-->>B: ответ (+ X-Correlation-ID, traceparent)
```

Для автоматизированных клиентов вместо cookie используется персональный API-ключ: gateway разрешает ключ через data-service и выпускает **краткоживущий внутренний JWT**, чтобы downstream-роуты работали по единому контракту, не зная про API-ключи.

## Асинхронная обработка

Тяжёлое (LLM, речь, видео) уводится в фон через RabbitMQ. Data-service публикует задачи и владеет состоянием; LLM- и gaze-сервисы остаются вычислительными узлами без состояния.

```mermaid
flowchart TD
    subgraph SYNC["Синхронно (gateway)"]
        UP["Создание вакансии,<br/>загрузка материалов"]
        REL["Выдача интервью:<br/>копия готового question-set"]
    end

    subgraph QUEUES["RabbitMQ"]
        Q1(["question-set generation"])
        Q2(["транскрибация"])
        Q3(["оценка ответов"])
        Q4(["финализация"])
        Q5(["gaze-анализ"])
    end

    subgraph WORKERS["Воркеры data-service"]
        W1["vacancy-question-set-worker<br/><sub>/vacancy/extract → /screening/generate</sub>"]
        W2["transcription-worker<br/><sub>ffmpeg → /speech/recognize</sub>"]
        W3["answer-evaluation-worker<br/><sub>/screening/evaluate_answer</sub>"]
        W4["finalization-worker<br/><sub>/screening/finalize + пересчёт score</sub>"]
        W5["gaze-worker<br/><sub>batch /analyze/video</sub>"]
    end

    UP --> Q1 --> W1
    REL -.->|"после ready question-set"| Q2
    Q2 --> W2 --> Q3 --> W3 --> Q4 --> W4
    Q5 --> W5 --> Q4

    classDef s fill:#1a2340,stroke:#3b82f6,color:#e8eef6
    classDef q fill:#1d2333,stroke:#6366f1,color:#e8eef6
    classDef w fill:#16241f,stroke:#4ade80,color:#e8eef6
    class UP,REL s
    class Q1,Q2,Q3,Q4,Q5 q
    class W1,W2,W3,W4,W5 w
```

Ключевой инвариант: **question-set версионируется на уровне вакансии** и лишь копируется в интервью. Поэтому создание интервью — быстрая синхронная операция, которая проверяет наличие `ready` набора нужного размера, а не запускает генерацию по PDF в момент прохождения.

## Хранение файлов

Медиа не лежат в локальной ФС сервера. Все бинарные объекты — в S3-совместимом хранилище (AWS S3 или MinIO), data-service выступает слоем абстракции над бакетом.

```
s3://<bucket>/
  interviews/{interview_id}/answers/{answer_id}/{filename}
  interviews/{interview_id}/recordings/{upload_id}-{filename}
  vacancies/{vacancy_id}/materials/{material_id}/{filename}
```

- **Chunked upload** — большие записи грузятся по чанкам, собираются на стороне data-service, отправляются в S3, временная директория чистится.
- **Same-origin просмотр** — вместо прямого bucket-URL клиент получает `/media/{object_key}`, читаемый через gateway.
- **Очистка по префиксу** — при удалении интервью/вакансии связанные объекты удаляются по префиксам.

## Границы доверия

```mermaid
flowchart LR
    B["🌐 Браузер"] -->|"cookie access/refresh + CSRF"| G["🚪 Gateway"]
    A["🤖 API-клиент"] -->|"персональный API-ключ"| G
    G -->|"X-Internal-Service-Token"| D["🧮 Data Service"]
    W["Воркеры"] -->|"X-Internal-Service-Token"| D
    D --> P[("🐘 PostgreSQL")]
    B -. "нет прямого доступа" .-x D
    B -. "нет прямого доступа" .-x S["🗄️ S3"]

    classDef ext fill:#1b2a3a,stroke:#2CA5E0,color:#e8eef6
    classDef gw fill:#171a21,stroke:#3f4654,color:#e8eef6
    classDef core fill:#1a2340,stroke:#3b82f6,color:#e8eef6
    classDef db fill:#16241f,stroke:#4ade80,color:#e8eef6
    class B,A,W ext
    class G gw
    class D core
    class P,S db
```

| Секрет | Направление | Смысл |
|---|---|---|
| cookie `access_token` | браузер → gateway | «я вошедший пользователь этого tenant'а» |
| персональный API-ключ | клиент → gateway | «я автоматизированный клиент» (hash+prefix в БД) |
| `X-Internal-Service-Token` | gateway/воркеры → data-service `/internal/*` | «вызов уже авторизован во внешнем слое» |

Публичный кандидатский контур защищён отдельно: **OTP**-подтверждение отклика, rate-limiting по email/IP, защита от повторного отклика и повторного входа в уже начатое интервью. При старте сервисы делают **fail-fast** проверку секретов и не поднимаются в production с дефолтными значениями JWT/API-key/internal-token.

## Наблюдаемость

Каждый сервис публикует метрики Prometheus и трейсы OpenTelemetry. Сквозной `X-Correlation-ID` и `traceparent` протягиваются через gateway во все внутренние вызовы и воркеры, так что путь одного запроса — от браузера до фонового воркера оценки — восстанавливается по трейсу целиком.
