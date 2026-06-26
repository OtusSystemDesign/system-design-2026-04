# UML — Маркетплейс

Три диаграммы, дополняющие C4: динамика checkout-saga (sequence), жизненный цикл заказа (state machine) и бизнес-процесс оформления (activity). Дублирующих диаграмм нет — каждая отвечает на свой вопрос.

## 1. Sequence — Checkout-saga (оркестрация)

**Что показывает:** как Order Service оркеструет распределённую транзакцию оформления заказа: оплата → резерв стока → отгрузка, и как выполняются **компенсации** при сбое шага. Главное — порядок шагов и откат уже выполненных действий, чтобы деньги и сток остались согласованы.

```mermaid
sequenceDiagram
    autonumber
    participant C as Покупатель
    participant O as Order Service (оркестратор)
    participant P as Payment Service
    participant I as Inventory Service
    participant S as Shipping Service

    C->>O: Оформить заказ (cartId)
    O->>O: Создать заказ (status=CREATED), записать saga

    O->>P: Списать оплату (orderId, amount)
    alt Оплата отклонена
        P-->>O: PaymentFailed
        O->>O: Заказ CANCELLED
        O-->>C: Отказ: оплата не прошла
    else Оплата успешна
        P-->>O: PaymentCaptured (status=PAID)

        O->>I: Зарезервировать сток (orderId, items)
        alt Стока не хватает
            I-->>O: ReservationFailed
            Note over O,P: Компенсация
            O->>P: Возврат оплаты (refund)
            P-->>O: Refunded
            O->>O: Заказ CANCELLED
            O-->>C: Отказ: товара нет в наличии
        else Резерв успешен
            I-->>O: Reserved (status=RESERVED)

            O->>S: Создать отгрузку (orderId)
            alt Сбой отгрузки
                S-->>O: ShippingFailed
                Note over O,I: Компенсации в обратном порядке
                O->>I: Освободить резерв (release)
                I-->>O: Released
                O->>P: Возврат оплаты (refund)
                P-->>O: Refunded
                O->>O: Заказ CANCELLED
                O-->>C: Отказ: не удалось оформить доставку
            else Отгрузка создана
                S-->>O: Shipped (status=SHIPPED)
                O->>O: Заказ подтверждён
                O-->>C: Заказ оформлен
            end
        end
    end
```

## 2. State machine — Жизненный цикл заказа

**Что показывает:** допустимые статусы заказа и переходы между ними. Подчёркивает, что переходы строго направлены (нельзя «отгрузить» неоплаченный заказ), а отмена/возврат возможны только из определённых состояний.

```mermaid
stateDiagram-v2
    [*] --> Created: оформление начато
    Created --> Paid: оплата прошла
    Created --> Cancelled: оплата отклонена / отмена

    Paid --> Reserved: сток зарезервирован
    Paid --> Cancelled: резерв не удался (refund)

    Reserved --> Shipped: отгрузка создана
    Reserved --> Cancelled: сбой отгрузки (release + refund)

    Shipped --> Delivered: доставлено покупателю
    Shipped --> Refunded: возврат после отгрузки

    Delivered --> Refunded: возврат после получения

    Cancelled --> [*]
    Delivered --> [*]
    Refunded --> [*]
```

## 3. Activity — Оформление заказа (от корзины до подтверждения)

**Что показывает:** бизнес-процесс с точками принятия решений: проверка наличия стока и результата оплаты. Видны ветви отказа и компенсаций, которые в sequence показаны как сообщения, а здесь — как шаги процесса.

```mermaid
flowchart TD
    A([Покупатель: оформить корзину]) --> B[Создать заказ CREATED]
    B --> C[Списать оплату]
    C --> D{Оплата прошла?}
    D -- Нет --> E[Заказ CANCELLED]
    E --> Z([Конец: отказ])
    D -- Да --> F[Заказ PAID]
    F --> G[Резервировать сток]
    G --> H{Сток есть?}
    H -- Нет --> I[Компенсация: refund оплаты]
    I --> E
    H -- Да --> J[Заказ RESERVED]
    J --> K[Создать отгрузку]
    K --> L{Отгрузка создана?}
    L -- Нет --> M[Компенсация: release резерва + refund]
    M --> E
    L -- Да --> N[Заказ SHIPPED]
    N --> O[Уведомить покупателя и продавца]
    O --> P([Конец: заказ подтверждён])
```
