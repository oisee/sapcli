# Справочник RFC функциональных модулей в sapcli

**Дата создания:** 2025-11-30
**Проект:** oisee/sapcli - Command Line Interface for SAP products
**Всего уникальных RFC модулей:** 19

---

## Оглавление

1. [Управление пользователями (User Management)](#1-управление-пользователями-user-management)
2. [Управление безопасностью и SSL (STRUST)](#2-управление-безопасностью-и-ssl-strust)
3. [Произвольное выполнение RFC](#3-произвольное-выполнение-rfc)
4. [Обработка BAPI ответов](#4-обработка-bapi-ответов)
5. [Сводная таблица всех модулей](#5-сводная-таблица-всех-модулей)

---

## 1. Управление пользователями (User Management)

**Файл:** `sap/rfc/user.py`
**Команда CLI:** `sapcli user`
**Количество модулей:** 6

### 1.1 BAPI_USER_CREATE1

**Назначение:** Создание нового ABAP пользователя

**Место вызова:** `UserManager.create_user()` (строка 427)

**Входные параметры:**
- `USERNAME` - имя пользователя (логин)
- `ADDRESS` - структура с данными:
  - `FIRSTNAME` - имя
  - `LASTNAME` - фамилия
  - `E_MAIL` - email адрес
- `PASSWORD` - структура:
  - `BAPIPWD` - пароль (для типа Service) или dummy-пароль
- `ALIAS` - структура:
  - `USERALIAS` - алиас для HTTP аутентификации
- `LOGONDATA` - структура:
  - `USTYP` - тип пользователя (A=Dialog, B=Service, C=Communications Data, S=System)
  - `CLASS` - группа пользователя
  - `GLTGV` - дата начала валидности (формат YYYYMMDD)
  - `GLTGB` - дата окончания валидности (формат YYYYMMDD)

**Выходные параметры:**
- `RETURN` - таблица BAPIRET2 с результатами операции

**Особенности:**
- Для пользователей типа Dialog (A) и Communications Data (C) после создания вызывается `SUSR_USER_CHANGE_PASSWORD_RFC`
- По умолчанию дата начала = сегодня, дата окончания = 20991231

---

### 1.2 BAPI_USER_CHANGE

**Назначение:** Изменение существующего ABAP пользователя

**Место вызова:** `UserManager.change_user()` (строка 443)

**Входные параметры:**
- `USERNAME` - имя пользователя
- `PASSWORD` - новый пароль (опционально)
- `PASSWORDX` - структура с флагом `BAPIPWD='X'` (если меняется пароль)
- `ALIAS` - новый алиас (опционально)
- `ALIASX` - структура с флагом `BAPIALIAS='X'` (если меняется алиас)

**Выходные параметры:**
- `RETURN` - таблица BAPIRET2

**Особенности:**
- Перед вызовом выполняется `BAPI_USER_GET_DETAIL` для определения типа пользователя
- Для Dialog и Communications Data типов дополнительно вызывается `SUSR_USER_CHANGE_PASSWORD_RFC`

---

### 1.3 BAPI_USER_GET_DETAIL

**Назначение:** Получение детальной информации о пользователе

**Место вызова:** `UserManager.fetch_user_details()` (строки 420-422)

**Входные параметры:**
- `USERNAME` - имя пользователя

**Выходные параметры:**
- `LOGONDATA` - структура с данными авторизации (включая `USTYP` - тип пользователя)
- `ADDRESS` - адресные данные
- `RETURN` - таблица BAPIRET2
- Множество других структур с детальной информацией

**Использование:**
- Получение типа пользователя (`USTYP`) перед изменением пароля

---

### 1.4 BAPI_USER_ACTGROUPS_ASSIGN

**Назначение:** Назначение ролей (Activity Groups) пользователю

**Место вызова:** `UserManager.assign_roles()` (строка 460)

**Входные параметры:**
- `USERNAME` - имя пользователя
- `ACTIVITYGROUPS` - таблица с ролями:
  - `AGR_NAME` - имя роли
  - `FROM_DAT` - дата начала валидности (по умолчанию = сегодня)
  - `TO_DAT` - дата окончания валидности (по умолчанию = 20991231)

**Выходные параметры:**
- `RETURN` - таблица BAPIRET2

---

### 1.5 BAPI_USER_PROFILES_ASSIGN

**Назначение:** Назначение профилей авторизации пользователю

**Место вызова:** `UserManager.assign_profiles()` (строка 470)

**Входные параметры:**
- `USERNAME` - имя пользователя
- `PROFILES` - таблица с профилями:
  - `BAPIPROF` - имя профиля

**Выходные параметры:**
- `RETURN` - таблица BAPIRET2

---

### 1.6 SUSR_USER_CHANGE_PASSWORD_RFC

**Назначение:** Изменение продуктивного пароля пользователя (для Dialog и Communications Data типов)

**Место вызова:** `UserManager._call_user_change_password()` (строка 385)

**Входные параметры:**
- `BNAME` - имя пользователя
- `PASSWORD` - текущий пароль (или dummy-пароль)
- `NEW_PASSWORD` - новый пароль
- `USE_BAPI_RETURN` - флаг (1 = использовать BAPI формат ответа)

**Выходные параметры:**
- `RETURN` - BAPIRET2 структура

**Особенности:**
- Используется обходной маневр: при создании устанавливается dummy-пароль, затем он меняется на реальный
- Dummy-пароль берется из переменной окружения `SAPCLI_ABAP_USER_DUMMY_PASSWORD` (по умолчанию: `DummyPwd123!`)
- Применяется только для типов: A (Dialog), C (Communications Data)
- Для типа B (Service) пароль устанавливается напрямую через BAPI_USER_CREATE1/CHANGE

---

## 2. Управление безопасностью и SSL (STRUST)

**Файл:** `sap/rfc/strust.py`
**Команда CLI:** `sapcli strust`
**Количество модулей:** 13

### 2.1 SSFR_PSE_CHECK

**Назначение:** Проверка существования и статуса PSE (Personal Security Environment) хранилища

**Место вызова:** `SSLCertStorage.exists()` (строки 148-151)

**Входные параметры:**
- `IS_STRUST_IDENTITY` - структура идентификации:
  - `PSE_CONTEXT` - контекст PSE (например: 'SSLS', 'SSLC')
  - `PSE_APPLIC` - приложение PSE (например: 'DFAULT', 'ANONYM')

**Выходные параметры:**
- `ET_BAPIRET2` - таблица с результатами проверки

**Коды возврата:**
- `TYPE='E', NUMBER='031'` - PSE не существует
- `TYPE='S'` - PSE существует и в порядке
- Другие ошибки - PSE поврежден

---

### 2.2 SSFR_PSE_CREATE

**Назначение:** Создание нового PSE файла с криптографическими ключами

**Место вызова:** `SSLCertStorage.create_pse()` (строка 185)

**Входные параметры:**
- `IS_STRUST_IDENTITY` - идентификация PSE
- `IV_ALG` - алгоритм шифрования:
  - `R` = RSA
  - `S` = RSAwithSHA256
  - `D` = DSA
  - `G` = GOST_R_34.10-94
  - `H` = GOST_R_34.10-2001
  - `X` = CCL (Custom Crypto Library)
- `IV_KEYLEN` - длина ключа в битах (по умолчанию: 2048)
- `IV_REPLACE_EXISTING_PSE` - флаг перезаписи ('X' = перезаписать, '-' = ошибка если существует)
- `IV_DN` - Distinguished Name (опционально)

**Выходные параметры:**
- `ET_BAPIRET2` - таблица результатов

---

### 2.3 SSFR_PSE_REMOVE

**Назначение:** Удаление PSE файла из хранилища STRUST

**Место вызова:** `SSLCertStorage.remove()` (строки 215-218)

**Входные параметры:**
- `IS_STRUST_IDENTITY` - идентификация PSE

**Выходные параметры:**
- `ET_BAPIRET2` - таблица результатов

---

### 2.4 SSFR_PSE_UPLOAD_XSTRING

**Назначение:** Загрузка PSE файла из бинарных данных (XSTRING)

**Место вызова:** `SSLCertStorage.upload()` (строки 236-239)

**Входные параметры:**
- `IS_STRUST_IDENTITY` - идентификация PSE
- `IV_PSE_XSTRING` - бинарное содержимое PSE файла
- `IV_REPLACE_EXISTING_PSE` - флаг перезаписи ('X' или '-')
- `IV_PSEPIN` - пароль PSE файла (опционально)

**Выходные параметры:**
- `ET_BAPIRET2` - таблица результатов

**Использование:**
- Импорт PSE файла, созданного внешними средствами (OpenSSL, etc.)

---

### 2.5 SSFR_IDENTITY_CREATE

**Назначение:** Создание новой STRUST Application Identity (регистрация PSE в системе)

**Место вызова:** `SSLCertStorage.create_identity()` (строки 196-200)

**Входные параметры:**
- `IS_STRUST_IDENTITY` - структура:
  - `PSE_CONTEXT` - контекст
  - `PSE_APPLIC` - приложение
  - `PSE_DESCRIPT` - описание (опционально)
  - `SPRSL` - языковой код SAP (опционально)
- `IV_REPLACE_EXISTING_APPL` - флаг перезаписи ('X' или '-')

**Выходные параметры:**
- `ET_BAPIRET2` - таблица результатов

---

### 2.6 SSFR_GET_ALL_STRUST_IDENTITIES

**Назначение:** Получение списка всех существующих STRUST идентификаций в системе

**Место вызова:** `list_identities()` (строка 358)

**Входные параметры:** нет

**Выходные параметры:**
- `ET_STRUST_IDENTITIES` - таблица всех зарегистрированных PSE идентификаций
- `ET_BAPIRET2` - таблица результатов

**Использование:**
- Получение списка всех доступных PSE хранилищ для администрирования

---

### 2.7 SSFR_GET_OWNCERTIFICATE

**Назначение:** Получение собственного (системного) сертификата из PSE

**Место вызова:** `SSLCertStorage.get_own_certificate()` (строки 278-281)

**Входные параметры:**
- `IS_STRUST_IDENTITY` - идентификация PSE

**Выходные параметры:**
- `EV_CERTIFICATE` - X.509 сертификат в Base64 формате
- `ET_BAPIRET2` - таблица результатов

**Использование:**
- Экспорт собственного сертификата для предоставления партнерам

---

### 2.8 SSFR_GET_CERTIFICATELIST

**Назначение:** Получение списка доверенных сертификатов из PSE хранилища

**Место вызова:** `SSLCertStorage.get_certificates()` (строки 292-295)

**Входные параметры:**
- `IS_STRUST_IDENTITY` - идентификация PSE

**Выходные параметры:**
- `ET_CERTIFICATELIST` - таблица X.509 сертификатов в бинарном формате

**Использование:**
- Просмотр всех импортированных CA сертификатов

---

### 2.9 SSFR_PUT_CERTIFICATE

**Назначение:** Добавление X.509 сертификата в список доверенных (trust list)

**Место вызова:** `SSLCertStorage.put_certificate()` (строки 252-256)

**Входные параметры:**
- `IS_STRUST_IDENTITY` - идентификация PSE
- `IV_CERTIFICATE` - бинарные данные X.509 сертификата

**Выходные параметры:**
- `ET_BAPIRET2` - таблица результатов

**Коды ошибок:**
- `TYPE='E', NUMBER='522'` - сертификат уже существует

**Использование:**
- Импорт CA сертификатов для установления доверия

---

### 2.10 SSFR_PUT_CERTRESPONSE

**Назначение:** Загрузка подписанного Identity Certificate (ответ от Certificate Authority)

**Место вызова:** `SSLCertStorage.put_identity_cert()` (строки 327-333)

**Входные параметры:**
- `IS_STRUST_IDENTITY` - идентификация PSE
- `IT_CERTRESPONSE` - таблица строк (каждая по 80 символов) с содержимым сертификата
- `IV_CERTRESPONSE_LEN` - общая длина сертификата в байтах
- `IV_PSEPIN` - пароль PSE (обычно пустой)

**Выходные параметры:**
- `ET_BAPIRET2` - таблица результатов

**Особенности:**
- Сертификат должен быть разбит на строки по 80 символов
- Используется класс `PKCResponseABAPData` для форматирования данных

**Workflow:**
1. Создать PSE с `SSFR_PSE_CREATE`
2. Получить CSR с `SSFR_GET_CERTREQUEST`
3. Подписать CSR в Certificate Authority
4. Загрузить подписанный сертификат с `SSFR_PUT_CERTRESPONSE`

---

### 2.11 SSFR_GET_CERTREQUEST

**Назначение:** Получение Certificate Signing Request (CSR) для отправки в Certificate Authority

**Место вызова:** `SSLCertStorage.get_csr()` (строки 311-314)

**Входные параметры:**
- `IS_STRUST_IDENTITY` - идентификация PSE

**Выходные параметры:**
- `ET_CERTREQUEST` - таблица строк с CSR в формате PEM
- `ET_BAPIRET2` - таблица результатов

**Использование:**
- Генерация CSR для получения подписанного сертификата от CA

---

### 2.12 SSFR_PARSE_CERTIFICATE

**Назначение:** Парсинг и извлечение информации из X.509 сертификата

**Места вызова:**
- `SSLCertStorage.parse_certificate()` (строки 301-304)
- `iter_storage_certificates()` (строка 372)

**Входные параметры:**
- `IV_CERTIFICATE` - бинарные данные X.509 сертификата

**Выходные параметры:**
- Структура с деталями сертификата:
  - Issuer (издатель)
  - Subject (владелец)
  - Serial Number (серийный номер)
  - Validity dates (даты валидности)
  - Public key info
  - Extensions

**Использование:**
- Просмотр деталей сертификата перед импортом
- Валидация сертификатов

---

### 2.13 ICM_SSL_PSE_CHANGED

**Назначение:** Уведомление ICM (Internet Communication Manager) об изменениях в PSE

**Место вызова:** `notify_icm_changed_pse()` (строка 352)

**Входные параметры:** нет

**Выходные параметры:** нет

**Использование:**
- ОБЯЗАТЕЛЬНЫЙ вызов после любых изменений PSE
- Заставляет ICM перечитать PSE без перезапуска системы
- Критично для активации новых сертификатов в SSL соединениях

---

### Стандартные PSE идентификации

Определены в `IDENTITY_MAPPING`:

| Константа | PSE_CONTEXT | PSE_APPLIC | Назначение |
|-----------|-------------|------------|-----------|
| `server_standard` | SSLS | DFAULT | SSL Server (стандартный HTTPS сервер) |
| `client_anonymous` | SSLC | ANONYM | Anonymous SSL Client |
| `client_standard` | SSLC | DFAULT | Standard SSL Client |

---

## 3. Произвольное выполнение RFC

**Файл:** `sap/cli/startrfc.py`
**Команда CLI:** `sapcli startrfc <FM_NAME>`

### 3.1 Динамический вызов любого RFC модуля

**Место вызова:** `startrfc()` (строка 116)

**Возможности:**
- Вызов **любого** RFC-enabled функционального модуля по имени
- Передача параметров в формате JSON
- Поддержка дополнительных типов параметров через флаги CLI

**Форматы параметров:**

1. **JSON (основной формат):**
```bash
sapcli startrfc RFC_SYSTEM_INFO '{}'
sapcli startrfc BAPI_USER_GET_DETAIL '{"USERNAME": "DEVELOPER"}'
```

2. **Чтение из stdin:**
```bash
echo '{"QUERY": "SELECT * FROM MARA"}' | sapcli startrfc MY_RFC_FM -
```

3. **String параметры (-S):**
```bash
sapcli startrfc BAPI_USER_CREATE1 '{}' -S USERNAME:TESTUSER -S ADDRESS.FIRSTNAME:John
```

4. **Integer параметры (-I):**
```bash
sapcli startrfc Z_CUSTOM_FM '{}' -I MAX_ROWS:1000 -I CLIENT:100
```

5. **File параметры (-F):**
```bash
sapcli startrfc Z_UPLOAD_FILE '{}' -F FILE_CONTENT:/path/to/file.bin
```

**Режимы проверки результата (-c):**

1. **raw (по умолчанию)** - вывести сырой ответ без проверок
2. **bapi** - проверить структуру RETURN и вернуть код ошибки при TYPE='E'

**Форматы вывода (-o):**

1. **human (по умолчанию)** - pretty-printed формат
2. **json** - JSON формат (bytes конвертируются в Base64)

**Примеры использования:**

```bash
# Получение информации о системе
sapcli startrfc RFC_SYSTEM_INFO '{}'

# Создание пользователя с проверкой BAPI ответа
sapcli startrfc BAPI_USER_CREATE1 \
  '{"USERNAME":"TEST001","PASSWORD":{"BAPIPWD":"Init1234"},"LOGONDATA":{"USTYP":"B"}}' \
  -c bapi

# Вывод в JSON файл
sapcli startrfc RFC_READ_TABLE \
  '{"QUERY_TABLE":"USR02","DELIMITER":"|"}' \
  -o json -R /tmp/response.json

# Загрузка файла
sapcli startrfc Z_UPLOAD_BINARY '{}' \
  -S FILENAME:document.pdf \
  -F BINARY_DATA:/path/to/document.pdf
```

---

## 4. Обработка BAPI ответов

**Файл:** `sap/rfc/bapi.py`

### 4.1 Класс BAPIReturn

Универсальный обработчик BAPI ответов (структура RETURN).

**Типы сообщений BAPI (BAPI_MTYPE):**

| Код | Тип | Описание |
|-----|-----|----------|
| S | Success | Успешное выполнение |
| E | Error | Ошибка |
| W | Warning | Предупреждение |
| I | Info | Информационное сообщение |
| A | Abort | Критическая ошибка (аварийное завершение) |

**Методы:**

1. **`__init__(bapi_return)`** - принимает dict или list[dict]
2. **`is_error`** - свойство, True если есть TYPE='E'
3. **`is_empty`** - свойство, True если нет сообщений
4. **`error_message`** - свойство, первое сообщение об ошибке
5. **`message_lines()`** - список всех сообщений в читаемом формате
6. **`contains(msg_class, msg_number)`** - проверка наличия конкретного сообщения
7. **`raise_for_error()`** - статический метод, выбрасывает BAPIError при ошибках

**Формат сообщения:**
```
{TYPE}({ID}|{NUMBER}): {MESSAGE}
```

Пример:
```
Error(SU|001): User TEST001 already exists
Success: User created successfully
```

**Использование:**

```python
# В UserManager._call_bapi_method()
resp = connection.call('BAPI_USER_CREATE1', **params)
BAPIError.raise_for_error(resp['RETURN'], resp)  # Выбросит исключение при ошибке
return resp

# В strust.py
bapiret = BAPIReturn(stat['ET_BAPIRET2'])
if bapiret.is_error:
    raise BAPIError(bapiret, stat)
```

---

## 5. Сводная таблица всех модулей

### 5.1 По категориям

| Категория | Количество | Файл | Команда CLI |
|-----------|------------|------|-------------|
| User Management | 6 | sap/rfc/user.py | `sapcli user` |
| STRUST/SSL | 13 | sap/rfc/strust.py | `sapcli strust` |
| **Всего фиксированных** | **19** | - | - |
| Динамический вызов | ∞ | sap/cli/startrfc.py | `sapcli startrfc` |

### 5.2 Алфавитный список всех модулей

| № | RFC Module | Категория | Назначение |
|---|-----------|-----------|-----------|
| 1 | BAPI_USER_ACTGROUPS_ASSIGN | User | Назначение ролей пользователю |
| 2 | BAPI_USER_CHANGE | User | Изменение пользователя |
| 3 | BAPI_USER_CREATE1 | User | Создание пользователя |
| 4 | BAPI_USER_GET_DETAIL | User | Получение данных пользователя |
| 5 | BAPI_USER_PROFILES_ASSIGN | User | Назначение профилей пользователю |
| 6 | ICM_SSL_PSE_CHANGED | STRUST | Уведомление ICM об изменении PSE |
| 7 | SSFR_GET_ALL_STRUST_IDENTITIES | STRUST | Список всех PSE идентификаций |
| 8 | SSFR_GET_CERTIFICATELIST | STRUST | Список доверенных сертификатов |
| 9 | SSFR_GET_CERTREQUEST | STRUST | Получение CSR |
| 10 | SSFR_GET_OWNCERTIFICATE | STRUST | Получение собственного сертификата |
| 11 | SSFR_IDENTITY_CREATE | STRUST | Создание PSE идентификации |
| 12 | SSFR_PARSE_CERTIFICATE | STRUST | Парсинг X.509 сертификата |
| 13 | SSFR_PSE_CHECK | STRUST | Проверка существования PSE |
| 14 | SSFR_PSE_CREATE | STRUST | Создание PSE файла |
| 15 | SSFR_PSE_REMOVE | STRUST | Удаление PSE файла |
| 16 | SSFR_PSE_UPLOAD_XSTRING | STRUST | Загрузка PSE из бинарных данных |
| 17 | SSFR_PUT_CERTIFICATE | STRUST | Добавление доверенного сертификата |
| 18 | SSFR_PUT_CERTRESPONSE | STRUST | Загрузка подписанного Identity Certificate |
| 19 | SUSR_USER_CHANGE_PASSWORD_RFC | User | Изменение продуктивного пароля |

### 5.3 По частоте использования

| RFC Module | Количество вызовов в коде | Критичность |
|-----------|---------------------------|-------------|
| BAPI_USER_CREATE1 | 1 | Высокая |
| BAPI_USER_CHANGE | 1 | Высокая |
| BAPI_USER_GET_DETAIL | 1 | Средняя |
| SUSR_USER_CHANGE_PASSWORD_RFC | 1 | Высокая |
| SSFR_PSE_CREATE | 1 | Высокая |
| SSFR_PUT_CERTIFICATE | 1 | Высокая |
| SSFR_PARSE_CERTIFICATE | 2 | Средняя |
| ICM_SSL_PSE_CHANGED | 1 | **Критическая** |
| Остальные | 1 | Средняя |

---

## 6. Важные замечания

### 6.1 Что НЕ использует RFC

1. **Transport Management (CTS)** - использует **ADT HTTP API**:
   - `/sap/bc/adt/cts/transportrequests/*`
   - `/sap/bc/adt/cts/transports/*/newreleasejobs`

2. **gCTS (Git-enabled CTS)** - использует **REST API**:
   - `/sap/bc/cts_abapvcs/repository`
   - `/sap/bc/cts_abapvcs/repository/{rid}/*`

3. **Все разработческие операции** - используют **ADT протокол** (HTTP/XML):
   - ABAP Classes, Programs, Includes, Interfaces
   - Function Groups, Function Modules
   - Data Definitions (CDS Views)
   - Packages
   - ABAP Unit (aunit)
   - ATC (Code Inspector)
   - Data Preview
   - BSP Applications
   - Fiori Launchpad

### 6.2 Когда используется RFC

RFC используется **только** в следующих случаях:

1. **Административные задачи без ADT API:**
   - Управление пользователями
   - Назначение ролей и профилей

2. **Системная безопасность:**
   - Управление PSE и SSL сертификатами
   - STRUST операции

3. **Legacy функции:**
   - Функции, которые исторически доступны только через RFC
   - Функции, для которых нет современных ADT endpoint'ов

4. **Специфические задачи по запросу:**
   - Через команду `startrfc` для вызова custom FM

### 6.3 Архитектурные принципы

1. **Предпочтение ADT над RFC:**
   - Где возможно, используется ADT HTTP API
   - RFC только когда ADT недоступен

2. **Модульная организация:**
   - Каждая категория RFC вызовов в отдельном файле
   - Четкое разделение User / STRUST / Custom

3. **Обработка ошибок:**
   - Все BAPI вызовы проверяются через `BAPIError.raise_for_error()`
   - Унифицированная обработка BAPIRET2 структур

4. **Dependency Injection:**
   - Все RFC вызовы получают connection как параметр
   - Нет глобальных connection объектов

---

## 7. Примеры полных рабочих процессов

### 7.1 Создание пользователя с ролями

```bash
# 1. Создать пользователя типа Dialog
sapcli user create DEV001 \
  --first-name John \
  --last-name Developer \
  --email john.developer@company.com \
  --type A \
  --password SecurePass123! \
  --password-productive

# Внутри выполняется:
# - BAPI_USER_CREATE1 (с dummy паролем)
# - SUSR_USER_CHANGE_PASSWORD_RFC (установка реального пароля)

# 2. Назначить роли
sapcli user assign-roles DEV001 Z_DEVELOPER SAP_BC_BASIS_ADMIN

# Внутри выполняется:
# - BAPI_USER_ACTGROUPS_ASSIGN
```

### 7.2 Настройка SSL сертификата

```bash
# 1. Создать PSE
sapcli strust createpse server_standard \
  --algorithm RSAwithSHA256 \
  --key-length 4096 \
  --dn "CN=myserver.company.com,O=My Company,C=US"

# Внутри выполняется:
# - SSFR_IDENTITY_CREATE
# - SSFR_PSE_CREATE

# 2. Получить CSR
sapcli strust getcsr server_standard > server.csr

# Внутри выполняется:
# - SSFR_GET_CERTREQUEST

# 3. (Подписать CSR в CA - внешний процесс)

# 4. Загрузить подписанный сертификат
sapcli strust putcertresponse server_standard server-signed.crt

# Внутри выполняется:
# - SSFR_PUT_CERTRESPONSE

# 5. Добавить CA сертификат в trust list
sapcli strust putcertificate server_standard ca-root.crt

# Внутри выполняется:
# - SSFR_PUT_CERTIFICATE

# 6. Уведомить ICM
sapcli strust icm-notify

# Внутри выполняется:
# - ICM_SSL_PSE_CHANGED
```

---

**Документ подготовлен на основе анализа исходного кода проекта oisee/sapcli**
**Все RFC вызовы найдены и задокументированы**
**Последнее обновление:** 2025-11-30
