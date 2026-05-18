# Маппинг камеры и потока Платформы в Thorx3

Документ описывает предлагаемое соответствие полей из основной системы (Платформа) полям API Thorx3 `0.16.0` для синхронизации камер, потоков и калибровки.

## Базовые правила интеграции

- Интеграция направлена из Платформы в Thorx3.
- В Платформе идентификаторы камер и потоков имеют тип UUID, а в Thorx3 идентификаторы камер имеют тип `int32`. Нужна таблица соответствий вида `platformCameraId -> thorx3CameraId` и, при необходимости, `platformStreamId -> thorx3StreamId`.
- В Thorx3 нет отдельного публичного метода создания потока. URL потока передается через поля камеры: `url`, `recognition_url`, `archive_url`, а дополнительные параметры захвата - через `stream_master` / `stream_slave`.
- Калибровка передается отдельным вызовом `/api/v1/ground-calibration` после создания или обновления камеры, когда уже известен `thorx3CameraId`.
- Координаты Платформы `location` передаются как строка `"latitude, longitude"` и должны быть разобраны в два числа.

## Таблица маппинга

| Поле в Платформе | Поле в Thorx3 | Источник (API) | Уровень | Тип интеграции | Пояснение |
| --- | --- | --- | --- | --- | --- |
| `camera.result.id` | `Camera.id` / локальная таблица соответствий | Платформа: камера; Thorx3: `POST /api/v1/cameras`, `GET /api/v1/cameras` | Камера | Справочник соответствий | UUID Платформы не передается напрямую в Thorx3, так как Thorx3 ожидает `int32`. После создания камеры нужно сохранить соответствие UUID камеры и `id`, полученного от Thorx3. |
| `camera.result.name` | `CameraCreationData.alias`, `CameraUpdateData.alias` | Платформа: камера; Thorx3: `POST /api/v1/cameras`, `PUT /api/v1/cameras` | Камера | Создание / обновление | В схемах создания и обновления Thorx3 нет поля `name`; ближайшее поле для человекочитаемого имени - `alias`. В ответной модели Thorx3 есть `Camera.name`, но оно не описано как входное поле. |
| `camera.result.purpose` | `CameraCreationData.description`, `CameraUpdateData.description` | Платформа: камера; Thorx3: `POST /api/v1/cameras`, `PUT /api/v1/cameras` | Камера | Создание / обновление | Назначение камеры переносится в описание. При необходимости можно объединять с адресом. |
| `camera.result.city` | `description` | Платформа: камера; Thorx3: `POST /api/v1/cameras`, `PUT /api/v1/cameras` | Камера | Создание / обновление, составное поле | В Thorx3 нет отдельного поля города. Рекомендуется добавлять в `description`, например: `Санкт-П, Невский пр`. |
| `camera.result.streetAddress` | `description` | Платформа: камера; Thorx3: `POST /api/v1/cameras`, `PUT /api/v1/cameras` | Камера | Создание / обновление, составное поле | В Thorx3 нет отдельного поля адреса. Используется как часть описания. |
| `camera.result.postalCode` | - | Платформа: камера | Камера | Не передается | В API Thorx3 нет отдельного поля почтового индекса. |
| `camera.result.operationalStatus` | `CameraUpdateData.active` / `PUT /api/v1/active-cameras` | Платформа: камера; Thorx3: `PUT /api/v1/cameras`, `PUT /api/v1/active-cameras` | Камера | Обновление, вычисляемое поле | Можно использовать, если статусы Платформы нормализованы до булевого признака активности. При отсутствии значения приоритетнее использовать `isInRepair` и `blockedUntil`. |
| `camera.result.isInRepair` | `CameraUpdateData.active` / `CameraUpdateData.active = false` | Платформа: камера; Thorx3: `PUT /api/v1/cameras`, `PUT /api/v1/active-cameras` | Камера | Обновление, вычисляемое поле | Рекомендуемое правило: `active = !isInRepair`, если нет более приоритетной блокировки. |
| `camera.result.blockedUntil` | `CameraUpdateData.active` / `CameraUpdateData.active = false` | Платформа: камера; Thorx3: `PUT /api/v1/cameras`, `PUT /api/v1/active-cameras` | Камера | Обновление, вычисляемое поле | Если дата блокировки заполнена и находится в будущем, камеру следует выключить в Thorx3. Сама дата в Thorx3 не передается. |
| `camera.result.timezone` | - | Платформа: камера | Камера | Не передается | В `CameraCreationData` и `CameraUpdateData` Thorx3 нет поля часового пояса. |
| `camera.result.location` | `CameraCreationData.latitude`, `CameraCreationData.longitude`, `CameraUpdateData.latitude`, `CameraUpdateData.longitude` | Платформа: камера; Thorx3: `POST /api/v1/cameras`, `PUT /api/v1/cameras` | Камера | Создание / обновление, преобразование | Строку формата `"53.231226, 50.104322"` нужно разобрать как `latitude = 53.231226`, `longitude = 50.104322`. |
| `camera.result.ipAddress` | - / основа для построения `url` | Платформа: камера | Камера / поток | Условное преобразование | Если `stream.result.streamUrl` доступен, IP отдельно не передается. Если URL потока отсутствует, IP может использоваться только для формирования RTSP URL по внешнему правилу. |
| `camera.result.azimuth` | `CameraCreationData.azimuth`, `CameraUpdateData.azimuth` | Платформа: камера; Thorx3: `POST /api/v1/cameras`, `PUT /api/v1/cameras` | Камера | Создание / обновление | Прямое соответствие, тип `int32`. |
| `camera.result.viewAngle` | - | Платформа: камера | Камера | Не передается | В API Thorx3 нет поля угла обзора камеры. |
| `camera.result.model` | - | Платформа: камера | Камера | Не передается | В API Thorx3 нет поля модели камеры. Можно хранить во внешней учетной системе или добавить в `description`, если это требуется бизнес-правилом. |
| `camera.result.icon` | - | Платформа: камера | Камера | Не передается | Поле относится к UI Платформы, аналога в Thorx3 нет. |
| `camera.result.isOnvif` | - | Платформа: камера | Камера | Не передается | В описанных схемах Thorx3 нет ONVIF-признака. |
| `camera.result.credentials` | - | Платформа: камера | Камера / поток | Не передается напрямую | Учетные данные не имеют отдельного поля в Thorx3. Если они нужны для подключения, они должны быть включены в RTSP URL согласно принятой политике безопасности. |
| `camera.result.ptzSettings` | - | Платформа: камера | Камера | Не передается | В API Thorx3 нет полей PTZ-настроек. |
| `camera.result.webPanelUrl` | - | Платформа: камера | Камера | Не передается | В API Thorx3 нет поля URL веб-панели. |
| `camera.result.showNameLabel` | - | Платформа: камера | Камера | Не передается | Поле относится к отображению в UI Платформы. |
| `camera.result.tagIds` | - | Платформа: камера | Камера | Не передается | В API Thorx3 нет поля тегов камеры. |
| `camera.result.tagNames` | - | Платформа: камера | Камера | Не передается | В API Thorx3 нет поля тегов камеры. |
| `camera.result.treeNodeId` | - | Платформа: камера | Камера | Не передается | В API Thorx3 нет поля узла дерева. При необходимости используется только во внутренней маршрутизации интеграции. |
| `camera.result.createdAt` | - | Платформа: камера | Камера | Не передается | Служебная дата Платформы. В схемах Thorx3 нет входного поля создания. |
| `camera.result.updatedAt` | - | Платформа: камера | Камера | Не передается | Служебная дата Платформы. Используется интеграцией для определения необходимости синхронизации. |
| `stream.result.id` | `Stream.id` / локальная таблица соответствий | Платформа: поток; Thorx3: `GET /api/v1/cameras` | Поток | Справочник соответствий | В Thorx3 поток возвращается внутри `Camera.streams[]` с `int32 id`. При необходимости нужно сохранять соответствие UUID потока Платформы и `Stream.id` Thorx3. |
| `stream.result.name` | `Stream.name` | Платформа: поток; Thorx3: `GET /api/v1/cameras` | Поток | Только чтение / косвенное соответствие | В схемах создания и обновления камеры нет входного поля имени потока. Имя можно использовать только для сверки с ответом Thorx3 или внутреннего логирования. |
| `stream.result.cameraId` | `Camera.id` / `camera_id` | Платформа: поток; Thorx3: все методы с `camera_id` | Поток / камера | Справочник соответствий | UUID камеры Платформы преобразуется в `thorx3CameraId` через таблицу соответствий. |
| `stream.result.cameraName` | - | Платформа: поток | Поток | Не передается | Дублирует `camera.result.name`; отдельного поля в Thorx3 нет. |
| `stream.result.streamUrl` | `CameraCreationData.url`, `CameraUpdateData.url` | Платформа: поток; Thorx3: `POST /api/v1/cameras`, `PUT /api/v1/cameras` | Поток | Создание / обновление | Основной RTSP URL камеры в Thorx3. Для примера: `rtsp://172.29.11.135:8654/testeram2`. |
| `stream.result.streamUrl` | `CameraCreationData.recognition_url`, `CameraUpdateData.recognition_url` | Платформа: поток; Thorx3: `POST /api/v1/cameras`, `PUT /api/v1/cameras` | Поток / распознавание | Создание / обновление | Если поток используется для распознавания, тот же URL передается в `recognition_url`. |
| `stream.result.streamUrl` | `CameraCreationData.archive_url`, `CameraUpdateData.archive_url` | Платформа: поток; Thorx3: `POST /api/v1/cameras`, `PUT /api/v1/cameras` | Поток / архив | Создание / обновление | Если для потока включен архив, тот же URL передается в `archive_url`, если нет отдельного архивного URL. |
| `stream.result.streamType` | `stream_master` / `stream_slave` | Платформа: поток; Thorx3: `POST /api/v1/cameras`, `PUT /api/v1/cameras` | Поток | Условное соответствие | `Main` рекомендуется связывать с `stream_master`. Для дополнительных потоков можно использовать `stream_slave`, если такой поток есть в Платформе и он нужен Thorx3. |
| `stream.result.videoServerId` | - | Платформа: поток | Поток | Не передается | В API Thorx3 нет поля видео-сервера Платформы. |
| `stream.result.videoServerName` | - | Платформа: поток | Поток | Не передается | В API Thorx3 нет поля имени видео-сервера Платформы. |
| `stream.result.retentionSeconds` | `CameraCreationData.archive`, `CameraUpdateData.archive` | Платформа: поток; Thorx3: `POST /api/v1/cameras`, `PUT /api/v1/cameras` | Поток / архив | Вычисляемое поле | Рекомендуемое правило: `archive = retentionSeconds > 0`. Длительность хранения `7200` секунд отдельным полем камеры Thorx3 не поддерживается в предоставленной спецификации. |
| `stream.result.retentionSeconds` | - / возможно `Settings` | Платформа: поток; Thorx3: `GET/POST/PUT /api/v1/settings` | Поток / архив | Требует уточнения | Если Thorx3 поддерживает длительность архива через настройки комплекса, нужен отдельный ключ настройки. В OpenAPI конкретный ключ для retention не описан. |
| `stream.result.calibration.rectangleHeight` | `SettingCalibrationCreationData.rectangle.height` | Платформа: поток; Thorx3: `POST /api/v1/ground-calibration` | Калибровка | Создание / пересоздание | Прямое соответствие высоты калибровочного прямоугольника. |
| `stream.result.calibration.rectangleWidth` | `SettingCalibrationCreationData.rectangle.width` | Платформа: поток; Thorx3: `POST /api/v1/ground-calibration` | Калибровка | Создание / пересоздание | Прямое соответствие ширины калибровочного прямоугольника. |
| `stream.result.calibration.rectanglePoints` | `SettingCalibrationCreationData.rectangle.points` | Платформа: поток; Thorx3: `POST /api/v1/ground-calibration` | Калибровка | Создание / пересоздание | Массив относительных точек `{x, y}` переносится без изменения структуры. |
| `stream.result.calibration.projectionMatrix` | `SettingCalibrationCreationData.projection_matrix` | Платформа: поток; Thorx3: `POST /api/v1/ground-calibration` | Калибровка | Создание / пересоздание | Массив чисел переносится в snake_case поле Thorx3. |
| `stream.result.createdAt` | - | Платформа: поток | Поток | Не передается | Служебная дата Платформы. Используется только для логики синхронизации. |
| `stream.result.updatedAt` | - | Платформа: поток | Поток | Не передается | Служебная дата Платформы. Используется для определения необходимости обновления URL, архива или калибровки. |

## Обязательные поля Thorx3, которых нет напрямую в Платформе

| Поле Thorx3 | Источник / правило заполнения | Уровень | Тип интеграции | Пояснение |
| --- | --- | --- | --- | --- |
| `CameraCreationData.url` | `stream.result.streamUrl` | Камера / поток | Создание | Обязательное поле Thorx3. Используется основной поток. |
| `CameraCreationData.description` | `camera.result.purpose` + адресные поля | Камера | Создание | Обязательное поле Thorx3. Если назначение пустое, рекомендуется использовать имя или адрес камеры. |
| `CameraCreationData.recognition` | Константа или настройка интеграции | Камера / распознавание | Создание | Обязательное поле Thorx3. Рекомендуемое значение `true`, если поток должен участвовать в распознавании. |
| `CameraCreationData.archive` | `stream.result.retentionSeconds > 0` | Камера / архив | Создание | Обязательное поле Thorx3. |
| `CameraUpdateData.id` | `thorx3CameraId` из таблицы соответствий | Камера | Обновление | Обязательное поле для обновления камеры в Thorx3. |
| `SettingCalibrationCreationData.camera_id` | `thorx3CameraId` из таблицы соответствий | Калибровка | Создание | Калибровка создается после того, как камера уже заведена в Thorx3. |

## Рекомендуемые вызовы Thorx3

1. Создать камеру: `POST /api/v1/cameras`.
2. Сохранить соответствие `platformCameraId -> thorx3CameraId`.
3. Если есть калибровка потока, создать калибровку: `POST /api/v1/ground-calibration`.
4. При изменении полей камеры или URL потока обновить камеру: `PUT /api/v1/cameras`.
5. При изменении только активности использовать `PUT /api/v1/active-cameras`.
6. При изменении калибровки удалить старую калибровку при необходимости и создать новую через `/api/v1/ground-calibration`.

## Пример payload для создания камеры Thorx3

```json
{
  "alias": "Поток с большой группой людей",
  "description": "тест2; Санкт-П, Невский пр",
  "url": "rtsp://172.29.11.135:8654/testeram2",
  "recognition": true,
  "recognition_url": "rtsp://172.29.11.135:8654/testeram2",
  "archive": true,
  "archive_url": "rtsp://172.29.11.135:8654/testeram2",
  "azimuth": 217,
  "latitude": 53.231226,
  "longitude": 50.104322,
  "stream_master": null,
  "stream_slave": null
}
```

## Пример payload для создания калибровки Thorx3

```json
{
  "camera_id": 1,
  "rectangle": {
    "height": 10,
    "width": 10,
    "points": [
      { "x": 0.3063157894736842, "y": 0.368458559570153 },
      { "x": 0.6711842105263158, "y": 0.3649048027830484 },
      { "x": 0.7282894736842105, "y": 0.8248762975097322 },
      { "x": 0.28421052631578947, "y": 0.8123691711914031 }
    ]
  },
  "projection_matrix": [
    28.273523594257032,
    1.4079268733166723,
    -9.17938940870005,
    0.2659159949408756,
    27.301910349854367,
    -10.141076828944787,
    0.08994838878469075,
    0.4806402124610377,
    0.7953513879226454
  ]
}
```
