# 5. Интерфейсы (ICD-lite)

Шесть контрактов: четыре REST-эндпоинта и два события Apache Kafka [1]. Поля, ошибки, версии описаны ниже. Для производственной версии документа рекомендуется выгрузить контракты в OpenAPI/Protobuf.

## 5.1. REST: POST /api/v1/receivings/{receivingId}/items — фиксация принятой позиции

**Назначение.** Кладовщик подтверждает приёмку конкретной позиции с фактическим количеством. Покрывает SR-SYS-9, SR-SYS-10.

**Request** (mobile client → API Gateway → Receiving Module).

| Поле | Тип | Обяз. | Описание |
| --- | --- | --- | --- |
| operationId | uuid | да | Идемпотентный ключ операции, сгенерирован на клиенте |
| productId | uuid | да | ID товара из справочника |
| expectedQty | integer | да | Ожидаемое количество из задания (отправляется обратно для проверки) |
| actualQty | integer | да | Фактическое количество |
| barcode | string | нет | Отсканированный штрихкод (для аудита) |
| userId | uuid | да | ID кладовщика |
| scannedAt | datetime | да | Время сканирования на клиенте (ISO-8601, UTC) |

**Response 200 OK.**

| Поле | Тип | Описание |
| --- | --- | --- |
| receivingItemId | uuid | ID записи приёмки |
| discrepancyDetected | boolean | true, если actual ≠ expected |
| requiresApproval | boolean | true, если нужно подтверждение супервайзера (SR-SYS-11) |
| nextAction | enum | await_supervisor / await_label_print / ready_for_putaway |

**Ошибки.**

| HTTP | Код | Когда |
| --- | --- | --- |
| 400 | INVALID_PAYLOAD | actualQty < 0 или productId не в задании |
| 404 | RECEIVING_NOT_FOUND | receivingId не существует |
| 409 | ALREADY_PROCESSED | operationId уже обработан (повтор → старый ответ) |
| 422 | PRODUCT_NOT_IN_SHIPMENT | товар не входит в ожидаемую поставку |
| 503 | OMS_UNAVAILABLE | OMS/ERP недоступна; запись принята локально |

**Версионирование.** Префикс /api/v1/. Несовместимые изменения — v2. Совместимые поля добавляются как optional.

## 5.2. REST: POST /api/v1/pickings/{pickingId}/scan — сканирование при сборке

**Назначение.** Валидация сканирования при сборке заказа. Покрывает SR-SYS-20, SR-SYS-21, SR-SYS-22.

**Request.**

| Поле | Тип | Обяз. | Описание |
| --- | --- | --- | --- |
| operationId | uuid | да | Идемпотентный ключ |
| scanType | enum | да | cell / product |
| code | string | да | Отсканированный код |
| expectedCellId | uuid | нет | Какая ячейка ожидается на текущем шаге |
| expectedProductId | uuid | нет | Какой товар ожидается |
| quantity | integer | нет | Количество (для scanType=product) |

**Response 200 OK.**

| Поле | Тип | Описание |
| --- | --- | --- |
| accepted | boolean | Принято ли сканирование |
| mismatchReason | string | wrong_cell / wrong_product / not_in_order |
| pickingProgress | object | completedItems, totalItems, currentStep |
| nextHint | string | Подсказка: следующая ячейка, следующий товар |

**Ошибки.**

| HTTP | Код | Когда |
| --- | --- | --- |
| 400 | INVALID_PAYLOAD | Отсутствует обязательное поле |
| 403 | FORBIDDEN | Пользователь не назначен на это задание |
| 404 | PICKING_NOT_FOUND | pickingId не существует |
| 409 | ALREADY_COMPLETED | Задание уже завершено |

## 5.3. REST (внешний): POST /shipments — регистрация в курьерской службе

**Назначение.** Контракт WMS → курьерская служба. Покрывает SR-SYS-27, SR-SYS-28. Courier Adapter Module нормализует ответы разных курьеров под единый внутренний формат.

**Request** (Courier Adapter → Carrier API).

| Поле | Тип | Обяз. | Описание |
| --- | --- | --- | --- |
| externalOrderId | string | да | ID заказа в WMS (для трассировки) |
| idempotencyKey | string | да | Защита от повторной регистрации |
| sender | object | да | Адрес склада-отправителя |
| recipient | object | да | Адрес получателя (имя, телефон, адрес, индекс) |
| package | object | да | weightKg, dimensionsCm, declaredValue |
| serviceCode | string | да | Код тарифа курьера |

**Response 201 Created.**

| Поле | Тип | Описание |
| --- | --- | --- |
| trackNumber | string | Трек-номер |
| labelUrl | string | URL для скачивания транспортной этикетки (PDF/ZPL) |
| estimatedDeliveryAt | datetime | Ожидаемая дата доставки |

**Ошибки (нормализуются в Courier Adapter).**

| Внешний HTTP | Внутренний код | Действие WMS |
| --- | --- | --- |
| 400 | INVALID_SHIPMENT | Заказ возвращается на проверку, уведомление менеджеру логистики |
| 401/403 | AUTH_FAILED | Уведомление администратору, заказ ставится в очередь регистрации |
| 429 | RATE_LIMITED | Экспоненциальный экспоненциальная задержка повторных попыток до 1 ч |
| 5xx, timeout | CARRIER_UNAVAILABLE | Очередь регистрации, статус «Ожидает регистрации доставки» (R-06) |

## 5.4. REST: GET /api/v1/support/orders/{orderId} — поиск по заказу (поддержка)

**Назначение.** Read-only доступ для службы поддержки. Покрывает SR-SH-8.

**Response 200 OK.**

| Поле | Тип | Описание |
| --- | --- | --- |
| orderId | uuid | ID заказа |
| externalOrderId | string | ID в OMS/ERP |
| currentStatus | enum | received, picking, picked, partial, supervisor_review, packing, awaiting_delivery_registration, awaiting_label, ready_to_ship, handed_to_carrier, cancelled |
| trackNumber | string | Трек-номер (если есть) |
| carrier | string | Курьерская служба |
| timeline | array | Список событий: ts, event, byUser, details |
| partialReasons | array | Если статус partial: причины недостачи |
| incidents | array | Связанные инциденты (расхождения, ошибки) |

**Ошибки.**

| HTTP | Код | Когда |
| --- | --- | --- |
| 404 | ORDER_NOT_FOUND | Заказа нет ни по WMS-id, ни по OMS-id |
| 403 | FORBIDDEN | Пользователь не имеет роли support |

## 5.5. Kafka-событие: wms.stock.updated

**Назначение.** Уведомить подписчиков (Sync Module → OMS/ERP, Reporting, Notification) об изменении остатка.

**Заголовки.**

| Заголовок | Описание |
| --- | --- |
| eventId | UUID, идемпотентность |
| eventVersion | Например, 1 (целое число) |
| correlationId | ID операции, инициировавшей изменение |
| producedAt | Timestamp UTC |

**Payload (JSON).**

| Поле | Тип | Описание |
| --- | --- | --- |
| productId | uuid | Товар |
| cellId | uuid | Ячейка (опционально для агрегированных остатков) |
| qtyBefore | integer | Остаток до операции |
| qtyAfter | integer | Остаток после операции |
| operationType | enum | receive / putaway / pick / pack / inventory_adjust / damage_write_off |
| operationId | uuid | Ссылка на операцию-источник |
| userId | uuid | Кто инициировал |
| ts | datetime | Время изменения |

**Гарантии.**

1. **At-least-once-доставка** [1] — обеспечивается комбинацией Apache Kafka и шаблона Transactional Outbox [8].
2. **Идемпотентность** на стороне потребителей — по eventId.
3. **Партиционирование Kafka-топика** wms.stock.updated по productId — гарантирует сохранение порядка событий по одному товару в пределах одной партиции. Уточнение: партиционирование Kafka-топика выполняется на уровне шины событий и не совпадает с партиционированием таблицы остатков PostgreSQL по cell_id (см. ADR-4 в §4.2). Потребители Sync Module (OMS/ERP) и Reporting Module, которым требуется детерминированный порядок событий по ячейке (cellId), обязаны выполнять собственную упорядочивающую группировку (например, по паре (productId, cellId) с буферизацией и сортировкой по полю ts); соответствующий инвариант проверяется в TC-04 (см. §7.3).

**Эволюция схемы.** Confluent Schema Registry [9] в режиме обратной совместимости (BACKWARD): новые поля — необязательные, со значением по умолчанию.

## 5.6. Kafka-событие: wms.order.status_changed

**Назначение.** Уведомление о смене статуса заказа. Потребители: OMS/ERP Sync, Notification, Reporting.

**Payload (JSON).**

| Поле | Тип | Описание |
| --- | --- | --- |
| orderId | uuid | ID заказа в WMS |
| externalOrderId | string | ID в OMS/ERP |
| previousStatus | enum | Предыдущий статус |
| newStatus | enum | Новый статус |
| transitionReason | string | Опциональная причина (partial:not_enough_stock) |
| trackNumber | string | Если статус ready_to_ship или shipped — трек-номер |
| userId | uuid | Инициатор перехода (может быть system) |
| ts | datetime | Время перехода |

**Допустимые переходы статусов.** 
received → picking;
picking → picked;
picking → partial;
partial → supervisor_review;
supervisor_review → picking;
supervisor_review → cancelled;
supervisor_review → awaiting_delivery_registration;
picked → packing;
packing → awaiting_delivery_registration;
awaiting_delivery_registration → awaiting_label;
awaiting_label → ready_to_ship;
ready_to_ship → handed_to_carrier;
любой незавершённый статус → cancelled при получении отмены из OMS/ERP.

Переход partial → supervisor_review обязателен, если заказ не может быть автоматически продолжен из-за недостачи, повреждения товара или расхождения остатков.
