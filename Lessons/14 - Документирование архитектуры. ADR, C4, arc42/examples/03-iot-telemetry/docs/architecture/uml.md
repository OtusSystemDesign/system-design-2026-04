# UML — Платформа IoT-телеметрии

Три диаграммы: сквозной сценарий приёма и алертинга (sequence), жизненный цикл алерта (state machine) и физическое размещение (deployment).

## 1. Sequence — ingest → обработка → алерт

Показывает путь одной точки телеметрии от устройства до нотификации: публикация через edge gateway по MQTT, перекладка в Kafka, обработка в потоке (валидация, обогащение, проверка порога), запись в TSDB и доставка алерта. Зачем: видно, где данные становятся доступны на чтение и в какой момент рождается алерт — и что запись сырья и эмиссия алерта идут параллельно.

```mermaid
sequenceDiagram
    autonumber
    participant Dev as IoT-устройство
    participant GW as Edge Gateway
    participant MQ as MQTT Broker
    participant BR as Ingestion Bridge
    participant K as Kafka
    participant PR as Processing (Flink)
    participant TS as TimescaleDB
    participant AL as Alert Service
    participant NT as Notification провайдер

    Dev->>GW: измерение (локальный протокол)
    GW->>MQ: PUBLISH telemetry/site/dev (MQTT/TLS, QoS1)
    MQ-->>GW: PUBACK
    MQ->>BR: доставка сообщения
    BR->>K: produce (партиция по deviceId)
    PR->>K: consume (checkpoint offset)
    PR->>PR: валидация + обогащение (кэш порогов)
    par Запись сырья
        PR->>TS: batch insert точки/агрегаты
    and Проверка правил
        PR->>PR: Rule Evaluator: значение > порога?
        alt Порог превышен
            PR->>AL: alert event (Kafka)
            AL->>AL: дедупликация / эскалация
            AL->>NT: отправка оповещения (HTTPS)
            NT-->>AL: accepted
        else В норме
            PR-->>PR: алерт не создаётся
        end
    end
```

## 2. State machine — жизненный цикл алерта

Показывает состояния алерта от обнаружения до закрытия с дедупликацией и эскалацией. Зачем: алертинг — не «однократное событие», а конечный автомат; это объясняет, почему `Alert Service` хранит состояние (см. [ADR-0004](../adr/0004-alerting-rules.md)) и как гасятся «дребезг» и повторные срабатывания.

```mermaid
stateDiagram-v2
    [*] --> OK: правило активно, значения в норме

    OK --> Pending: порог превышен (одно измерение)
    Pending --> OK: вернулось в норму до for-окна (антидребезг)
    Pending --> Firing: условие держится дольше for-окна

    Firing --> Firing: повтор того же алерта (дедупликация по ключу)
    Firing --> Escalated: не подтверждён за SLA → эскалация
    Escalated --> Acknowledged: оператор подтвердил
    Firing --> Acknowledged: оператор подтвердил

    Acknowledged --> Resolved: значения вернулись в норму
    Firing --> Resolved: значения вернулись в норму (auto-resolve)
    Escalated --> Resolved: значения вернулись в норму

    Resolved --> [*]: закрыт, тикет обновлён

    note right of Pending
        for-окно (напр. 60 с) отсекает
        кратковременные выбросы
    end note
    note right of Firing
        дедупликация по (deviceId, ruleId):
        повтор не плодит новые нотификации
    end note
```

## 3. Deployment — edge → cloud

Показывает физическое размещение: edge-зона на объекте (gateway + локальные датчики и буфер) и облачная зона (ingestion, processing, storage, alerting). Зачем: видно границу edge/cloud, какие артефакты где крутятся и какой канал между зонами — это обосновывает буферизацию на gateway и MQTT поверх ненадёжного канала (см. [ADR-0001](../adr/0001-mqtt-edge-gateway.md)).

```mermaid
flowchart TB
    subgraph EDGE["🏭 Edge-зона (объект: фабрика / подстанция)"]
        direction TB
        SENS["Датчики<br/>(temp, vibration, power)"]
        GWNODE["Edge Gateway<br/><i>node: industrial PC</i>"]
        GWBUF[("Локальный буфер<br/><i>store-and-forward</i>")]
        SENS -->|Modbus / OPC-UA| GWNODE
        GWNODE --- GWBUF
    end

    subgraph CLOUD["☁️ Cloud-зона (Kubernetes-кластер)"]
        direction TB
        subgraph INGEST["Ingestion"]
            BROKER["MQTT Broker<br/><i>pod: EMQX (3 реплики)</i>"]
            BRIDGE["Ingestion Bridge<br/><i>pod: Go</i>"]
            KAFKA[["Kafka<br/><i>statefulset: 3 брокера</i>"]]
        end
        subgraph PROC["Processing"]
            FLINK["Processing Service<br/><i>Flink JobManager + N TaskManager</i>"]
        end
        subgraph STORE["Storage"]
            TSDB[("TimescaleDB<br/><i>primary + replica</i>")]
            META[("Metadata DB<br/><i>PostgreSQL</i>")]
        end
        subgraph SERVE["Serving & Alerting"]
            ALERT["Alert/Notification Service<br/><i>pod: Go</i>"]
            QAPI["Query API<br/><i>pod: Go</i>"]
            DEVSVC["Device Registry<br/><i>pod: Go</i>"]
        end
    end

    EXT_NT["Notification провайдер"]
    EXT_INC["Incident / CRM"]

    GWNODE -->|MQTT/TLS, QoS1| BROKER
    BROKER --> BRIDGE --> KAFKA --> FLINK
    FLINK -->|batch insert| TSDB
    FLINK -->|gRPC: пороги| DEVSVC
    FLINK -->|alert events| ALERT
    DEVSVC --- META
    ALERT --- META
    ALERT -->|HTTPS| EXT_NT
    ALERT -->|HTTPS| EXT_INC
    QAPI --> TSDB
    QAPI --> META
```
