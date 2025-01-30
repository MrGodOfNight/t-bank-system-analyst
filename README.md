# t-bank-system-analyst

---

OpenApi:
```yaml:
openapi: 3.0.0
info:
  title: Payment Block Service API
  version: 1.0.0
  description: API для управления блокировками платежей клиентов

paths:
  /api/v1/blocks/{clientId}:
    get:
      summary: Проверить статус блокировки клиента
      parameters:
        - name: clientId
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Блокировка активна
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BlockResponse'
        '404':
          description: Активная блокировка не найдена

    delete:
      summary: Разблокировать клиента
      parameters:
        - name: clientId
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Успешная разблокировка
        '404':
          description: Активная блокировка не найдена

  /api/v1/blocks:
    post:
      summary: Создать новую блокировку
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateBlockRequest'
      responses:
        '201':
          description: Блокировка создана
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BlockResponse'
        '409':
          description: Активная блокировка уже существует

components:
  schemas:
    CreateBlockRequest:
      type: object
      properties:
        clientId:
          type: string
          description: Идентификатор клиента
        blockType:
          type: string
          enum: [FRAUD, INCORRECT_DETAILS]
          description: Тип блокировки
        createdBy:
          type: string
          description: Идентификатор сотрудника
        comment:
          type: string
          nullable: true
      required:
        - clientId
        - blockType
        - createdBy

    BlockResponse:
      type: object
      properties:
        clientId:
          type: string
        blockType:
          type: string
        createdAt:
          type: string
          format: date-time
        createdBy:
          type: string
        comment:
          type: string
          nullable: true
```
---

## 2. Структура хранения данных в БД

Для хранения информации о блокировках платежей предлагается следующая структура таблиц:

### Таблица `PaymentBlocks`
| Поле          | Тип данных       | Описание                                                                 |
|---------------|------------------|-------------------------------------------------------------------------|
| `id`          | UUID             | Уникальный идентификатор блокировки                                      |
| `client_id`   | VARCHAR(255)     | Идентификатор клиента                                                   |
| `reason`      | TEXT             | Причина блокировки (например, "Подозрение на мошенничество")             |
| `block_type`  | ENUM('FRAUD_SUSPICION', 'INCORRECT_DETAILS') | Тип блокировки (мошенник или добропорядочный клиент) |
| `created_at`  | TIMESTAMP        | Дата и время создания блокировки                                         |
| `is_active`   | BOOLEAN          | Флаг активности блокировки (true — активна, false — разблокирована)      |

### Таблица `Clients`
| Поле          | Тип данных       | Описание                                                                 |
|---------------|------------------|-------------------------------------------------------------------------|
| `id`          | UUID             | Уникальный идентификатор клиента                                         |
| `name`        | VARCHAR(255)     | Наименование клиента                                                    |
| `status`      | ENUM('ACTIVE', 'BLOCKED') | Статус клиента (активен или заблокирован)               |
| `created_at`  | TIMESTAMP        | Дата и время регистрации клиента                                         |

---

## 3. Описание логики работы

- **Блокировка клиента**: 
  - При вызове `/payment-blocks` с методом `POST` создается запись в таблице `PaymentBlocks` с указанием типа блокировки (`block_type`) и причины (`reason`). Поле `is_active` устанавливается в `true`.
  - Если клиент уже заблокирован, система может вернуть ошибку или обновить существующую запись.

- **Разблокировка клиента**:
  - При вызове `/payment-blocks/{blockId}` с методом `DELETE` поле `is_active` в таблице `PaymentBlocks` устанавливается в `false`.

- **Проверка статуса блокировки**:
  - При вызове `/payment-blocks` с методом `GET` система проверяет наличие активных блокировок для указанного `clientId`. Если блокировка существует, возвращается информация о ней.

- **Отличие типов блокировок**:
  - Тип блокировки (`block_type`) позволяет различать случаи мошенничества (`FRAUD_SUSPICION`) и ошибок в реквизитах (`INCORRECT_DETAILS`).

---

## 4. Примеры использования

- **Создание блокировки**:
  ```json
  POST /payment-blocks
  {
    "clientId": "12345",
    "reason": "Подозрение на мошенничество",
    "blockType": "FRAUD_SUSPICION"
  }
  ```

- **Проверка статуса блокировки**:
  ```
  GET /payment-blocks/{clientId}
  ```

- **Разблокировка клиента**:
  ```
  DELETE /payment-blocks/{blockId}
  ```
