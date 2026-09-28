# План міграції YouTube Ambilight на macOS Safari

Документ описує стратегію перенесення розширення **Ambient light for YouTube™** на macOS Safari як повноцінного **Safari Web Extension** із супутнім нативним host-застосунком для macOS.

---

## 1. Поточна архітектура репозиторію

Репозиторій побудований на базі стандарту WebExtension (Manifest V3) із модульною структурою JavaScript (ES Modules), яка транспілюється за допомогою **Rollup** та **Babel** у самодостатні бандли формату **IIFE** для браузерного оточення:

- **`src/manifest.json`**: Конфігурація розширення під Manifest V3. Визначає фоновий service worker, контентні скрипти, сторінку опцій, доступні ресурси (`web_accessible_resources`) та дозволи (`storage`).
- **`src/scripts/background.js`**: Service Worker життєвого циклу розширення. Обробляє події `runtime.onInstalled` та натискання на іконку розширення `action.onClicked`.
- **`src/scripts/content.js`**: Вхідна точка контентного скрипта (`run_at: document_start`). Відстежує стан завантаження DOM, динамічно інжектує таблицю стилів `styles/content.css`, головний скрипт сторінки `scripts/injected.js` та динамічно імпортує `scripts/content-main.js`.
- **`src/scripts/injected.js`**: Скрипт, який виконується в головному контексті сторінки (`MAIN` world). Має прямий доступ до внутрішніх API відеоплеєра YouTube (`ytd-app`, `videoPlayerElem.setSize()`, `isHdrVideo()`, WebGL-контекст VR-відео).
- **`src/scripts/content-main.js`**: Основна логіка контентного скрипта в ізольованому контексті (`ISOLATED` world). Керує ініціалізацією Ambilight, перехоплює SPA-навігацію YouTube (`yt-navigate-finish`, `yt-page-data-updated`), мутації DOM та зміни розміру плеєра.
- **`src/scripts/libs/ambientlight.js`**: Серце розширення. Забезпечує захоплення відеокадрів (`requestVideoFrameCallback`), обчислення світіння, підтримку WebGL-шейдерів (`canvas-webgl.js`, `projector-webgl.js`) та резервного 2D Canvas (`projector-2d.js`), динамічне автовизначення чорних смуг (`bar-detection.js`).
- **`src/scripts/live-chat.js`**: Контентний скрипт для фрейму живого чату (`https://www.youtube.com/live_chat*`), що адаптує кольорову тему чату під кольори підсвічування.
- **`src/options.html` та `src/scripts/options.js`**: Інтерфейс керування розширенням. Забезпечує налаштування параметрів підсвічування, імпорт/експорт конфігурацій (у файл та через синхронізоване сховище `storage.sync`), а також керування телеметрією.
- **`src/styles/`**: SCSS-джерела (`content.scss`, `live-chat.scss`, `options.css`), що компілюються у CSS через `sass`.

### Міжкомпонентна взаємодія
- **Main World <-> Isolated World**: Реалізована через `CustomEvent` (`ytal-message`) на об'єкті `document` із серіалізацією повідомлень у JSON. Не залежить від специфічних браузерних розширень і надійно працює в WebKit.
- **Content Script <-> Background / Options**: Реалізована через `chrome.storage.local` з відстеженням подій `storage.onChanged`.

---

## 2. Що можна залишити без змін

1. **Рушій Ambilight та шейдери**:
   - Повний математичний апарат аналізу кольору відео.
   - WebGL / WebGL2 шейдери в `canvas-webgl.js` та `projector-webgl.js`.
   - Резервний 2D Canvas проектор `projector-2d.js`.
   - Алгоритм виявлення letterbox/pillarbox смуг (`bar-detection.js`).
   - Підтримка режимів Fullscreen, Theater mode, Default player size.
2. **Сторінка налаштувань (Options UI)**:
   - Вся HTML розмітка `options.html` та стилі `options.css`.
   - Схема збереження параметрів (`setting-*`), логіка валідації та налаштування за замовчуванням.
   - Експорт/імпорт конфігурації у JSON-файл.
3. **Комунікаційний шар**:
   - `CustomEvent` обмін між `injected.js` та `content-main.js`.
4. **Стилі та графічні ресурси**:
   - `styles/content.scss`, `styles/live-chat.scss`.
   - Растрові та векторні ресурси: іконки (`icon-16.png`, `icon-32.png`, `icon-128.png`), текстури шуму (`noise-1.png` .. `noise-3.png`), `donate.svg`, `settings-moved-to.png`.

---

## 3. Що потрібно адаптувати

1. **Детекція браузера (`src/scripts/libs/utils.js`)**:
   - Додати розпізнавання браузера Safari у списку `browsersUAList`. Оскільки User-Agent браузерів Chrome, Edge та Opera містить токен `Safari`, токен `Safari` повинен перевірятися після них або із запереченням наявності Chrome.
   - Адаптувати функцію `getBrowserVersion()` для Safari (версія витягується з токена `Version/X.Y.Z`).
2. **Обробка мережевих помилок (`src/scripts/libs/generic.js`)**:
   - У функції `isNetworkError(ex)` додати розпізнавання помилок Safari WebKit (`ex?.message === 'Load failed'`).
3. **Адаптація Manifest V3 для Safari**:
   - Safari підтримує Manifest V3, починаючи з Safari 15.4.
   - Поле `open_in_tab` у секції `options_ui` не підтримується Safari і викликає попередження валідатора — його слід адаптувати при формуванні маніфесту для Safari.
   - Зберегти всі правила `matches`, `exclude_matches`, `all_frames`, `web_accessible_resources` та дозволи `storage`.
4. **Робота зі сховищем (`storage.sync`)**:
   - Safari Web Extensions підтримують API `storage.sync`, проте зберігають дані локально (без автоматичної крос-девайсної синхронізації iCloud). Оскільки код уже має належну обробку помилок і фолбек, інтерфейс працює стабільно без збоїв.

---

## 4. Необхідні Safari та WebKit API

- **WebExtension API**:
  - `browser.runtime` / `chrome.runtime`: `getURL`, `onInstalled`, `openOptionsPage`.
  - `browser.storage.local` / `chrome.storage.local`: `get`, `set`, `remove`, `onChanged`.
  - `browser.storage.sync` / `chrome.storage.sync`: `get`, `set`, `onChanged`.
  - `browser.action` / `chrome.action`: `onClicked`.
- **WebKit DOM & Web APIs**:
  - `HTMLVideoElement.prototype.requestVideoFrameCallback` (стандартно підтримується в Safari 15.4+ на macOS).
  - `WebGLRenderingContext` / `WebGL2RenderingContext` з повною апаратною підтримкою Metal на Apple Silicon.
  - `CanvasRenderingContext2D`.
  - `MutationObserver`, `IntersectionObserver`, `CustomEvent`.
  - Поліфіл `requestIdleCallback` (у проєкті вже реалізовано коректний фолбек на `setTimeout`).

---

## 5. Файли, що створюються та оновлюються

1. **Документація та конфігурація**:
   - `SAFARI_MIGRATION_PLAN.md` — детальний план міграції.
   - `README.md` — додано інструкції щодо збірки для Safari та налаштування Xcode.
   - `package.json` — додано скрипти `build:safari`.
2. **Адаптація вихідного коду**:
   - `src/scripts/libs/utils.js` — додано підтримку Safari.
   - `src/scripts/libs/generic.js` — оновлено детекцію мережевих помилок WebKit.
   - `manifest-copy.js` — підтримка генерації маніфесту з урахуванням Safari.
3. **Xcode проєкт для macOS (`Xcode/`)**:
   - `Xcode/YouTube Ambilight.xcodeproj` — проєкт Xcode з двома цілями (targets):
     - `YouTube Ambilight` (macOS Host Application на Swift).
     - `YouTube Ambilight Extension` (Safari Web Extension target).
   - `Info.plist` та `Entitlements` для обох цілей з App Sandbox та відповідними дозволами.
   - Swift-код хост-додатку з інформаційним вікном для користувача про стан розширення та кнопкою відкриття налаштувань Safari.
   - Ресурси розширення, підключені безпосередньо з `dist/` (або автоматично оновлювані білд-фазою).

---

## 6. Процес збірки (Build Workflow)

1. **Встановлення залежностей**:
   ```bash
   npm install
   ```
2. **Збірка веб-ресурсів розширення**:
   ```bash
   npm run build:safari
   # або загальний білд
   npm run build
   ```
   Скрипт транспілює Rollup-модулі в `dist/scripts/`, компілює SCSS в `dist/styles/`, копіює `options.html`, зображення та валідує `manifest.json`.
3. **Збірка нативного бандлу macOS через Xcode**:
   - Відкрити `Xcode/YouTube Ambilight.xcodeproj`.
   - Обрати схему `YouTube Ambilight` та платформу `My Mac`.
   - Натиснути **Run** (`Cmd + R`).
   - Xcode збирає додаток та бандл розширення `.appex`, вкладаючи зібрані веб-ресурси.

---

## 7. План тестування та верифікації

1. **Активація в Safari**:
   - Увімкнути меню розробника: *Safari > Settings > Advanced > Show features for web developers*.
   - Увімкнути *Develop > Allow Unsigned Extensions*.
   - У *Safari > Settings > Extensions* активувати галочку навпроти "Ambient light for YouTube™" та надати дозвіл для `youtube.com`.
2. **Перевірка функціоналу на YouTube**:
   - Головна сторінка `https://www.youtube.com`.
   - Стандартне відео 16:9 та вертикальне відео Shorts.
   - Перемикання режимів: Default, Theater Mode, Fullscreen.
   - SPA-навігація: послідовний перехід Відео A -> Відео B -> Відео C -> Відео A (переконатися у відсутності дублювання підсвічування та DOM-елементів).
   - Зміна якості відео та пауза/відтворення.
   - Перевірка живого чату на трансляціях (`live_chat`).
   - Перевірка вбудованих плеєрів (`/embed/*`).
3. **Перевірка налаштувань (Options)**:
   - Відкриття сторінки налаштувань через іконку розширення в панелі Safari.
   - Зміна параметрів яскравості, розмиття, кольоровості, синхронізації кадрів.
   - Експорт налаштувань у файл JSON та імпорт.
4. **Продуктивність на Apple Silicon**:
   - Перевірка навантаження CPU та GPU під час відтворення 4K/60fps.
   - Перевірка відсутності витоків пам'яті WebGL-контекстів при зміні сторінок.
