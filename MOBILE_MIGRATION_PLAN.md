# План переноса MZC Calendar на iOS и Android

## Контекст проекта

**MZC Calendar** — PWA-приложение для учёта рабочих дней и доходов фрилансера.

Текущий стек:
- Single-page app: один `index.html` (~2900 строк) с инлайн CSS и JS
- Service Worker (`sw.js`) — кеширование, офлайн-режим
- `manifest.json` — уже настроен как PWA
- Хранилище данных: `localStorage`
- Без фреймворков, ванильный JS

Приложение уже оптимизировано под мобильные: `safe-area-inset`, touch-события, `user-select: none`, свайп-жесты, viewport-fit=cover.

---

## Выбор подхода: Capacitor

**Почему Capacitor, а не альтернативы:**

| Подход | Изменений кода | iOS + Android | Нативные API | Итог |
|---|---|---|---|---|
| **Capacitor** | Минимум | Да | Да | ✅ Оптимально |
| TWA (Android only) | Нет | Только Android | Нет | Частичное решение |
| React Native | Полная переработка | Да | Да | Слишком дорого |
| Flutter | Полная переработка | Да | Да | Слишком дорого |
| Cordova | Минимум | Да | Да | Устаревший инструмент |

Capacitor оборачивает существующий веб-код в нативный WebView, даёт доступ к нативным API через плагины, и используется как стандарт в экосистеме Ionic.

---

## Этапы миграции

### Этап 1 — Подготовка проекта (1–2 дня)

**1.1 Переход на файловую структуру проекта**

Сейчас весь код в одном `index.html`. Capacitor требует наличия папки с финальным билдом.

```
freelancers-time-calendar/
├── www/                    ← папка для Capacitor (билд)
│   ├── index.html
│   ├── manifest.json
│   ├── sw.js
│   └── icons/
├── android/                ← генерирует Capacitor
├── ios/                    ← генерирует Capacitor
├── capacitor.config.json
└── package.json
```

Задача: скопировать текущие файлы в `www/` или настроить webDir.

**1.2 Проверка Service Worker**

Service Worker (`sw.js`) в мобильном приложении работает иначе — Capacitor использует `capacitor://` protocol. Нужно убедиться, что SW корректно обрабатывает этот origin, или отключить SW для нативных сборок.

```js
// sw.js — добавить проверку
if (location.protocol !== 'capacitor:') {
  // регистрировать SW только для веб-версии
}
```

**1.3 Замена `localStorage` на Capacitor Preferences (опционально)**

Текущий `localStorage` будет работать в WebView, но может быть очищен системой. Для надёжности — плагин `@capacitor/preferences`.

Это необязательно на старте, можно сделать позже.

---

### Этап 2 — Установка и инициализация Capacitor (0.5 дня)

**Требования:** Node.js 16+, npm/yarn, Xcode 14+ (для iOS), Android Studio (для Android).

```bash
# Инициализация npm проекта
npm init -y

# Установка Capacitor
npm install @capacitor/core @capacitor/cli

# Инициализация (вводим имя приложения, bundle id, webDir)
npx cap init "MZC Calendar" "com.mzc.calendar" --web-dir www

# Добавление платформ
npm install @capacitor/ios @capacitor/android
npx cap add ios
npx cap add android
```

**`capacitor.config.json` (базовая конфигурация):**

```json
{
  "appId": "com.mzc.calendar",
  "appName": "MZC Calendar",
  "webDir": "www",
  "server": {
    "androidScheme": "https"
  },
  "plugins": {
    "SplashScreen": {
      "launchShowDuration": 600,
      "backgroundColor": "#080808",
      "showSpinner": false,
      "androidSpinnerStyle": "small"
    },
    "StatusBar": {
      "style": "Dark",
      "backgroundColor": "#080808"
    }
  }
}
```

---

### Этап 3 — Нативные настройки iOS (1–2 дня)

**3.1 Иконки приложения**

Нужно сгенерировать все размеры иконок для iOS App Store из `icon-512.png`:

- Использовать инструмент: [capacitor-assets](https://github.com/ionic-team/capacitor-assets)

```bash
npm install -D @capacitor/assets
npx capacitor-assets generate --ios
```

**3.2 Splash Screen**

Добавить `@capacitor/splash-screen` и сконфигурировать под тёмный стартовый экран (совпадает с текущим splash в `index.html`).

```bash
npm install @capacitor/splash-screen
npx cap sync
```

**3.3 Status Bar**

```bash
npm install @capacitor/status-bar
```

В `index.html` добавить инициализацию после загрузки:

```js
import { StatusBar, Style } from '@capacitor/status-bar';

// Вызвать после определения темы из localStorage
async function setNativeStatusBar(theme) {
  const isDark = ['t-dark', 't-night'].includes(theme);
  await StatusBar.setStyle({ style: isDark ? Style.Dark : Style.Light });
  await StatusBar.setBackgroundColor({ color: isDark ? '#080808' : '#f5f0e8' });
}
```

**3.4 Настройки Xcode**

В `ios/App/App/Info.plist`:
- `UIRequiresFullScreen` = YES (запрет split-view)
- Ориентация: только Portrait (уже задано в manifest)
- Privacy описания (если будет доступ к фото галерее)

**3.5 Тест на симуляторе**

```bash
npx cap open ios
# Далее в Xcode: выбрать симулятор → Run
```

---

### Этап 4 — Нативные настройки Android (1–2 дня)

**4.1 Иконки**

```bash
npx capacitor-assets generate --android
```

**4.2 Адаптивные иконки (Adaptive Icons)**

Android 8+ требует `foreground` и `background` слои. `capacitor-assets` генерирует автоматически из исходника.

**4.3 Тема и цвета**

В `android/app/src/main/res/values/styles.xml`:

```xml
<style name="AppTheme.NoActionBar" parent="Theme.AppCompat.DayNight.NoActionBar">
    <item name="android:windowBackground">@color/splash_bg</item>
    <item name="android:statusBarColor">#080808</item>
    <item name="android:navigationBarColor">#080808</item>
</style>
```

**4.4 Разрешения**

В `android/app/src/main/AndroidManifest.xml` добавить (если нужно фото):

```xml
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES"/>
```

**4.5 Target SDK**

Google Play требует `targetSdk >= 34` (на 2025 год). Проверить в `android/app/build.gradle`.

**4.6 Тест на эмуляторе**

```bash
npx cap open android
# Далее в Android Studio: выбрать AVD → Run
```

---

### Этап 5 — Нативные улучшения (2–3 дня, опционально)

Эти задачи не блокируют выпуск, но улучшают UX.

**5.1 Push-уведомления**

Напоминания о незаполненных рабочих днях.

```bash
npm install @capacitor/push-notifications
```

**5.2 Хранилище (замена localStorage)**

```bash
npm install @capacitor/preferences
```

Обернуть `cfg` объект в слой абстракции, чтобы переключить хранилище без изменения основной логики.

**5.3 Share (поделиться итогами)**

```bash
npm install @capacitor/share
```

Кнопка «Поделиться итогами месяца» в `.tcard`.

**5.4 Haptic Feedback**

Тактильный отклик при нажатии на кружки календаря.

```bash
npm install @capacitor/haptics
```

```js
import { Haptics, ImpactStyle } from '@capacitor/haptics';
// в обработчике dotPress:
await Haptics.impact({ style: ImpactStyle.Light });
```

---

### Этап 6 — Подготовка к публикации (2–3 дня)

**iOS App Store:**

1. Аккаунт Apple Developer ($99/год)
2. Создать App ID в [developer.apple.com](https://developer.apple.com)
3. Настроить Signing в Xcode (автоматическое или ручное)
4. Собрать архив: Product → Archive
5. Загрузить через Transporter или Xcode Organizer
6. Заполнить метаданные в App Store Connect
7. Прохождение ревью: ~1–3 дня

**Google Play:**

1. Аккаунт Google Play Developer ($25 единовременно)
2. Собрать подписанный AAB:
   ```bash
   cd android
   ./gradlew bundleRelease
   ```
3. Создать keystore для подписи (хранить в безопасном месте)
4. Загрузить в Google Play Console
5. Заполнить метаданные, скриншоты, описание
6. Ревью: ~3–7 дней

---

## Риски и решения

| Риск | Решение |
|---|---|
| Service Worker конфликтует с Capacitor | Отключить SW для нативных сборок через `navigator.serviceWorker` check |
| `localStorage` очищается на Android | Перейти на `@capacitor/preferences` |
| Кастомные шрифты (Google Fonts) не грузятся офлайн | Скачать и включить шрифты в `www/` |
| Фото из галереи (`<input type="file">`) | Работает в WebView, но может потребовать `@capacitor/camera` на iOS 16+ |
| Splash Screen отличается от веб-splash | Настроить нативный splash через плагин |
| Ревью Apple отклоняет "web wrapper" | Добавить хотя бы 1–2 нативных фичи (Push, Haptics) |

---

## Порядок работ (итого ~10–14 рабочих дней)

```
Неделя 1:
  Пн  — Этап 1: Структура проекта, SW fix, npm init
  Вт  — Этап 2: Capacitor init, обе платформы
  Ср  — Этап 3: iOS (иконки, splash, statusbar, Info.plist)
  Чт  — Этап 4: Android (иконки, тема, manifest, build.gradle)
  Пт  — Тестирование на симуляторах, баги

Неделя 2:
  Пн-Вт — Этап 5: Нативные улучшения (haptics, share, preferences)
  Ср    — Этап 6: Подготовка сборок, скриншоты, метаданные
  Чт-Пт — Отправка в App Store и Google Play
```

---

## Дополнительные материалы

- [Capacitor Docs](https://capacitorjs.com/docs)
- [capacitor-assets CLI](https://github.com/ionic-team/capacitor-assets)
- [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)
- [Google Play Policy](https://play.google.com/about/developer-content-policy/)
