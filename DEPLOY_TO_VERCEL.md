# SELLFORM — публикация на Vercel

## 1. Создать GitHub-репозиторий

1. Откройте https://github.com/new и войдите в свой аккаунт.
2. Repository name: `sellform`.
3. Выберите **Private**.
4. Не добавляйте README, .gitignore или лицензию (они уже есть в проекте).
5. Нажмите **Create repository**.

## 2. Загрузить проект

Распакуйте актуальный архив `SELLFORM_MVP_v2.4.0.zip`. Содержимое распакованного архива должно оказаться в корне репозитория `sellform` (папки `app`, `components`, `lib`, `supabase`, а также `package.json`). Не загружайте ZIP как единственный файл.

Самый простой способ — GitHub Desktop:
- File → Add Local Repository → выберите распакованную папку.
- Если GitHub Desktop предложит создать репозиторий в этой папке, согласитесь.
- Commit to main → Publish repository → оставьте репозиторий Private.

## 3. Импортировать в Vercel

1. Откройте https://vercel.com/new и импортируйте GitHub-репозиторий `sellform`.
2. Framework Preset: Next.js; Root Directory: `./`.
3. Build Command: `npm run build`; Install Command: `npm install`.
4. До первого production deployment добавьте переменные окружения:
   - `NEXT_PUBLIC_SUPABASE_URL`
   - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - `SUPABASE_SERVICE_ROLE_KEY` (только серверная переменная, нужна для безопасной загрузки файлов)

URL и публичный anon/publishable key можно получить в Supabase Dashboard → Project Settings → API. `SUPABASE_SERVICE_ROLE_KEY`/secret key также берётся в настройках API проекта, но его нужно добавить только в Vercel Environment Variables и локальный `.env.local`. Не публикуйте его, не помещайте в GitHub и никогда не добавляйте префикс `NEXT_PUBLIC_`.

## 4. База данных

В `supabase/migrations/` лежат миграции `0001`–`0018`. Не запускайте их вслепую на существующей базе. Сначала проверьте историю применённых миграций и текущие таблицы; затем примените только отсутствующие миграции по порядку. Сохраните резервную копию базы перед изменениями. Миграция `0011_provider_moderation.sql` сохраняет доступность ранее опубликованных профилей. Миграция `0012_role_aware_registration.sql` разрешает публичную регистрацию только ролей SELLER, AGENCY и FREELANCER. Миграция `0013_admin_order_audit_access.sql` добавляет администраторам доступ только на чтение к вложениям и истории. Миграция `0014_deliverable_submission_workflow.sql` закрывает прямой Storage INSERT для пользовательских JWT, вводит защищённую RPC регистрации файлов и требует свежий файл `DELIVERABLE` для передачи заказа на QA.

## Важно

Проект пока не подтверждён успешной production-сборкой. После подключения переменных и проверки миграций выполните deployment, затем проверьте регистрацию, вход и сценарии ролей. Внутренний финансовый ledger не является платёжной системой; эквайринг и выплаты не подключены.


## Регистрация участников

После применения миграции 0012 на странице `/login` доступны три типа регистрации: селлер, агентство и независимый исполнитель. Публичная форма не может создать администратора. Для первого ADMIN используйте доверенный SQL Editor после проверки UUID владельца аккаунта:

```sql
update public.profiles
set role = 'ADMIN', updated_at = now()
where id = '<AUTH_USER_UUID>';
```

Замените placeholder на UUID нужного пользователя из Supabase Authentication → Users. Не выдавайте ADMIN обычным участникам.

## Проверка готовности после deployment

Откройте `https://ВАШ-ДОМЕН/api/health`:
- HTTP 200 и `status: "ok"` означают, что публичная конфигурация Supabase и серверный Storage secret присутствуют.
- HTTP 200 и `status: "partial"` означают, что приложение отвечает, но `SUPABASE_SERVICE_ROLE_KEY` отсутствует; файловый API вернёт HTTP 503 при загрузке.
- HTTP 503 и `status: "not_configured"` означают, что нужно добавить `NEXT_PUBLIC_SUPABASE_URL` и `NEXT_PUBLIC_SUPABASE_ANON_KEY` в Vercel и повторить deployment.

Endpoint не проверяет подключение к PostgreSQL или применение миграций.

Это проверка доступности приложения и наличия конфигурации, **не** проверка соединения с PostgreSQL, применения миграций, авторизации или платёжной системы. Эти сценарии проверяются отдельно.

## v2.0.0 database prerequisite
Before deploying the v2.0.0 application, apply `supabase/migrations/0015_assignment_revocation_and_access.sql` in the existing Supabase project after all earlier migrations through 0014. This adds auditable assignment revocation and changes RLS/Storage authorization to require an active assignment. Back up the database first and verify the migration in a non-production environment where possible. Do not deploy the v2.0.0 UI/API against a database that has not applied migration 0015.


### v2.2.0 — финансовая приватность

Перед развёртыванием v2.2.0 примените `supabase/migrations/0016_finance_visibility_and_settlement_hardening.sql` после 0015. Резервная копия обязательна. После применения проверьте доступ под SELLER, ответственным AGENCY, назначенным FREELANCER, партнёрским AGENCY и ADMIN: полный финансовый breakdown должен быть виден только селлеру заказа, ответственному агентству и ADMIN; исполнитель должен видеть только собственные записи ledger. Не применяйте миграцию без проверки существующей истории миграций.


### v2.3.0 — лимиты исполнителей и активные назначения

Перед развёртыванием v2.3.0 примените `supabase/migrations/0017_active_provider_capacity_and_finance.sql` после 0016. Сначала сделайте резервную копию и проверьте историю миграций. Миграция заменяет `assign_order_provider()` с восстановлением проверок публикации профиля, доступности, специализации и лимита активной нагрузки; учитываются только активные назначения. Также `set_order_financial_terms()` требует активное назначение получателя выплаты на соответствующий заказ. До применения 0017 не развёртывайте v2.3.0. После применения проверьте сценарии назначения при лимите, после отзыва, при недоступном/неопубликованном профиле и при попытке указать отозванного исполнителя в финансовых условиях.


### v2.4.0 — фиксация search_path SECURITY DEFINER

Перед развёртыванием v2.4.0 примените `supabase/migrations/0018_security_definer_search_path_hardening.sql` после миграции 0017. Она закрепляет пустой `search_path` для всех текущих SECURITY DEFINER функций. Применяйте только после резервного копирования и проверки истории миграций. Это изменение не подтверждает автоматически безопасность всех RLS-политик и не заменяет интеграционные тесты под реальными ролями.
