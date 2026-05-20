# NetVision

## Внешние системы: DTO и endpoint

### Соглашения API

- Базовый путь API: `/api/external-systems`.
- Идентификатор внешней системы передается в формате `UUID`.
- Даты и время в API передаются в формате ISO 8601 с часовым поясом.
- На UI даты отображаются в формате `ДД.ММ.ГГГГ, ЧЧ:ММ`; если значение отсутствует, отображается `-`.
- В DTO используется корректное API-наименование полей (`description`, `isActiveSync`, `syncStatus`, `syncDate`, `updatedAt`). Маппинг на поля БД выполняется на уровне backend.
- Поле `syncStatus` принимает значения: `IDLE`, `IN_PROGRESS`, `SUCCESS`, `ERROR`.

### Перечень endpoint

| Название | Запрос | DTO входные параметры | DTO выходные параметры |
| --- | --- | --- | --- |
| Получение списка внешних систем для грида | `GET /api/external-systems` | `ExternalSystemListRequestDto` | `ExternalSystemListResponseDto` |
| Изменение признака регулярной синхронизации | `PATCH /api/external-systems/{id}/sync-enabled` | `ExternalSystemIdPathDto`, `UpdateExternalSystemSyncEnabledRequestDto` | `ExternalSystemDto` |
| Запуск принудительной синхронизации | `POST /api/external-systems/{id}/sync` | `ExternalSystemIdPathDto` | `StartExternalSystemSyncResponseDto` |
| Получение полей внешней системы | `GET /api/external-systems/{id}/fields` | `ExternalSystemIdPathDto` | `ExternalSystemFieldsResponseDto` |
| Изменение наименования и описания внешней системы | `PATCH /api/external-systems/{id}` | `ExternalSystemIdPathDto`, `UpdateExternalSystemRequestDto` | `ExternalSystemDto` |
| Получение истории синхронизаций | `GET /api/external-systems/{id}/sync-history` | `ExternalSystemIdPathDto`, `ExternalSystemSyncHistoryRequestDto` | `ExternalSystemSyncHistoryResponseDto` |

### DTO

#### `ExternalSystemStatus`

Enum статусов синхронизации внешней системы.

```ts
type ExternalSystemStatus =
  | 'IDLE'
  | 'IN_PROGRESS'
  | 'SUCCESS'
  | 'ERROR';
```

#### `ExternalSystemDto`

Основной DTO внешней системы, используется для отображения строки грида и возврата состояния после изменений.

```ts
interface ExternalSystemDto {
  id: string; // UUID
  name: string;
  description: string | null;
  isActiveSync: boolean;
  syncStatus: ExternalSystemStatus;
  syncDate: string | null; // ISO 8601
  createdAt: string; // ISO 8601
  updatedAt: string; // ISO 8601
}
```

Валидация:

- `name`: обязательное поле, длина от 1 до 255 символов, не может состоять только из пробелов.
- `description`: необязательное поле, до 2000 символов.
- `isActiveSync`: обязательное boolean-значение.
- `syncDate`: не может быть датой в будущем.

#### `ExternalSystemIdPathDto`

DTO path-параметров для операций с конкретной внешней системой.

```ts
interface ExternalSystemIdPathDto {
  id: string; // UUID внешней системы
}
```

#### `ExternalSystemListRequestDto`

DTO query-параметров для получения списка внешних систем в гриде.

```ts
interface ExternalSystemListRequestDto {
  page?: number;
  pageSize?: number;
  search?: string;
  sortBy?: ExternalSystemSortField;
  sortDirection?: SortDirection;
}

type ExternalSystemSortField =
  | 'name'
  | 'isActiveSync'
  | 'syncStatus'
  | 'syncDate'
  | 'createdAt';

type SortDirection = 'asc' | 'desc';
```

Валидация и поведение:

- `page`: номер страницы, значение по умолчанию `1`, минимальное значение `1`.
- `pageSize`: размер страницы, значение по умолчанию `20`, допустимый диапазон `1-100`.
- `search`: поиск по полям `name` и `description`, до 255 символов.
- `sortBy`: сортировка доступна только по полям, указанным в `ExternalSystemSortField`.
- `sortDirection`: направление сортировки, значение по умолчанию `asc`.

#### `ExternalSystemListResponseDto`

DTO ответа со списком внешних систем.

```ts
interface ExternalSystemListResponseDto {
  items: ExternalSystemDto[];
  pagination: PaginationDto;
}
```

#### `PaginationDto`

DTO параметров пагинации.

```ts
interface PaginationDto {
  page: number;
  pageSize: number;
  totalItems: number;
  totalPages: number;
}
```

#### `UpdateExternalSystemSyncEnabledRequestDto`

DTO запроса для переключателя "Включена".

```ts
interface UpdateExternalSystemSyncEnabledRequestDto {
  isActiveSync: boolean;
}
```

Поведение:

- `true` включает регулярную синхронизацию.
- `false` отключает регулярную синхронизацию.
- При отключении активный ручной или автоматический процесс синхронизации не прерывается, если он уже был запущен.

#### `StartExternalSystemSyncResponseDto`

DTO ответа после запуска принудительной синхронизации.

```ts
interface StartExternalSystemSyncResponseDto {
  id: string; // UUID внешней системы
  syncStatus: ExternalSystemStatus; // IN_PROGRESS
  syncStartedAt: string; // ISO 8601
}
```

Поведение:

- Endpoint доступен, если внешняя система включена (`isActiveSync = true`) и синхронизация не находится в статусе `IN_PROGRESS`.
- После успешного старта статус системы устанавливается в `IN_PROGRESS`.

#### `ExternalSystemFieldsResponseDto`

DTO ответа со списком полей внешней системы для модального окна "Поля системы".

```ts
interface ExternalSystemFieldsResponseDto {
  externalSystemId: string; // UUID
  fields: ExternalSystemFieldDto[];
}
```

#### `ExternalSystemFieldDto`

DTO поля внешней системы.

```ts
interface ExternalSystemFieldDto {
  id: string; // UUID
  code: string;
  name: string;
  type: ExternalSystemFieldType;
  description: string | null;
  isRequired: boolean;
  isLocked: boolean;
}

type ExternalSystemFieldType =
  | 'STRING'
  | 'NUMBER'
  | 'BOOLEAN'
  | 'DATE'
  | 'DATETIME'
  | 'ENUM';
```

Назначение полей:

- `code`: технический код поля.
- `name`: отображаемое наименование поля.
- `type`: тип значения.
- `description`: описание поля, если задано.
- `isRequired`: признак обязательности поля.
- `isLocked`: признак заблокированного поля, которое нельзя изменить на UI.

#### `UpdateExternalSystemRequestDto`

DTO запроса для действия "Редактировать".

```ts
interface UpdateExternalSystemRequestDto {
  name: string;
  description: string | null;
}
```

Валидация:

- `name`: обязательное поле, длина от 1 до 255 символов, уникально среди внешних систем, не может состоять только из пробелов.
- `description`: необязательное поле, до 2000 символов.

#### `ExternalSystemSyncHistoryRequestDto`

DTO query-параметров для получения истории синхронизаций.

```ts
interface ExternalSystemSyncHistoryRequestDto {
  page?: number;
  pageSize?: number;
  status?: ExternalSystemStatus;
  dateFrom?: string; // ISO 8601
  dateTo?: string; // ISO 8601
}
```

Валидация и поведение:

- `page`: номер страницы, значение по умолчанию `1`, минимальное значение `1`.
- `pageSize`: размер страницы, значение по умолчанию `20`, допустимый диапазон `1-100`.
- `dateFrom` и `dateTo`: необязательный период фильтрации по дате запуска синхронизации.
- `dateTo` не может быть меньше `dateFrom`.

#### `ExternalSystemSyncHistoryResponseDto`

DTO ответа с историей синхронизаций.

```ts
interface ExternalSystemSyncHistoryResponseDto {
  externalSystemId: string; // UUID
  items: ExternalSystemSyncLogDto[];
  pagination: PaginationDto;
}
```

#### `ExternalSystemSyncLogDto`

DTO записи истории синхронизации из аудита `external_system_sync_log`.

```ts
interface ExternalSystemSyncLogDto {
  id: string; // UUID записи аудита
  status: ExternalSystemStatus;
  startedAt: string; // ISO 8601
  finishedAt: string | null; // ISO 8601
  totalCameras: number | null;
  processedCameras: number | null;
  errorMessage: string | null;
}
```

Назначение полей:

- `status`: итоговый или текущий статус синхронизации.
- `startedAt`: дата и время старта синхронизации.
- `finishedAt`: дата и время завершения; `null`, если процесс еще выполняется.
- `totalCameras`: количество камер, полученных из внешней системы.
- `processedCameras`: количество успешно обработанных камер.
- `errorMessage`: текст ошибки для статуса `ERROR`; для остальных статусов `null`.

#### `ErrorResponseDto`

Единый DTO ошибки для всех endpoint.

```ts
interface ErrorResponseDto {
  code: string;
  message: string;
  details?: Record<string, unknown>;
}
```

Типовые ошибки:

- `400 Bad Request` - некорректные входные параметры или ошибка валидации.
- `404 Not Found` - внешняя система с указанным `id` не найдена.
- `409 Conflict` - конфликт состояния, например повторный запуск синхронизации при статусе `IN_PROGRESS` или неуникальное `name`.
