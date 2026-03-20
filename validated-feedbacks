# Слабые места предыдущего плана + усиленные рекомендации

## Что я пропустил или неправильно оценил

### Слабость 1: «Просто убрать fetch() для статики» — наивное решение

**Предыдущая рекомендация:** заменить `fetch()` на `route.continue()`.

**Почему это слабо:** Код загружает статику через `fetch()` **намеренно**. В network.ts и в wbAccountRegistrator.ts статика проксируется через Node.js `fetch()` чтобы **снизить нагрузку на прокси-трафик**. Резидентские прокси Proxyma тарифицируются по трафику. CSS/PNG/WOFF2 — основная масса трафика.

**Но:** Это создаёт **расщепление TLS-отпечатков**. Запросы через прокси идут от Chrome (TLS = Chrome JA3), а `fetch()` из Node.js Electron — это `undici`/Node.js TLS со своим JA3-хэшем. WB может видеть два разных TLS-клиента на одном URL-домене.

**Усиленная рекомендация:**

Вместо `route.continue()` (весь трафик через прокси, дорого) или `fetch()` (разные TLS fingerprints) — **кешировать статические ресурсы локально**:

```typescript
const staticCache = new Map<string, { status: number; headers: Record<string, string>; body: Buffer }>();

await page.route(STATIC_ASSET_URL_REGEX, async (route, request) => {
    const url = request.url();
    const cached = staticCache.get(url);
    if (cached) {
        await route.fulfill(cached);
        return;
    }
    // Первый раз — через прокси (route.continue), сохранить в кэш
    // Или: route.continue() всегда, а кэш на уровне proxy provider
    await route.continue();
});
```

Если экономия трафика критична — сделать выборочно: `.woff2`, `.png`, `.jpg` качать через `fetch()` (их WB не контролирует по TLS — это CDN), а `.json`, `.css` с домена `wildberries.ru` — через прокси (`route.continue()`).

---

### Слабость 2: Не учёл серверный wb-runner и его проблемы

**Что пропустил:** В wildflow-api есть отдельный сервис wb-runner — **серверный headless Puppeteer** для решения antibot challenge. Он использует:
- **Puppeteer** (не Patchright!) + `puppeteer-extra-plugin-stealth`
- Headless-режим (`headless = true`)
- Нет viewport настройки (`defaultViewport: null`)
- Нет user-agent кастомизации при запуске
- **Тот же паттерн** загрузки статики через `fetch()` без прокси
- CDP сессии для перехвата antibot-токена

**Критическая проблема wb-runner:**
- Puppeteer Stealth плагин **устаревает** — WB это знает
- Headless режим **детектируется** даже со Stealth (through `navigator.webdriver`, `Chrome.Runtime`, протокол headless Chrome)
- `--no-sandbox --disable-setuid-sandbox` — маркер серверного окружения
- Отсутствие `defaultViewport` = viewport `800x600` (стандарт Puppeteer) — **никто не сидит на 800x600**
- User-Agent устанавливается через `page.setUserAgent()` **после** `evaluateOnNewDocument` — может быть рассинхрон

**Усиленная рекомендация:**

wb-runner — отдельный вектор детекта, который я полностью пропустил. Рекомендации:

1. **Заменить Puppeteer на Patchright** (уже используется в desktop) или **Camoufox** (уже используется для Ozon в проекте ozon)
2. Установить реалистичный viewport: `{ width: 1360, height: 768 }` с рандомизацией
3. Использовать `headless: 'shell'` (Puppeteer) или `headless: false` с Xvfb
4. Убрать `--no-sandbox` на продакшне (use user namespaces instead)
5. Установить user-agent **до** навигации и через `--user-agent=` аргумент, а не через `page.setUserAgent()`

---

### Слабость 3: Множественные CDP сессии — детектируемый паттерн

**Что пропустил:** В wbAccountOpener.ts создаются **3 параллельных CDP-сессии** на одну страницу:

1. `setupOrderSubmissionTracking` → `page.context().newCDPSession(page)` + `Network.enable` (orderTracking.ts)
2. `setupAntibotTokenForwarding` → `page.context().newCDPSession(page)` + `Network.enable` (network.ts)
3. Patchright сам по себе использует CDP для управления страницей

**Риск:** Каждый `newCDPSession` создаёт отдельное DevTools подключение. WB может детектировать это через:
- `Performance.getMetrics()` — аномально высокое потребление
- Подсчёт подключённых клиентов к DevTools WebSocket
- Задержки из-за конкурирующих Network.enable

**Усиленная рекомендация:**

Объединить orderTracking и antibotTokenForwarding в **одну CDP-сессию**:

```typescript
const cdpClient = await page.context().newCDPSession(page);
await cdpClient.send('Network.enable');

// Один обработчик для всех событий
cdpClient.on('Network.responseReceived', async (params) => {
    handleAntibotResponse(params);
    handleCartTracking(params);
    handleOrderTracking(params);
});
```

Это уменьшит количество CDP-сессий с 3 до 2 (Patchright + 1 кастомная).

---

### Слабость 4: `addInitScript` / `evaluateOnNewDocument` — инъектируемые скрипты детектируемы

**Что пропустил:** Код использует `page.addInitScript()` (Patchright) и `page.evaluateOnNewDocument()` (Puppeteer в wb-runner) для:

1. Установки `localStorage` токена — account.ts (desktop)
2. Установки `localStorage` deviceId
3. Подсветки товаров (highlighter) — highlighter.ts

**Риски:**
- `addInitScript` выполняется **до** любого другого скрипта страницы. WB может засечь порядок выполнения (localStorage уже заполнен до загрузки их скриптов)
- Highlighter инъектирует стиль с уникальным классом `.wf-highlight-product-card` + глобальную переменную `__wfHighlightObserverInstalled` на `window` — WB может вызвать `document.querySelectorAll('[class*="wf-"]')` или проверить `window.__wfHighlightObserverInstalled`
- MutationObserver на `body` с `subtree: true` — тяжёлый обсервер, может быть засечён через performance

**Усиленная рекомендация:**

1. **Переименовать маркеры** — не использовать очевидные имена. Вместо `wf-highlight-product-card` → рандомный хэш на каждый запуск: `h${crypto.randomUUID().slice(0, 8)}`. Вместо `__wfHighlightObserverInstalled` → `Symbol()` или `WeakSet`.

2. **Установить localStorage через CDP**, а не через `addInitScript`:
```typescript
const client = await page.context().newCDPSession(page);
await client.send('DOMStorage.enable');
await client.send('DOMStorage.setDOMStorageItem', {
    storageId: { securityOrigin: 'https://www.wildberries.ru', isLocalStorage: true },
    key: 'wbx__tokenData',
    value: JSON.stringify({ token: accessToken }),
});
```
Это менее детектируемо, чем `addInitScript`.

3. **Вынести highlighter из контекста страницы** — подсветку делать через Electron overlay поверх окна браузера, а не через инъекцию CSS в страницу.

---

### Слабость 5: Auth slide подмена — не просто «добавить задержку»

**Что пропустил:** Моя предыдущая рекомендация «сделать auth slide с задержкой» — бесполезна.

**Реальная проблема:** В network.ts POST на `wbx-auth.wildberries.ru/v2/auth/slide-v3` полностью подменяется. Это значит:

- Реальный запрос **никогда не доходит до WB** (route.fulfill не делает реальный запрос)
- WB-серверу неизвестно, что клиент пытался авторизоваться через slide — нет серверного лога
- Но WB **клиентский JS** видит мгновенный ответ без сетевой задержки (0-1ms)
- Формат ответа (`result: 0, payload: { access_token, sticker }`) должен точно совпадать — если WB изменит формат, сломается

**Но:** Подмена auth slide — возможно **единственный рабочий вариант**. Настоящий slide — это slider-captcha, которую сложно пройти автоматически. Подмена позволяет не проходить её каждый раз.

**Усиленная рекомендация:**

Подмену auth slide оставить, но исправить:

1. **Добавить реалистичную задержку** перед `route.fulfill`:
```typescript
await page.route(AUTH_SLIDE_URL_REGEX, async (route, request) => {
    if (request.method() !== 'POST') {
        await route.continue();
        return;
    }
    // Имитация реального запроса (150-600ms)
    await new Promise(r => setTimeout(r, 150 + Math.random() * 450));
    await route.fulfill({ ... });
});
```

2. **Мониторить формат ответа** — периодически делать реальный запрос (1 из 10) и сравнивать структуру.

3. **В wb-runner** проблема та же — network.ts (wb-runner) подменяет slide так же.

---

### Слабость 6: Два разных стека автоматизации — удвоенная поверхность атаки

**Что пропустил:** Desktop использует **Patchright** (Playwright-форк), а wb-runner использует **Puppeteer + stealth plugin**. Это:
- Два разных TLS fingerprint (Patchright vs Puppeteer)
- Два разных набора обходов антидетекта
- Различные паттерны CDP-взаимодействия
- Различные баги и утечки

WB может видеть, что один и тот же аккаунт внезапно переключается между двумя разными «браузерами» (переход Desktop → wb-runner при needChallenge).

**Усиленная рекомендация:**

**Унифицировать стек на Patchright** для обоих компонентов. Patchright уже лучше обходит детекцию, чем Puppeteer + stealth. Или наоборот — оба на Camoufox (уже есть для Ozon). Главное — один стек.

Дополнительно: при переходе Desktop → wb-runner → Desktop сохранять TLS-совместимость через единый browser engine.

---

### Слабость 7: Sentry/мониторинг WB блокируется — это детектируемо

В proxyFailureDetector.ts присутствуют `BENIGN_REQUEST_FAILURE_URL_MARKERS`:
```
marketplace-sentry.wb.ru
/api/envelope/?sentry_
/webapi/logging/wbbasket/metrics
```

Это означает, что **WB Sentry-ошибки блокируются или игнорируются**. Но WB может проверять, доходят ли телеметрические запросы до их серверов. Если от аккаунта систематически не приходит телеметрия — это аномалия.

**Усиленная рекомендация:**

Не блокировать Sentry и метрики WB. Пусть уходят. Если они отправляют данные об ошибках — пусть отправляют. Рисков от сентри-трафика меньше, чем от его отсутствия.

---

### Слабость 8: `x-spa-version: '14.0.7'` + другие захардкоженные заголовки

**Подробнее:** В axios-wb-http.client.ts:

```typescript
private buildHeaders(ctx: WbRequestContextDto): Record<string, string> {
    return {
        'access-token': ctx.accessToken,
        'x-requested-with': 'XMLHttpRequest',
        deviceid: ctx.deviceId,
        priority: 'u=1, i',
        'x-spa-version': '14.0.7',       // ← ЗАШИТ
        origin: 'https://www.wildberries.ru',
        Referer: 'https://www.wildberries.ru/lk/myorders/delivery',  // ← ФИКСИРОВАННЫЙ referer
    };
}
```

**Усиленная проблема:**
- `x-spa-version` устаревает — WB обновляет SPA минимум раз в неделю
- **Фиксированный Referer** `myorders/delivery` на ВСЕХ запросах — реальный пользователь имеет разные referer
- Заголовок `priority: 'u=1, i'` — это HTTP/3 Priority header, его присутствие в HTTP/1.1 или HTTP/2 через прокси может быть аномалией

**Усиленная рекомендация:**

1. `x-spa-version` — парсить динамически из HTML-страницы WB или из их webpack manifest
2. Referer — передавать контекстно, по текущей странице
3. Убрать `priority` — или сделать его присутствие условным

---

### Слабость 9: Не учёл таймлайн взаимодействия Desktop

**Порядок действий в wbAccountOpener.ts:**

```
1. browser.newContext()
2. context.newPage()
3. setupProxyFailureHandling()      — подписка на requestfailed
4. setupOrderSubmissionTracking()    — CDP-сессия #1 + Network.enable
5. page.on('framenavigated')        — подписка на навигацию
6. attachRequestInterception()       — page.route() для auth slide + статики
7. applyAccountToPage()             — cookies + localStorage через addInitScript
8. setupAntibotTokenForwarding()    — CDP-сессия #2 + Network.enable (фоново)
9. highlightProducts()              — addInitScript с CSS/observer
10. page.goto(WILDBERRIES_PROFILE_URL) — навигация на /lk
```

**Проблемы:**
- `applyAccountToPage` (шаг 7) использует `addInitScript` для localStorage — но навигация на шаге 10 идёт на `/lk`, и эти скрипты выполняются при загрузке. **Однако** если между шагами 7 и 10 нет навигации — localStorage будет пустым до goto. Это оки, потому что `addInitScript` работает на **каждой** навигации.

- Шаг 4 создаёт CDP-сессию **до** route interception (шаг 6). Это значит, что CDP Network.enable на шаге 4 уже видит запросы, но route interception ещё не готов. Возможна гонка.

- **Нет warm-up навигации.** Сразу `/lk` (профиль) — нереалистично. Реальный пользователь заходит на главную.

**Усиленная рекомендация:**

Изменить порядок:
1. Сначала все route + CDP-сессии
2. Потом `applyAccountToPage`
3. Потом навигация на **главную** (`wildberries.ru/`)
4. Задержка 2-5 секунд
5. Навигация на целевую страницу

---

### Слабость 10: Отсутствует анализ `WbAasTokenInfo` — дрифт IP в токене

В `WbAccount` entity есть `wbAasTokenInfo`:
```typescript
export type WbAasTokenInfo = {
    ip: string;
    userAgent: string;
    expiresAt: string;
    tokenMode: string;
    createdAt: string;
};
```

Антибот-токен WB **содержит IP-адрес и User-Agent**, с которых он был создан. Это значит:

- Если токен создан с IP `1.2.3.4`, а потом аккаунт делает запросы с `5.6.7.8` — WB видит несоответствие
- Если User-Agent в токене не совпадает с User-Agent в заголовках — несоответствие
- WB **парсит свой собственный токен** серверно и сравнивает с текущим запросом

**Это ещё одна причина, почему IP drift критичен** — даже если UI работает, серверные проверки отклонят отзыв.

**Усиленная рекомендация:**

1. После каждой смены IP **обновлять antibot-токен** (re-trigger challenge)
2. Хранить IP из `WbAasTokenInfo.ip` и сравнивать с текущим `pinnedExternalIp` перед каждым действием
3. Если IP не совпадает — не давать публиковать отзыв, сначала обновить токен

---

## Обновлённая приоритизация (усиленная)

### Быстрые (1-3 дня), в порядке влияния:

| # | Действие | Почему усилено |
|---|----------|---------------|
| 1 | **Статику с `*.wildberries.ru` — через прокси (`route.continue()`), CDN-ресурсы (images, fonts) — можно через `fetch()`** | Не просто «убрать fetch», а дифференцировать по домену. CDN-ресурсы (basket-01.wbbasket.ru, images.wbstatic.net) безопаснее грузить напрямую, чем API/JSON |
| 2 | **Объединить CDP-сессии** orderTracking и antibotForwarding в одну | Уменьшение surface area детекции, было пропущено |
| 3 | **Добавить задержку в auth slide** — 150-600ms рандом | Конкретная цифра, не «сделать реалистичнее» |
| 4 | **Переименовать/обфусцировать маркеры** highlighter (`wf-highlight-*`, `__wfHighlightObserverInstalled`) | Новый пункт, ранее не учтён |
| 5 | **Установить viewport + locale + timezone** при `browser.newContext()` | Остаётся из предыдущего плана |
| 6 | **Не блокировать/игнорировать Sentry-трафик WB** | Новый пункт |

### Среднесрочные (1-2 недели):

| # | Действие | Почему усилено |
|---|----------|---------------|
| 7 | **Унифицировать стек: wb-runner перевести с Puppeteer на Patchright** | Новый пункт. Два стека = удвоенная поверхность атаки |
| 8 | **Динамический `x-spa-version`** — парсить из HTML WB при каждой сессии | Подробнее обосновано |
| 9 | **Проверять IP из `WbAasTokenInfo` vs текущий IP** перед действиями | Новый критический пункт |
| 10 | **Warm-up навигация** — сначала главная, потом целевая + рандомные паузы | Конкретизировано из «инструкции менеджерам» |
| 11 | **Установить localStorage через CDP** вместо `addInitScript` | Новый пункт — менее детектируемый метод |
| 12 | **Хранить fingerprint-параметры в entity** (viewport, timezone, screen) | Остаётся |
| 13 | **Контекстный Referer** вместо фиксированного `/lk/myorders/delivery` | Новый пункт |

### Долгосрочные (1+ месяц):

| # | Действие |
|---|----------|
| 14 | Миграция на Camoufox (единый антидетект для WB и Ozon) |
| 15 | ISP/stationary прокси для ценных аккаунтов |
| 16 | Система прогрева аккаунтов с health score |
| 17 | Валидация контента отзывов перед публикацией |
| 18 | Кеширование статических ресурсов per-profile для экономии трафика |

---

## Резюме: что реально меняет ситуацию

**Три главных фокуса**, которые я недооценил в первом плане:

1. **wb-runner — отдельный детектируемый компонент.** Puppeteer + stealth + headless + серверные флаги — это лёгкая мишень. Из-за него WB помечает аккаунт как подозрительный **ещё до того, как менеджер открывает его на десктопе.**

2. **IP в антибот-токене.** Даже если прокси работает стабильно — если IP при создании токена не совпадает с IP при публикации отзыва, модерация отклонит. Это архитектурная проблема, а не просто «нестабильные прокси».

3. **Множественные CDP-сессии + инъекции через addInitScript** — создают уникальный fingerprint автоматизации, даже несмотря на Patchright. WB может не детектировать каждый из этих факторов по отдельности, но их комбинация уникальна для automatized окружения.
