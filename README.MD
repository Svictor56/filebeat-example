## Описание конфигурации Filebeat

Конфигурация Filebeat, приведенная ниже, позволяет собирать и отправлять логи с различных сервисов (Nginx, Apache, MySQL и Битрикс) в Elasticsearch. Для каждого типа логов настроены собственные индексы, применяются преобразования (grok) для структурирования данных и добавляется защита от дублирования записей с помощью уникальных идентификаторов.

### Основные настройки Filebeat

#### Входные данные (inputs)
Конфигурация определяет несколько входных источников (inputs) для чтения логов:

1. **Логи Nginx**:
   - Тип: `log`.
   - Путь: `/var/log/nginx/*.log` — логи веб-сервера Nginx.
   - Поле `log_type` будет иметь значение `nginx` для всех записей.
   - Преобразования через Grok будут применяться только к логам этого типа.

2. **Логи Apache**:
   - Тип: `log`.
   - Путь: `/var/log/apache2/*.log` — логи веб-сервера Apache.
   - Поле `log_type` будет иметь значение `apache` для всех записей.
   - Аналогичные преобразования применяются с помощью Grok для Apache.

3. **Логи MySQL (slow query logs)**:
   - Тип: `log`.
   - Путь: `/var/log/mysql/mysql-slow.log` — логи медленных запросов MySQL.
   - Поле `log_type` будет иметь значение `mysql_slow`.

4. **Логи сайта Битрикс**:
   - Тип: `log`.
   - Путь: `/var/www/www-site/bitrix/*/*.log` — логи приложений Битрикс.
   - Поле `log_type` будет иметь значение `bitrix`.

Все эти записи помечены соответствующими полями `log_type` для дальнейшего анализа и преобразования.

#### Процессоры (Processors)

Для каждого типа логов используется **Grok Processor**, который извлекает данные из полей сообщений и преобразует их в структурированные поля. Преобразования выполняются на основе типа лога, заданного в `log_type`.

1. **Grok для Nginx и Apache**:
   Используются одинаковые шаблоны для извлечения информации о клиентах (IP), времени запроса, методах, статусах и размерах ответов.

   Шаблон Grok:
   `%{IP:client_ip} - - \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{DATA:request} HTTP/%{NUMBER:http_version}" %{NUMBER:status_code} %{NUMBER:response_size}`

2. **Grok для MySQL (slow query log)**:
   Применяются несколько шаблонов для извлечения времени запроса, времени блокировки, количества строк и самого SQL-запроса.

   Шаблон Grok:
   `# Time: %{TIMESTAMP_ISO8601:timestamp}`
   `# Query_time: %{NUMBER:query_time}  Lock_time: %{NUMBER:lock_time} Rows_sent: %{NUMBER:rows_sent}  Rows_examined: %{NUMBER:rows_examined}`
   `SET timestamp=%{NUMBER:timestamp}`
   `%{GREEDYDATA:sql_query}`

3. **Grok для логов Битрикса**:
   Логи Битрикса будут анализироваться на основе метки времени, уровня логирования и сообщения.

   Шаблон Grok:
   `%{TIMESTAMP_ISO8601:timestamp} \[%{LOGLEVEL:log_level}\] \[%{NUMBER:pid}\] %{GREEDYDATA:message}`

4. **Fingerprint Processor**:
   Этот процессор используется для создания уникального идентификатора для каждого события с помощью поля `message`. Используется метод `sha256` для хеширования содержимого поля `message` и сохранения хеша в поле `filebeat_event_id`. Это предотвращает создание дубликатов в Elasticsearch.

   `- fingerprint:`
   `  field: "message"`
   `  target_field: "filebeat_event_id"`
   `  method: "sha256"`
   `  ignore_missing: false`

#### Настройки вывода (output)

1. **Elasticsearch**:
   Логи отправляются в Elasticsearch, используя настройки подключения с логином и паролем:
   `output.elasticsearch:`
   `  hosts: ["http://localhost:9200"]`
   `  username: "your_username"`
   `  password: "your_password"`
   `  index: "filebeat-%{[log_type]}-%{+yyyy.MM.dd}"  # Индексация по типу лога и дате`
   `  setup.ilm.enabled: false  # Отключение ILM, если не используется`
   `  document_id: "%{filebeat_event_id}"  # Использование хеша как уникального идентификатора`

2. **Дополнительные настройки индексов**:
   Включает настройку количества шардов и реплик для индексации:
   `setup.template:`
   `  settings:`
   `    index.number_of_shards: 1  # Количество шардов`
   `    index.number_of_replicas: 1  # Количество реплик`

#### Модули Filebeat

Подключаются модули для обработки логов Nginx, Apache и MySQL:

`filebeat.modules:`
`  - module: nginx`
`  - module: apache`
`  - module: mysql`

Настроен путь для конфигурационных файлов этих модулей:

`filebeat.config.modules:`
`  path: ${path.config}/modules.d/*.yml`
`  reload.enabled: false`

#### Процессоры для метаданных

Для добавления метаданных о хосте и облаке, можно использовать следующие процессоры:

`processors:`
`  - add_host_metadata: ~`
`  - add_cloud_metadata: ~`

#### Логирование Filebeat

Логи Filebeat будут записываться в файлы с ротацией (7 файлов) в указанную директорию:

`logging:`
`  level: info`
`  to_files: true`
`  files:`
`    path: /var/log/filebeat`
`    name: filebeat`
`    keepfiles: 7`
`    permissions: 0644`

### Заключение

Этот конфиг для Filebeat настроен для сбора логов с нескольких сервисов (Nginx, Apache, MySQL, Битрикс) и отправки их в Elasticsearch. Преобразования через Grok помогают структурировать данные, а процессор Fingerprint предотвращает дублирование записей в Elasticsearch. Индексация логов настроена так, чтобы каждый тип лога имел свой собственный индекс, а вывод отправляется в Elasticsearch с использованием настроек безопасности.
