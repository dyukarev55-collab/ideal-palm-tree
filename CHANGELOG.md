# Changelog

## 1.9.0 — улучшение назначения исполнителей
- Повторно назначенные участники исключаются из списка кандидатов в карточке заказа, чтобы не провоцировать заведомо конфликтующий запрос.
- В списке назначений вместо UUID показываются имя/компания, понятный тип участия, сумма и дата назначения.
- Ошибки назначения переводятся в безопасные пользовательские сообщения; технические коды БД остаются только в серверном журнале.
- Добавлены модульные тесты фильтрации кандидатов и отображения назначений.
- Новая SQL-миграция не требуется.


## 1.8.0
- Добавлена ранняя проверка UUID для всех API-маршрутов заказов, включая идентификаторы файлов.
- Схемы назначения исполнителей, смены статуса и отзывов переведены на строгую проверку входных полей.
- Для суммы назначения добавлены проверка конечного числа, верхняя граница NUMERIC(12,2) и ограничение до двух знаков после запятой.

## 1.7.0 — рабочие настройки B2B-аккаунта
- Страница `/settings` заменена с демонстрационных заглушек на рабочее редактирование имени и компании/бренда.
- Добавлен защищённый API `/api/profile` для чтения и обновления собственного профиля.
- API принимает только `full_name` и `company_name`; роль, идентификатор аккаунта и права доступа нельзя изменить через этот endpoint.
- Email и тип кабинета отображаются только для чтения; добавлен выход из аккаунта.
- Главная `/` перенаправляет в единое ролевое рабочее пространство `/workspace`, вместо отдельной общей панели.
- Новых SQL-миграций не требуется: обновление использует существующие ограничения RLS и права на изменение колонок профиля из миграции 0003.

## 1.6.0 — сдача результата и закрытая загрузка файлов
- Вложения разделены на техническое задание (`BRIEF`), исходные материалы (`REFERENCE`) и готовый результат (`DELIVERABLE`).
- API проверяет роль и участие в заказе; PostgreSQL повторно проверяет права в `register_order_file()` и статусной RPC.
- Прямая загрузка в приватный Storage через обычный пользовательский JWT закрыта; серверная загрузка использует только `SUPABASE_SERVICE_ROLE_KEY`.
- Перед передачей заказа в QA требуется новый `DELIVERABLE`, загруженный после последнего перехода в `IN_PROGRESS`.
- Селлер может принять результат или вернуть заказ на доработку; после возврата требуется загрузить новую версию результата.
- Назначенный фрилансер или партнёрское агентство может отправить на QA только назначенную ему работу; управление основной очередью остаётся у ответственного агентства.
- Добавлены модульные тесты правил переходов и прав на типы файлов.

## 1.5.1 — проверка содержимого вложений
- Загрузка вложений теперь проверяет соответствие расширения и MIME-типа, а также сигнатуру содержимого (magic bytes) для PDF, JPEG, PNG, WebP, Office, ZIP и текстовых файлов.
- Добавлена ранняя проверка `Content-Length` для отклонения заведомо слишком больших multipart-запросов до разбора формы.
- Лимит файла 20 МБ, приватное хранилище и действующие правила доступа сохранены.

# Changelog

## 1.5.0 — административный аудит заказов
- Исправлено ограничение RLS: администратор мог открыть карточку заказа, но не мог прочитать метаданные вложений и историю статусов.
- Добавлена миграция `0013_admin_order_audit_access.sql`: ADMIN получает доступ только на чтение к этим двум разделам.
- Права на загрузку, изменение и удаление материалов/истории не расширялись; бизнес-модель и финансовая логика не менялись.
- Документация и версия пакета обновлены до 1.5.0.

## 1.2.0 — Модерация каталога и аудит действий
- Добавлены статусы модерации профилей исполнителей и партнёрских агентств: `PENDING`, `APPROVED`, `REJECTED`.
- Новые и изменённые публичные данные профиля требуют повторного одобрения; уже опубликованные до обновления профили сохраняют доступность.
- Каталог и назначение на заказы допускают только одобренные и опубликованные профили.
- Добавлена закрытая административная очередь проверки, одобрение/отклонение с обязательной причиной отказа и журнал решений.
- Модерация выполняется транзакционной SECURITY DEFINER RPC; владельцы профилей не могут изменять поля статуса модерации напрямую.
- Добавлена миграция `0011_provider_moderation.sql`.

## 1.1.0 — Административная панель
- Добавлена отдельная закрытая страница `/admin` с проверкой роли `ADMIN` на сервере.
- Добавлена сводка по аккаунтам и ролям, статусам заказов, последним заказам и новым профилям.
- Финансовые начисления показываются раздельно по валютам; интерфейс явно указывает, что ledger не подтверждает реальные платежи.
- Добавлена защищённая RPC `get_admin_dashboard_stats()` с проверкой роли администратора и агрегированными данными без выдачи финансовых записей пользователей.
- Администратор автоматически перенаправляется из `/workspace` в `/admin`; маршрут `/admin` защищён middleware.
- Добавлена миграция `0010_admin_dashboard.sql`.

## 1.0.0 — Уведомления внутри платформы
- Добавлена таблица уведомлений с RLS: пользователь читает только собственные события.
- Уведомления автоматически создаются PostgreSQL-триггерами при новой заявке, смене статуса, назначении исполнителя, загрузке файла, отзыве и финансовом начислении.
- Добавлены защищённые RPC для отметки одного или всех уведомлений прочитанными.
- Добавлен API `/api/notifications` и блок уведомлений в рабочем пространстве.
- Дедупликация событий выполняется уникальным ключом на уровне БД.
- Email, push и realtime-доставка не входят в этот этап.

## 0.9.0 — Финансовый контур
- Добавлены финансовые условия заказа: сумма клиента, выплата назначенному исполнителю, ставка/сумма комиссии SELLFORM и остаток агентства.
- Добавлена миграция `0008_financial_ledger.sql` с RLS и транзакционными RPC.
- Начисления по заказу формируются только после `COMPLETED`; повторный вызов не создаёт дубликаты.
- После создания проводок финансовые условия заказа заблокированы от изменения.
- Добавлены финансовый блок в карточке заказа и кабинет `/finance`.
- В документации явно отмечено, что это внутренний учёт начислений, а не фактический приём/перевод денег.

## 0.8.0 — Управление загрузкой исполнителей
- Добавлен лимит активных заказов и транзакционная проверка доступности при назначении.
- В каталоге отображается агрегированная загрузка.

## 0.7.0 — Рейтинг и отзывы
- Добавлены отзывы по завершённым заказам и агрегированный рейтинг исполнителей.

## 0.6.0 — Каталог исполнителей
- Добавлены профили исполнителей и партнёрских агентств, API и каталог `/team`.

## 0.5.0 — Файлы заказов
- Добавлена загрузка файлов в приватное хранилище Supabase и скачивание через временные ссылки.


## 1.3.0 — Регистрация трёх типов B2B-аккаунтов

- Добавлена регистрация селлеров, агентств и независимых исполнителей из формы входа.
- При регистрации передаются имя, компания/бренд и выбранная роль; существующие аккаунты не меняются.
- Добавлена миграция `0012_role_aware_registration.sql`: серверный триггер принимает только `SELLER`, `AGENCY` и `FREELANCER`; роль `ADMIN` через публичную регистрацию назначить нельзя.
- После успешного входа пользователь возвращается на безопасный внутренний адрес `next` либо в рабочее пространство.
- Добавлена конфигурация ESLint flat config, чтобы команда `npm run lint` имела конфигурацию.
- Сборка и runtime-проверки требуют установки зависимостей и подключения проекта к Supabase/Vercel.

## 1.4.0 — production configuration readiness

- Centralized validation of the public Supabase URL and anon key for browser, server and middleware clients.
- Middleware now returns a controlled HTTP 503 configuration response instead of failing while constructing a Supabase client when deployment variables are missing.
- Added `GET /api/health` for application liveness and public Supabase-configuration readiness. The endpoint deliberately does not claim that the database is reachable.
- Added `npm run typecheck` for repeatable TypeScript validation.

## 2.0.0 — 2026-11-17
- Added auditable assignment revocation: old assignment rows are retained with revocation actor, timestamp and optional reason.
- Added transactional `revoke_order_provider` RPC and authenticated DELETE API endpoint.
- Reassignment is supported by a partial unique index that permits one active assignment per provider/order while retaining revoked history.
- Updated order, file metadata, Storage, status-history and financial-terms RLS read rules so revoked providers lose access to the order and its files.
- Updated file registration and status workflow RPCs to require active assignment membership.
- Added assignment management UI and made revoked providers eligible for a new assignment.
- Added migration-contract and assignment-helper tests.
- Migration `0015_assignment_revocation_and_access.sql` must be applied to Supabase before using the new assignment management endpoint.

## 2.1.0
- Centralized Supabase/PostgreSQL error translation for API responses; database internals are no longer returned to clients from the affected routes.
- Added stable HTTP mappings for authorization, conflict, validation and not-found errors, with generic fallback messages for unknown failures.
- Hardened order creation against malformed JSON bodies (returns HTTP 400 instead of an unhandled route error).
- Added unit tests for public API error mapping.


## 2.2.0 — role and finance access hardening
- Restricted full order financial terms to the order-owning seller, responsible agency and administrators; assigned providers can read only their own beneficiary-scoped ledger entries.
- Replaced the settlement RPC with a `SECURITY DEFINER` implementation using an empty `search_path` and explicit responsible-agency/admin authorization.
- Removed raw Supabase Storage and file-registration error messages from upload API responses; detailed diagnostics remain server-side.
- Added migration-contract tests for financial visibility and settlement authorization.
- Added migration `0016_finance_visibility_and_settlement_hardening.sql`; apply after 0015 before deploying v2.2.0.


## 2.3.0 — 2026-11-17

- Restored transactional provider-capacity enforcement in the assignment RPC after assignment revocation was introduced. Only ACTIVE assignments in REVIEW, IN_PROGRESS, and QA consume provider capacity.
- Assignment now re-checks provider publication, availability, service specialty, role, and configured capacity while locking the provider profile to serialize concurrent assignments.
- Financial terms now require the payout beneficiary to have an ACTIVE assignment on the same order; the assignment row is locked to serialize the check against concurrent revocation.
- Hardened both SECURITY DEFINER functions with an empty `search_path` and authenticated-only execution.
- Added migration-contract tests for provider capacity and active-assignment finance validation.
- Added migration `0017_active_provider_capacity_and_finance.sql`; apply after 0016 before deploying v2.3.0.


## 2.4.0 — SECURITY DEFINER search_path hardening
- Added migration `0018_security_definer_search_path_hardening.sql` to pin all current SECURITY DEFINER RPC and trigger functions to an empty `search_path`.
- Application relations are schema-qualified; PostgreSQL built-ins remain available from `pg_catalog`.
- Added migration-contract tests covering the complete current SECURITY DEFINER function inventory.
- Updated setup and deployment documentation; migration 0018 must be applied after 0017.
- This is configuration hardening, not a substitute for executing migrations and role-based integration tests in Supabase.
