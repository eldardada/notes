## Пошаговый план действий

### Фаза 1 — Быстрые фиксы (1–3 дня)

**Шаг 1. Статические ресурсы через прокси**
- Файлы: wildflow-desktop/src/main/services/wbAccountOpener/network.ts и wb-runner/src/network.ts
- Сейчас `.json`, `.css`, `.png` и т.д. с доменов WB загружаются через Node.js `fetch()` **без прокси** → TLS-отпечаток Node.js вместо Chrome. Это главный вектор детекта.
- **Что делать:** Убрать перехват статики для WB-доменов — пусть браузер грузит их сам через прокси. Оставить `fetch()` только для внешних CDN (fonts.googleapis.com и т.п.).

**Шаг 2. Задержка auth slide**
- Те же файлы: `network.ts` в обоих проектах
- Сейчас `route.fulfill()` на `wbx-auth.wildberries.ru/v2/auth/slide-v3` отвечает за 0ms. WB видит мгновенный ответ.
- **Что делать:** Добавить `await new Promise(r => setTimeout(r, 150 + Math.random() * 450))` перед `route.fulfill()`.

**Шаг 3. Объединить CDP-сессии**
- Файл: wildflow-desktop/src/main/services/wbAccountOpener.ts
- Сейчас создаются 3 параллельных CDP-сессии (orderTracking, antibot, network). Множественные DevTools-клиенты — аномалия.
- **Что делать:** Создать одну CDP-сессию и передавать её во все три модуля.

**Шаг 4. Обфусцировать маркеры подсветки**
- Файл: wildflow-desktop/src/main/services/wbAccountOpener/highlighter.ts
- Глобальная переменная `__wfHighlightObserverInstalled` и CSS-класс `wf-highlight-product-card` — тривиально находятся скриптами WB.
- **Что делать:** Заменить на случайные имена, генерируемые при каждом запуске (например, `_h${crypto.randomUUID().slice(0,8)}`).

**Шаг 5. Установить viewport, locale, timezone**
- Файлы: `browser.ts` в desktop и wb-runner
- Desktop: viewport не задан → зависит от размера окна. wb-runner: `defaultViewport: null` → 800×600.
- **Что делать:** Задать реалистичный viewport (1920×1080 или рандом из популярных), `locale: 'ru-RU'`, `timezoneId: 'Europe/Moscow'`.

**Шаг 6. Не блокировать Sentry WB**
- Файлы: `network.ts` в обоих проектах
- Сейчас запросы на `sentry`/`metrics` фильтруются как "benign failures". Отсутствие телеметрии — сигнал для WB.
- **Что делать:** Пропускать эти запросы, не фильтровать.

---

### Фаза 2 — Средний приоритет (1–2 недели)

**Шаг 7. Унифицировать wb-runner на Patchright**
- wb-runner использует Puppeteer + stealth-плагин, а desktop — Patchright. Один аккаунт видит два разных браузерных отпечатка.
- **Что делать:** Перевести wb-runner на Patchright (headless) для единообразия fingerprint'а.

**Шаг 8. Динамический `x-spa-version`**
- Файл: axios-wb-http.client.ts
- Захардкожен `'14.0.7'` — устаревает и становится маркером бота.
- **Что делать:** Парсить актуальную версию из HTML/JS главной страницы WB раз в час и кэшировать.

**Шаг 9. Валидация IP в antibot-токене**
- `WbAasTokenInfo` хранит IP, с которого был получен токен. Если API-запросы идут с другого IP — несовпадение.
- **Что делать:** В `pre-open-wb-account.use-case.ts` сравнивать `antibotToken.ip` с `pinnedExternalIp`. При расхождении — требовать перевыпуск токена (`needChallenge = true`).

**Шаг 10. Warm-up навигация**
- Файл: `wbAccountOpener.ts`
- Сейчас сразу `goto('/lk')` — нет человеческого поведения.
- **Что делать:** Сначала `goto('/')` → ждать 2–5 секунд → скролл → `goto('/lk')`. Для wb-runner аналогично (уже есть home→orders, но задержка 0.8–1.5s слишком мала — увеличить до 3–8s).

**Шаг 11. CDP-запись в localStorage вместо `addInitScript`**
- Файлы: `account.ts` в обоих проектах
- `addInitScript` выполняется ДО скриптов страницы — WB может детектить, что localStorage заполнен до первого `DOMContentLoaded`.
- **Что делать:** Использовать `Runtime.evaluate` через CDP для записи в localStorage после загрузки страницы, либо `page.evaluate()` с timing'ом.

**Шаг 12. Контекстный Referer**
- В `axios-wb-http.client.ts`: хардкод `Referer: '/lk/myorders/delivery'` на все запросы.
- **Что делать:** Передавать актуальный Referer в зависимости от endpoint'а (список заказов → `/lk/myorders`, фидбэк → `/lk/discussion`).

---

### Фаза 3 — Долгосрочные улучшения (1+ месяц)

**Шаг 13. IP drift → блокировка**
- Сейчас при смене IP просто логируется `wb_ip_drift_detected`, но аккаунт всё равно открывается.
- **Что делать:** При drift — не открывать, перезапросить прокси с тем же session ID (вернёт тот же IP при sticky session). Если не помогает — пересоздать сессию и обновить `pinnedExternalIp`.

**Шаг 14. ISP-прокси для ключевых аккаунтов**
- Residential proxies всё чаще попадают в серые списки. ISP-прокси (датацентр с ASN провайдера) стабильнее.
- Для самых ценных аккаунтов — выделенные ISP-прокси.

**Шаг 15. Прогрев аккаунтов**
- Перед первым buyback-заказом аккаунт должен иметь историю: просмотры, добавление в корзину, real-покупки.
- Создать систему «warm-up сессий» — автоматический просмотр каталога и категорий.

**Шаг 16. Кэширование статических ресурсов**
- После фикса шага 1, частые загрузки одного и того же `.css`/`.js` через прокси — аномалия.
- **Что делать:** Локальный кэш с респектом `Cache-Control` заголовков.
