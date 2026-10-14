# Аналіз APK: `composeApp-debug.apk` («Audits», Астарта)

Статичний аналіз: розпакування APK, розбір маніфесту/ресурсів та декомпіляція DEX (androguard).
Додаток не запускався, мережеві запити до бекенду не виконувались.

## 1. Загальні дані

| Параметр | Значення |
|---|---|
| Пакет | `com.astarta.audit.debug` |
| Версія | `0.1.0.380070-debug` (versionCode 380070) |
| Назва | Audits |
| minSdk / targetSdk | 26 (Android 8.0) / 36 |
| Збірка | debug, `android:debuggable="true"`, без R8 (16 DEX, 23.6 МБ) |
| Бекенд | `https://gateway.agrichain.com.ua/` (AgriChain Gateway) |
| Jira | `https://help.astarta.ua/jira/browse/` |
| Точка входу | `com.astarta.audit.MainActivity` |

**Дозволи:** `CAMERA`, `RECORD_AUDIO` (голосовий ввід), `READ_MEDIA_IMAGES`, `READ_EXTERNAL_STORAGE`,
`INTERNET`, `ACCESS_NETWORK_STATE`; `FOREGROUND_SERVICE`, `RECEIVE_BOOT_COMPLETED`, `WAKE_LOCK` (від WorkManager).

## 2. Технологічний стек

- **Kotlin Multiplatform + Compose Multiplatform** (модуль `composeApp`, файли `*_android.kt` → є спільний код, імовірно також iOS-ціль).
- UI: Jetpack Compose, Material 3.
- Мережа: **Ktor Client** (OkHttp engine), kotlinx.serialization, HttpCookies, HttpTimeout (connect 15 с, request/socket 30 с).
- БД: **SQLDelight** (`AuditDatabase`).
- DI: **Koin**.
- Фон: **WorkManager**: періодична задача `astarta-audit-sync`, кожні 15 хв, лише за наявності мережі.
- Інше: kotlinx-datetime, kaml/snakeyaml (YAML), Room (тягнеться залежностями WorkManager).

Код додатку (~1400 класів, `com.astarta.audit`) розкладено за шарами:

```
domain/model, domain/repository   — моделі та інтерфейси репозиторіїв
data/remote                       — Ktor-клієнти API, DTO, Fake-реалізації
data/local                        — SQLDelight-репозиторії, AuthManager, SessionStore, налаштування
data/sync                         — SyncEngine (офлайн-черга), мова→текст, мережа
presentation                      — AuditFlowController (стан майстра аудиту)
ui/screens/{audit,assignment,proposal,resolve}, ui/components, ui/theme
```

## 3. Функціональність

Інтерфейс українською. Бокове меню містить: «Реєстр аудитів», «Реєстр контрольних доручень», «Синхронізація», «Налаштування», «Вихід».

1. **Аудит (майстер).** Фіксація невідповідностей на об'єкті: підприємство → напрям → підрозділ → опис (з голосовим вводом) → фото. Працює офлайн, аудит стає в чергу на відправку.
2. **Контрольні доручення.** Створення задачі з терміном і відповідальними, реєстр з фільтрами й вкладками, виконання і звіт виконавця з фото, перевірка якості (прийняти / повернути на доопрацювання / закрити), скасування.
3. **Пропозиції з покращення.** Подання з співавторами, реалізаторами й чернетками, далі передача на оцінку та впровадження.
4. **Історія аудитів, екран статусу синхронізації, налаштування зв'язку.**

Усі сутності врешті стають задачами в Jira (`jiraKey`, `browseUrl`). Довідник підприємств і напрямів частково вшитий у код (`OrgCatalog`), а частково завантажується з `catalog/audit`.

## 4. Автентифікація

- OIDC discovery: `GET {gateway}/.well-known/openid-configuration` → `token_endpoint`.
- Вхід: **OAuth2 Resource Owner Password Credentials**: `grant_type=password`, `client_id=ScoutMobile` (публічний клієнт), `username`, `password`, `scope=openid`.
- Оновлення: `grant_type=refresh_token`.
- Профіль: `GET services/profiling/api/v2/account` (запасний варіант `services/profiling/api/account`).
- Логін завжди йде на захардкоджений `https://gateway.agrichain.com.ua/` (`BackendConfig.AUTH_GATEWAY_URL`), а не на URL із налаштувань.
- Сесія (`accessToken`, `refreshToken`, `expiresAtMs`) зберігається в таблиці `Session` SQLite. Якщо refresh не вдається, користувача просять увійти знову (`forceReauth`).

## 5. API (відносно `{baseUrl}{apiPrefix}`)

`apiPrefix` = `services/audits/api/v1/`, якщо `baseUrl` містить `gateway.agrichain.com.ua`, інакше `api/v1/`.
Усі запити мають заголовок `Authorization: Bearer <access token>`. Бот-синхронізація додає `X-Sync-Bot: tms-audit-bot` та `Idempotency-Key`.

| Ресурс | Ендпоінти |
|---|---|
| Сесії бота (аудити) | `sessions`, `sessions/{id}`, `sessions/{id}/push`, `sessions/{id}/media`, `sessions/{id}/finish` |
| Задачі | `tasks`, `tasks/{key}`, `tasks/{key}/media`, `tasks/{key}/report` |
| Контрольні доручення | `assignments`, `assignments/permissions`, `assignments/{key}/media?reset=`, `/review` |
| Пропозиції | `proposals`, `proposals/{key}/media?reset=`, `proposals/drafts/{id}/media?reset=`, `/implement`, `/to-implement` |
| Довідники | `catalog/audit`, `users` |
| Вкладення | `/attachments/{id}` |

## 6. Офлайн-синхронізація

Таблиця `QueueItem` (`kind`, `payloadJson`, `idempotencyKey`, `status` ∈ PENDING/FAILED/BLOCKED, `attempts`,
`nextAttemptAtMs`, `remoteSessionId`, `stepsPushed`, `ownerLogin`) реалізує надійну чергу:

- ідемпотентні повтори з backoff;
- відновлення частково відправленої сесії через `stepsPushed`;
- елемент з `ownerLogin` іншого користувача не відправляється, доки той не увійде («Аудит створено іншим користувачем…»);
- статус `BLOCKED` з помилкою «Недостатньо прав».

Інші таблиці: `CompletedAudit` (історія), `DownloadedTask` (кеш задач з Jira), `AppSettings`, `UiPrefs`.

## 7. Зауваження щодо безпеки та якості

### Високий пріоритет

1. **Налаштування бекенду доступні будь-якому користувачу, і на змінений URL іде токен.**
   Пункт «Налаштування» в меню ніяк не залежить від `BuildConfig.DEBUG`. Там можна змінити «URL бекенду», «ID бота (X-Sync-Bot)» і перемикач «Спілкуватися з реальним ботом».
   Логін при цьому завжди йде на справжній gateway, тому **пароль не витікає**. Але всі API-клієнти (`KtorBotTransport`, `KtorTaskRepository`, `KtorMediaUploader` тощо) будують запити від `BackendConfig.baseUrl` і додають `Authorization: Bearer <access token>`.
   Тобто якщо підмінити URL (соціальна інженерія або доступ до розблокованого телефону), чинний токен доступу разом із даними аудитів і фото піде на сторонній сервер.
   **Рекомендація:** у release-збірці прибрати екран або сховати його за `BuildConfig.DEBUG`; не дозволяти довільний хост (allowlist); додавати токен тільки для довірених хостів.
2. **ROPC (password grant).** Додаток бачить пароль користувача. В OAuth 2.1 цей потік прибрано.
   **Рекомендація:** Authorization Code + PKCE через системний браузер (AppAuth).
3. **Токени у відкритому вигляді.** `accessToken` і `refreshToken` лежать у звичайній SQLite (`Session`). Клас `SecureStorage` попри назву є звичайним `SharedPreferences("secure_auth_prefs")`.
   Резервні копії вимкнені, тож витягти токени можна лише з root-доступом або через `run-as` на debuggable-збірці.
   **Рекомендація:** шифрувати ключем з Android Keystore (Tink / EncryptedSharedPreferences / SQLCipher).

### Середній пріоритет

4. **Debug-артефакти:** `debuggable=true`, експортовані `PreviewActivity` з compose-tooling, `Fake*Repository` (`fake-access`/`fake-refresh`, користувач `fake_user`), які вмикаються перемикачем у налаштуваннях. Цей APK не можна видавати реальним користувачам.
5. **Логування:** `System.out.println("Sync: [AUTH] …")` пише в logcat текст помилок сервера та час життя токенів. Data-класи `AuthTokens`/`Session` мають автогенерований `toString()` з `accessToken=`, тож випадковий лог розкриє токен. Варто перевизначити `toString()` і вимикати логи в release.
6. **FileProvider:** у `res/xml/file_paths.xml` шлях `external-path … Android/data/com.astarta.audit/files/Pictures` вшитий жорстко, а в debug пакет називається `com.astarta.audit.debug`, тому для debug цей шлях не збігається. Варто використати `external-files-path path="Pictures"`.

### Низький пріоритет

7. `READ_EXTERNAL_STORAGE` без `android:maxSdkVersion="32"`, хоча на Android 13+ вистачає `READ_MEDIA_IMAGES`.
8. `AcceptAllCookiesStorage` спільний для всіх запитів. Він живе лише в пам'яті, але при зміні URL cookie теж підуть на новий хост.
9. `Json { isLenient = true }` без потреби послаблює парсинг відповідей.

### Що зроблено добре

- `allowBackup=false`, усі домени виключені з cloud-backup і device-transfer (`backup_rules.xml`, `data_extraction_rules.xml`).
- Cleartext-трафік дозволений тільки для `10.0.2.2` (емулятор); весь інший трафік іде через HTTPS.
- FileProvider і внутрішні компоненти не експортовані (експортовані лише стандартні з `DUMP`/`BIND_JOB_SERVICE`).
- Продумана офлайн-черга: ідемпотентність, backoff, прив'язка до автора.
- Окрема обробка закінчення сесії з переходом в офлайн-режим.

## 8. Методика

```
unzip composeApp-debug.apk
androguard: AnalyzeAPK → маніфест, дозволи, компоненти, класи com.astarta.audit
androguard DAD decompiler → BuildConfig, BackendConfig, KtorAuthRepository, AuthManager,
  SecureStorage, HttpClientFactory, Ktor*Api, SyncWorker, App (навігація), SettingsScreen
AXMLPrinter → res/xml/*.xml
витяг const-string з коду додатку → URL, SQL-схема, тексти UI
```
