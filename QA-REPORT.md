# Отчёт о тестировании AI-QA sandbox-test-C

**Дата:** 2026-09-28 (полный прогон ~13:40–15:10 MSK)
**Target:** https://sandbox-session-cand-c26fa443df2a4d52ba4c0f61ea385fac.fly.dev/
**Стек (наблюдаемый):** Medusa v2 + Next.js 15 storefront (Medusa Next.js Starter), Fly.io
**Issues:** https://github.com/JakovJL/AI-QA-sandbox-test-C/issues (создано 10 штук)

## Что было сделано

### Этап 1. Разведка (API + headers)
- Карта страниц и роутинга: `[countryCode]`-префиксы (/de, /dk), честные 404 на несуществующие страны/товары/поиск.
- Каталог: 4 published-товара (3 «Medusa» + Shorts; в середине дня появился сторонний «QA Test Product»), 1 регион Europe/EUR, 4 категории, 0 коллекций, 2 shipping-опции, провайдер оплаты `pp_system_default`.
- Security-обзор заголовков, cookies, кэширования (статика immutable 1 год, HTML no-store — корректно).

### Этап 2. Функциональные API-тесты
- Полный чекаут-флоу через Store API: cart → items → email → address → shipping-method → payment-collection → session → complete → **order создан** (200, type=order).
- Границы количества: 0 и −5 корректно отклонены (400); 1 000 000 принято → баг oversell.
- Auth-флоу покупателя: register → create customer → login → /me; слабые пароли приняты.
- Admin API (авторизация по выданным кредам): инвентарь, заказы, sales channels, api-keys, склады. Без токена — 401 (корректно).
- Промокод: API честно отклоняет несуществующий код (400), но UI-слой падает (см. issue #2).

### Этап 3. Функциональные UI-тесты (agent-browser, Chrome headless CDP)
- Главная, витрина, категории, PDP, корзина, чекаут, аккаунт (вход/заказы), country-switcher, промокоды.
- Каждый шаг: a11y-snapshot + DOM-eval + console + network + скриншоты (`dogfood-output/screenshots/`).

### Этап 4. Нефункциональные
- Производительность (3 замера на точку): home 275–755 мс, PDP 440–510 мс, API products 265–290 мс, cart 294–300 мс — **в норме**.
- Безопасность: security headers отсутствуют, куки без флагов, X-Powered-By раскрыт, односимвольные пароли, нет rate limiting на auth.
- Надёжность: невалидный промокод рушит Server Components render.
- SEO: soft-404 на robots/sitemap, задвоенный title.
- XSS-рефлексии в поиске/параметрах не обнаружено (поиск как страница отсутствует).
- A11y-замечания не собирались прицельно (нет подтверждённых находок).

## Подтверждённые баги (все поданы в Issues)

| # | Severity | Суть | Верификация |
|---|---|---|---|
| [#1](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/1) | critical | Чекаут без полей формы; заказать невозможно | 2 локали, DOM-замеры, API-состояние корзины |
| [#2](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/2) | high | Фейковый промокод рушит страницу корзины | 2 воспроизведения, контролируемый повтор |
| [#3](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/3) | high | Country-switcher пуст; fetch на 127.0.0.1:9001 | Сеть + консоль + разбор бандла (root cause: hardcoded baseUrl) |
| [#4](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/4) | high | Oversell: qty=1 000 000 принимается | API, 2 повтора, сверен manage_inventory/allow_backorder |
| [#5](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/5) | high | Пароль из 1 символа проходит | register + login подтверждены |
| [#6](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/6) | medium | products count = размер страницы | A/B-тест в один момент |
| [#7](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/7) | low/med | robots/sitemap отдают HTML 200 | 2 повтора, сравнение с честным 404 |
| [#8](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/8) | medium | Нет security headers; кука без флагов | Dump заголовков, document.cookie |
| [#9](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/9) | low | Дубль опции «1» в qty-селекте | DOM-замер |
| [#10](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/10) | low | Title категорий с двойным суффиксом | document.title |

## Отклонённые кандидаты (не баги / не подтверждено)
- «S /Black» без пробела — артефакт a11y-дерева, в DOM всё корректно.
- Задержка бейджа корзины — три замера 1472–2492 мс; задержка равна длительности Server Action POST (round-trip с ре-рендером, кнопка при этом корректно уходит в «Loading...»/disabled) — архитектура Next.js Server Actions, не клиентский баг. Исходные «~5s» — холодный первый заход.
- Пустые коллекции — норма для seed-данных Medusa.
- `/blocking-fault.js` — маркер песочницы.
- Производительность — в норме.

## Что осталось за кадром (следующие заходы)
- ~~Флаги куки админ-сессии~~ — проверено 2026-09-28: document.cookie не видит auth-куку, Set-Cookie от /auth/session отдаёт `connect.sid → HttpOnly; Secure; SameSite=Lax`. Флаги корректны.
- Полный a11y-аудит (axe), i18n-детали, мобильный вьюпорт, HAR-трейсы.
- Нагрузочные (k6/JMeter) — осознанно не запускались: песочница общая, риск повлиять на других.

## Файлы
- `TEST-PLAN.md` — план и правила.
- `FINDINGS.md` — журнал находок со статусами и маппингом на issues.
- `EVIDENCE/` — сырые HTTP/API-логи всех прогонов.
- `dogfood-output/screenshots/` — скриншоты UI-находок.

## Дополнение ко второму проходу (2026-09-28, вечер)

### Новые подтверждённые UI-баги (второй обход)
| Issue | Severity | Находка | Доказательство |
|---|---|---|---|
| [#11](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/11) | low | Профиль: PHONE выводит литеральный «null» | innerText + 12-profile-null-typo.png |
| [#12](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/12) | low | 4 тоста с опечаткой «succesfully» | DOM innerText (4 вхождения) |
| [#13](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/13) | low/med | html lang="en" на /de и /dk | document.documentElement.lang |
| [#14](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/14) | medium | Double-click Add to cart = два добавления (qty 1→3) | badge + Store API (qty=3) |
| [#15](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/15) | low | Пустые/шаблонные alt у изображений PDP | document.images dump |

### Скриншоты
- Все 14 PNG закоммичены в qa-evidence/ (коммиты 4f3ae50, 990f43d), raw-ссылки проверены (HTTP 200).
- Issues #1, #2, #3, #9, #10 — скриншоты в комментариях; #11, #12 — инлайн в теле; #13–#15 — текст+консольные/DOM-замеры.

### Суб-агенты
- Первая пара (API-фаззер, перекрёстный проверщик) упала на старте (~0.7 c, отчётов нет).
- Перезапущены v2 с самодиагностикой окружения; отчёты: EVIDENCE/subagent-api-fuzz.md, EVIDENCE/subagent-verify.md (статус см. FINDINGS).

### Проверено — НЕ баги (второй обход)
- Мобильная вёрстка 375px: без горизонтального переполнения (home/PDP/cart).
- Order transfer с несуществующим ID: аккуратная ошибка «Order id not found».
- Аккордеон Product Information с прочерками — seed-данные.
- /de/checkout без куки корзины — честный 404 (корзина обязательна).

## Финальный реестр issues (2026-09-28, 19 штук)

| Issue | Severity | Суть |
|---|---|---|
| [#1](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/1) | critical | Чекаут без полей формы |
| [#2](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/2) | high | Промокод рушит корзину |
| [#3](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/3) | high | Country-switcher пуст (127.0.0.1:9001) |
| [#4](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/4) | high | Oversell (порог сместился, баг жив — см. update) |
| [#5](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/5) | high | Пароль из 1 символа |
| [#6](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/6) | medium | count = размер страницы |
| [#7](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/7) | low/med | robots/sitemap soft-404 |
| [#8](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/8) | medium | Security headers + cookie flags |
| [#9](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/9) | low | Дубль «1» в qty-селекте |
| [#10](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/10) | low | Title категорий задвоен |
| [#11](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/11) | low | PHONE null в профиле |
| [#12](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/12) | low | Опечатка succesfully ×4 |
| [#13](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/13) | low/med | html lang=en на /de,/dk |
| [#14](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/14) | medium | Double-click add to cart ×2 |
| [#15](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/15) | low | Пустые alt изображений |
| [#16](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/16) | medium | limit=-1/offset=-5 → 500 |
| [#17](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/17) | medium | Невалидные email проходят регистрацию |
| [#18](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/18) | medium | quantity=1.5 принимается |
| [#19](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/19) | low/med | first_name без лимита длины |

## Cross-check вердикты (самостоятельные, вместо упавших суб-агентов)
- #4 confirmed (update-комментарий), #5 confirmed, #6 confirmed (update), #7 confirmed, #8 confirmed, #14 API-сторона confirmed.
- Отклонено при фаззинге (корректное поведение): qty строкой, битый JSON, невалидный email корзины, country_code вне региона, fake shipping option, промокоды-мусор, double-slash, fake cart, double complete (идемпотентно).

## Инцидент
- К 15:49 админ-пароль admin@sandbox.local перестал работать — песочница, вероятно, ротирована владельцем. Cross-check #4 по инвентарю через admin недоступен; факт фиксирован по API-ответам.

## Не выполнено (причина)
- Суб-агентский прогон (API-фаззер/проверщик): обе попытки падают с 403 «GLM-5.3 free quota used up» (модель суб-агентов исчерпана). Все проверки выполнены основным агентом с теми же лимитами мутаций.

## Дополнение: Admin UI (третий проход, 2026-09-28 вечер)

| Issue | Severity | Находка |
|---|---|---|
| [#20](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/20) | low/med | redacted-маска ключа не совпадает с токеном |
| [#21](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/21) | low/med | Пустая страница деталей workflow |

Скриншоты: 15-admin-store-settings.png, 16-admin-store-edit.png, 17-workflow-detail-empty.png (коммит 99f1c41).
Позитивные подтверждения: валидация пустого имени стора работает; admin-кука HttpOnly; order detail функционален.

**Итог: 21 issue** (1 critical, 4 high, 7 medium, 9 low/low-med).
