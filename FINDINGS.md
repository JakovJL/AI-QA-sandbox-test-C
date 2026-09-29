# FINDINGS — черновик наблюдений

| # | Наблюдение | Статус | Методы | Свидетельство |
|---|-----------|--------|--------|---------------|
| R1 | /robots.txt отдаёт text/html 200 (30922 байт) вместо текстового robots.txt | candidate | HTTP GET | EVIDENCE/recon1.txt |
| R2 | /sitemap.xml отдаёт text/html 200 (30934 байт) вместо XML | candidate | HTTP GET | EVIDENCE/recon1.txt |
| R3 | /store/collections пуст (count=0?) | info — сверить с админкой | HTTP GET | EVIDENCE/recon1.txt |

## UI findings (dogfood, 2026-09-28)
| # | Наблюдение | Статус | Свидетельство |
|---|-----------|--------|---------------|
| U1 | Checkout: страница отображает шаги как пройденные (галочки) но 0 input-полей; адрес/email/доставка/оплата ввести нельзя; Continue to review ничего не делает; корзина за чекаутом без email/addr/shipping/payment | confirmed (UI x2 locales de/dk + API state) | 03-checkout.png, eval logs |
| U2 | Невалидный промокод на /cart -> Server Components render error, рушится вся страница (нужно перезагружать) | confirmed (de, воспроизведён) | 04-promo-crash.png |
| U3 | Селект количества: опция "1" дублируется (12 опций вместо 11) | confirmed (DOM eval) | sel-check eval |
| U4 | Вариант в PDP/cart отображается как "S /Black" (пропущен пробел) | confirmed (a11y tree) | snapshot |
| U5 | Title категорий дублирует суффикс: "Shirts | Medusa Store | Medusa Store" | confirmed (get title) | eval |
| U6 | Cart badge обновляется с задержкой после Add to cart: замеры 2492/1641/1472 мс — задержка равна длительности Server Action POST (2480/1630/1440 мс по resource timing) | resolved — не баг | timeline + resource timing |
| U7 | Console-скрипт /blocking-fault.js — маркер песочницы (не баг продукта) | info | script text |

## API findings — verification status (2026-09-28)
| # | Наблюдение | Статус | Свидетельство |
|---|-----------|--------|---------------|
| A1 | GET /store/products: count = размер страницы, а не общее число товаров (limit=1 -> count=1; limit=100 -> count=4) | confirmed (A/B same-moment) | pagination-check.txt |
| A2 | Quantity=1_000_000 принимается в корзину (subtotal 10,000,000 EUR), нет проверки против inventory (manage_inventory=true, backorder=false) | confirmed (API, 2 повтора) | qty-bounds.txt |
| A3 | Регистрация покупателя с паролем «1» успешно создаёт аккаунт (login с этим паролем работает) | confirmed (register+login) | auth-price3.txt |
| A4 | /robots.txt и /sitemap.xml отдают HTML 404-страницу со статусом 200 (soft-404) | confirmed (2 повтора) | recon2.txt |
| A5 | Security-заголовки отсутствуют: нет CSP, HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy; X-Powered-By: Next.js раскрыт | confirmed (headers dump) | recon2.txt |
| A6 | Кука _medusa_cache_id без HttpOnly/Secure/SameSite | confirmed (Set-Cookie dump) | recon2.txt |
| A7 | Admintoken в куки? _medusa_admin_http не виден в document.cookie — добавить проверку | partially verified | — |

## Соответствие findings → GitHub Issues (созданы 2026-09-28)

| Issue | Findings | Title |
|---|---|---|
| [#1](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/1) | U1 | [Critical] Checkout: нет полей формы, шаги помечены выполненными, заказать невозможно |
| [#2](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/2) | U2 | [High] Невалидный промокод рушит страницу корзины |
| [#3](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/3) | — | [High] Переключатель страны: пустой список, запрос на 127.0.0.1:9001 |
| [#4](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/4) | A2 | [High] quantity=1000000 принимается без проверки склада (oversell) |
| [#5](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/5) | A3 | [High] Регистрация принимает односимвольный пароль |
| [#6](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/6) | A1 | [Medium] /store/products: count = размер страницы |
| [#7](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/7) | A4 | [Low/Medium] robots.txt/sitemap.xml — soft-404 |
| [#8](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/8) | A5+A6 | [Medium] Security-заголовки + флаги куки |
| [#9](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/9) | U3 | [Low] Дублирование опции «1» в селекте количества |
| [#10](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/10) | U5 | [Low] Задвоенный title категорий |

## Отклонённые / не поданные

| Наблюдение | Причина отказа |
|---|---|
| U4 «S /Black» без пробела | Артефакт a11y-дерева headless-браузера; в DOM текст «S / Black» корректен |
| U6 Задержка бейджа корзины | Повторные замеры (2026-09-28 17:00): 2492/1641/1472 мс; задержка = длительность Server Action POST (performance resource timing совпадает с точностью ~10–40 мс): бейдж обновляется только после завершения re-render round-trip. Кнопка при этом корректно блокируется: через 150 мс после клика — «Loading...», disabled=true, после ответа восстанавливается. Вывод: архитектура Next.js Server Actions (обновление серверного состояния), не клиентский баг; исходные «~5s» — холодный первый заход + гидрация. Пограничное UX-замечание: POST 1.4–2.5s в песочнице заметен, но double-submit заблокирован |
| A7 Админ-кука в document.cookie | Закрыто (2026-09-28 17:00), два свидетельства: (1) UI — document.cookie в /app содержит только lng и _medusa_cache_id, auth-кука не видна; (2) API — Set-Cookie от /auth/session: connect.sid → HttpOnly; Secure; SameSite=Lax — все флаги корректны. Не баг |
| Производительность | home 275–755ms, PDP 440–510ms, API 265–290ms — в норме для холодной песочницы |
| /store/collections пуст | В эталонном Medusa seed коллекции не создаются — не баг |
| U7 /blocking-fault.js | Специальный маркер песочницы («you found a bug») — не дефект витрины |

## Особые наблюдения

- В каталоге в середине сессии появился товар «QA Test Product 2026-09-28 €5.00» (handle qa-test-product-2026-09-28), созданный НЕ в рамках этого прогона — песочницей пользуются параллельно. Это объясняет часть «плавающих» наблюдений по каталогу; ключевые проверки сделаны с контролем состояния.
- В песочнице искусственно занижен инвентарь/счётчики? Нет: надёжные подтверждения сделаны A/B-методами в один момент времени.

## Second UI pass (дополнительный обход, 2026-09-28 ~15:20–15:40)
| # | Наблюдение | Статус | Свидетельство |
|---|-----------|--------|---------------|
| U8 | Профиль: 4 шаблонных тоста с опечаткой "succesfully" (Name/Email/Phone/Billing address) | confirmed (DOM innerText) | 12-profile-null-typo.png |
| U9 | Профиль: PHONE-секция выводит литеральный "null" при пустом телефоне | confirmed (innerText + screenshot) | 12-profile-null-typo.png |
| U10 | html lang="en" на всех страницах, включая /de и /dk (несоответствие языка контента) | confirmed (document.documentElement.lang) | eval log |
| U11 | Product images: у миниатюр галереи пустые alt="" (без описания), у главных alt="Product image N" (неинформативно) | confirmed (document.images dump) | eval log |
| U12 | Add to cart double-click: оба клика обработаны (qty 1→3 за один даблклик), юзер может случайно удвоить позицию | confirmed (badge+API: qty=3) | cart API state |
| U13 | Аккордеон Product Information: поля Material/Country/Type/Dimensions = "-", Weight 400g — seed-данные, не баг | info | eval log |
| U14 | Мобильная вёрстка 375px: без горизонтального переполнения на home/PDP/cart | OK — не баг | 13/14 screenshots |
| U15 | Order transfer с несуществующим ID: аккуратное "Order id not found" | OK — не баг | eval log |

## Self-run API fuzz + cross-check (2026-09-28 15:41–15:52, вместо упавших суб-агентов)

### Новые баги (подтверждены 2+ прогонами)
| # | Находка | Свидетельство |
|---|---------|---------------|
| N1 | GET /store/products?limit=-1 → 500; ?offset=-5 → 500; /store/product-categories?limit=-1 → 500 (отсутствие нижней границы валидации пагинации) | fuzz-readonly.txt, repro ×2 |
| N2 | POST /auth/customer/emailpass/register принимает email «qa-main-1@», «a b@example.com», «@example.com» — аккаунты реально создаются (login 200) | fuzz-mutations3/4.txt |
| N3 | POST /store/carts line-items принимает quantity=1.5 (дробное количество) | fuzz-mutations1.txt (qty=102 после 1.5+100 → 1.5+... агрегация), повтор через 100 |
| N4 | shipping_address.first_name=10000 символов принимается без ограничения длины | fuzz-mutations3.txt (stored len≈10000) |

### Корректное поведение (НЕ баги)
- qty строкой «2» → 400; битый JSON → 400; cart email=notanemail → 400; country_code=us/zz → 400; fake shipping option → 400; promo [''] и 5000-симв. → 400; //store/products → 404; region_id=garbage → 400; fake cart → 404; /store/orders без auth → 404; limit=0 → 200 пусто.
- Double complete: второй POST /complete возвращает ТУ ЖЕ запись заказа (тот же order id), второй заказ не создаётся — идемпотентно. НЕ баг.

### Cross-check ранее поданных issues
| Issue | Вердикт | Факт |
|---|---|---|
| #4 oversell | confirmed с уточнением | qty=1000000 теперь 400 (задепleted stocks или hotfix за день), но 100000/10000/1000 всё ещё принимаются → oversell жив, порог изменился |
| #5 weak password | confirmed | register+login с паролем «1» → 200/200 на свежем аккаунте |
| #6 count | confirmed | limit=1→count=1, limit=100→count=5 (в каталоге 5 товаров); categories limit=2 → count=4 при ret=2 |
| #7 soft-404 | confirmed | robots/sitemap 200 text/html; контрольный /zzz → 404 |
| #8 headers | confirmed | Все 6 security-заголовков отсутствуют; X-Powered-By=Next.js; кука без HttpOnly/Secure/SameSite |
| #14 double-click | API-сторона подтверждена | последовательные добавления агрегируются (клиентская гонка остаётся причиной UI-дубля) |

### Инцидент
- Админ-пароль admin@sandbox.local/supersecret перестал работать к 15:49 (Invalid email or password) — вероятно, владелец песочницы ротирировал доступы или sandbox пересобран. Влияние: admin-проверки ограничены до повторной выдачи кредов.

## Admin UI-обход (/app) — 2026-09-28 16:00–16:10

### Контекст
- Логин admin@sandbox.local работает (ранний «Invalid email or password» в 15:49 был разовым сбоем/кратковременной ротацией — повторный вход успешен и через UI, и через API).
- Админ-сессия: токен в куки не виден из document.cookie → HttpOnly у admin-куки стоит (хорошо), куки витрины видны как раньше.

### Осмотрено
- Settings → Store: name/currency/region/channel; форма /edit открывается; Save с пустым Name НЕ применяет изменения (имя осталось «Default Store») — валидация есть (либо клиентская, либо сервер отклонил). Данные: currencies eur+usd, tax inclusive pricing=False у обеих, Metadata: 0 keys, «JSON 9 keys» (виджет, детали не раскрылись).
- Settings → Regions: 1 регион Europe (7 стран), провайдер «System (DEFAULT)».
- Settings → Publishable API Keys: 1 ключ Active (замаскирован pk_085•••5e3 — НО выданный нам ключ начинается с pk_2044...! См. находку A2 ниже).
- Settings → Secret API Keys: пусто (0).
- Settings → Users: 1 админ (Sergei hi / admin@sandbox.local).
- Settings → Workflows: история complete-cart; состояния Done/Reverted; прогонов «8 of 22 steps» несколько (это прогоны, где корзина была без оплаты/адреса — наш «мёртвый чекаут» и double-complete тесты). Один workflow «Reverted 4/22».
- Workflow DETAIL страница: пустая (только навигация, 276 символов, воспроизведено 2 раза) — кандидат в баги рендера.
- Order detail: работает; payment Pending, «ready to be captured».

### Находки-кандидаты
| # | Наблюдение | Статус |
|---|-----------|--------|
| A1 | Workflow detail страница /app/settings/workflows/{transaction_id} рендерится пустой (2 повтора) | confirmed-UI |
| A2 | Publishable key в админке маскирован как «pk_085•••5e3», а фактический выданный ключ — «pk_2044…7186»: маска НЕ соответствует реальному ключу (неправильное превью), либо в системе два ключа и старый не удалён/не показан | candidate — требует проверки через admin API list |
| A3 | Кнопки Capture payment/фи statements на заказе не тестировались (мутации денег в общей песочнице) | out of scope |

### Изменений в магазине не вносилось (только Cancel-пути и read-only).

## Admin UI findings (2026-09-28 16:00–16:15)
| # | Наблюдение | Статус | Issue |
|---|-----------|--------|-------|
| A1 | Workflow detail страница пустая (2 повтора) | confirmed-UI | [#21](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/21) |
| A2 | redacted-маска ключа не совпадает с токеном (pk_085***5e3 vs pk_204…7186) | confirmed (API json) | [#20](https://github.com/JakovJL/AI-QA-sandbox-test-C/issues/20) |
| A3 | Save с пустым Name не применяется (валидация работает) | OK — не баг | — |
| A4 | Admin session cookie HttpOnly (не видна в document.cookie) | OK — подтверждено вторым методом 17:00: Set-Cookie connect.sid → HttpOnly; Secure; SameSite=Lax (см. A7-резолюцию) | — |
| A5 | Orders detail: payment Pending, «ready to be captured» — работает | OK | — |
| A6 | Ранний сбой входа 15:49 (Invalid email or password) прошёл к 16:00 — transient | info | — |

Примечание: adминки UI — новый слой тестирования, покрыт Settings разделы Store/Regions/API Keys/Users/Workflows + Orders detail. Мутации в админке не выполнялись (общая песочница).
