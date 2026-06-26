# UML — Онлайн-кинотеатр

Три диаграммы, дополняющие C4: динамика старта воспроизведения (sequence), физическое размещение по зонам (deployment) и жизненный цикл задания транскодинга (state machine).

---

## 1. Sequence — старт воспроизведения через CDN с выбором битрейта (ABR)

Что показывает: путь от нажатия Play до первого кадра. Плеер получает манифест с ABR-лесенкой, скачивает сегменты с CDN edge (cache hit/miss с тягой из origin) и адаптирует битрейт по измеренной пропускной способности. Зачем: видно, что тяжёлый трафик идёт мимо прикладных сервисов, а Playback Service лишь авторизует и собирает манифест.

```mermaid
sequenceDiagram
    actor User as Зритель
    participant Player as Клиентский плеер
    participant API as API Gateway
    participant PB as Playback Service
    participant DRM as DRM-провайдер
    participant Edge as CDN Edge (PoP)
    participant Origin as Origin / Object Storage

    User->>Player: Нажимает Play (titleId)
    Player->>API: GET /playback/{titleId} (+ токен)
    API->>PB: StartPlayback(titleId, userId)
    PB->>PB: Проверка entitlements (подписка, регион)
    PB-->>API: Манифест (HLS/DASH) + подписанные URL + DRM-init
    API-->>Player: Манифест с ABR-лесенкой (240p…4K)

    Player->>DRM: Запрос лицензии (DRM-init)
    DRM-->>Player: Лицензия (ключи расшифровки)

    Note over Player: Старт с консервативного битрейта
    Player->>Edge: GET сегмент #1 (720p)
    Edge->>Origin: cache miss → pull сегмента
    Origin-->>Edge: Сегмент #1
    Edge-->>Player: Сегмент #1 (+ кэширует)

    Note over Player: Измерил пропускную способность → выше
    Player->>Edge: GET сегмент #2 (1080p)
    Edge-->>Player: Сегмент #2 (cache hit)

    Note over Player: Сеть просела → ABR вниз
    Player->>Edge: GET сегмент #3 (480p)
    Edge-->>Player: Сегмент #3 (cache hit)

    Player->>API: Heartbeat / QoE (битрейт, ребуферинг)
```

---

## 2. Deployment — edge → origin → transcoding farm → storage

Что показывает: физическое размещение по зонам. Edge-узлы (CDN PoP) стоят близко к зрителю и кэшируют сегменты; региональный origin отдаёт промахи кэша из object storage; в core-зоне работают прикладные сервисы и асинхронная transcoding farm. Зачем: видно, где кэшируется трафик и почему origin видит <5% запросов.

```mermaid
flowchart TB
    subgraph EdgeZone["Edge-зона — CDN PoP (близко к зрителю)"]
        edge1["Edge cache PoP-1<br/>артефакт: видео-сегменты (hot)"]
        edge2["Edge cache PoP-N<br/>артефакт: видео-сегменты (hot)"]
    end

    subgraph RegionZone["Региональная зона — Origin"]
        origin["Origin Shield<br/>артефакт: манифесты + сегменты (warm)"]
        objstore[("Object Storage<br/>артефакт: мастер-файлы + все сегменты")]
    end

    subgraph CoreZone["Core-зона — прикладные сервисы (k8s)"]
        gw["API Gateway / BFF"]
        pb["Playback Service<br/>+ Redis (манифесты, entitlements)"]
        cat["Catalog Service<br/>+ PostgreSQL"]
        reco["Recommendation Service<br/>+ Reco Store"]
    end

    subgraph PipelineZone["Pipeline-зона — медиа-пайплайн (async)"]
        ingest["Ingest / Orchestrator"]
        queue[["Job Queue (Kafka/SQS)<br/>артефакт: задания транскодинга"]]
        farm["Transcoding Farm<br/>GPU/CPU воркеры (FFmpeg)"]
    end

    client["Клиентское устройство<br/>(Smart TV / mobile / web)"]

    client -->|"GET сегменты (HTTPS)"| edge1
    client -->|"GET сегменты (HTTPS)"| edge2
    client -->|"REST: манифест, каталог"| gw

    edge1 -->|"cache miss → pull"| origin
    edge2 -->|"cache miss → pull"| origin
    origin -->|"читает сегменты"| objstore

    gw --> pb
    gw --> cat
    gw --> reco
    pb -->|"подписанные URL"| objstore

    ingest -->|"кладёт задание"| queue
    queue -->|"забирает задание"| farm
    farm -->|"читает мастер / пишет сегменты"| objstore
    farm -->|"публикует тайтл"| cat
```

---

## 3. State machine — жизненный цикл задания транскодинга

Что показывает: статусы одного задания от загрузки мастер-файла до публикации, с ветками сбоев и ретраев. Зачем: это контракт медиа-пайплайна — на нём строятся мониторинг лага, алерты и идемпотентность воркеров (см. ADR-0002).

```mermaid
stateDiagram-v2
    [*] --> Uploaded: мастер-файл загружен в storage
    Uploaded --> Queued: оркестратор создал задание
    Queued --> Transcoding: воркер забрал задание

    Transcoding --> Packaging: рендишены готовы (ABR ladder)
    Transcoding --> Failed: ошибка транскодинга

    Packaging --> Published: HLS/DASH + DRM упакованы и опубликованы
    Packaging --> Failed: ошибка упаковки

    Failed --> Retry: попытка < N
    Failed --> DeadLetter: исчерпаны попытки (нужен ручной разбор)
    Retry --> Queued: повторная постановка в очередь

    Published --> [*]
    DeadLetter --> [*]

    note right of Transcoding
        Идемпотентность по jobId:
        повторная обработка не дублирует сегменты
    end note
    note right of Published
        Тайтл становится доступен в каталоге,
        CDN прогревается (warm-up)
    end note
```
