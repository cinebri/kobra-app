# KOBRA VPN App — План разработки

> База: форк [hiddify/hiddify-app](https://github.com/hiddify/hiddify-app)  
> Наш репо: [cinebri/kobra-app](https://github.com/cinebri/kobra-app)  
> Upstream remote: `git pull upstream main`  
> Стек: Flutter + Sing-box core

---

## Почему Hiddify, а не FlClashX

- **Sing-box core** — тот же движок что в NekoBox (который стабильнее FlClashX на Android)
- **Все платформы**: Android, iOS, Windows, macOS, Linux из одной кодовой базы
- FlClashX = Clash Meta core → нестабильность, которую ты ощущаешь
- Remnawave-интеграция делается через `kobra-*` HTTP-заголовки — чище, чем `flclashx-*`

---

## Архитектура kobra-* заголовков

Remnawave умеет добавлять кастомные заголовки в ответ на запрос подписки через
`responseModifications.headers` в Response Rules. Приложение читает их при каждом
обновлении профиля.

### Заголовки (настраиваются в панели Remnawave)

| Header | Описание | Пример |
|--------|----------|--------|
| `kobra-renew-url` | URL для продления (Telegram deeplink или web) | `https://t.me/kobraink_bot?start=renew` |
| `kobra-support-url` | Поддержка (переопределяет `support-url`) | `https://t.me/kobraink_support_bot` |
| `kobra-service-name` | Название сервиса в UI | `KOBRA VPN` |
| `kobra-service-logo` | URL лого (SVG/PNG) | `https://kobra.ink/assets/logo.svg` |
| `kobra-theme-color` | HEX акцентного цвета | `FF2D00` |

### Как настроить в Remnawave

В панели → Subscription → Response Rules → добавить в `responseModifications`:

```json
{
  "headers": {
    "kobra-renew-url": "https://t.me/kobraink_bot?start=renew",
    "kobra-support-url": "https://t.me/kobraink_support_bot",
    "kobra-service-name": "KOBRA VPN",
    "kobra-service-logo": "https://kobra.ink/assets/logo.svg",
    "kobra-theme-color": "FF2D00"
  }
}
```

---

## Фазы разработки

### Фаза 1 — Базовый брендинг ✅ (сделано)

- [x] Fork hiddify/hiddify-app → cinebri/kobra-app
- [x] `constants.dart` — имя, URLs, Telegram
- [x] `profile_parser.dart` — allowedProfileHeaders добавлены kobra-* заголовки
- [x] `profile_entity.dart` — SubscriptionInfo расширена полями renewUrl, serviceName, serviceLogoUrl, themeColor
- [x] `environment.dart` — Sentry DSN отключён
- [x] Android manifest — label "KOBRA VPN", добавлена схема `kobra://`
- [x] iOS Info.plist — CFBundleDisplayName "KOBRA VPN"
- [x] Windows Runner.rc — CompanyName, ProductName
- [x] Linux my_application.cc — заголовок окна
- [ ] Иконка приложения (logo.svg → все форматы)
- [ ] Цветовая тема (акцентный цвет KOBRA)
- [ ] Убрать Hiddify из всех строк локализации (`arb/` файлы)

### Фаза 2 — KOBRA UI виджеты

Реализовать отображение kobra-* данных в профиле:

- [ ] **ServiceInfoWidget** — логотип + название сервиса на главном экране  
  (аналог `serviceInfo` в FlClashX)
- [ ] **RenewBanner** — баннер когда подписка истекает через ≤3 дня или истекла.  
  Кнопка "Продлить" открывает `renewUrl`
- [ ] **SubscriptionStatusCard** — дней до конца, остаток трафика, `themeColor` как акцент
- [ ] Применение `themeColor` из заголовка как акцентного цвета темы приложения

### Фаза 3 — Упрощение UX

Убрать для обычного пользователя лишнее:

- [ ] Скрыть Warp-настройки (не нужны для KOBRA)
- [ ] Скрыть Fragment/TLS Tricks настройки в UI (управляются подпиской)
- [ ] Скрыть ручное добавление одиночных прокси (только subscription URL/QR)
- [ ] Упростить "Per-app proxy" — скрыть или облегчить UI
- [ ] Убрать `telegramChannelUrl` из UI или заменить на @kobraink_support_bot

### Фаза 4 — Auto-update из наших релизов

Hiddify уже имеет систему auto-update через `appcast.xml` + GitHub Releases API.  
Нам нужно:

- [ ] `appcast.xml` настроить для `cinebri/kobra-app` releases
- [ ] CI/CD: GitHub Actions → сборка всех платформ → release
- [ ] `githubReleasesApiUrl` уже указывает на наш repo (✅ из Фазы 1)

### Фаза 5 — CI/CD и дистрибуция

- [ ] **Android APK** — GitHub Actions `flutter build apk`  
  (arm64-v8a + arm7 + x86_64 как у Hiddify)
- [ ] **Windows Installer** — `flutter build windows` + NSIS/Inno Setup
- [ ] **macOS DMG** — `flutter build macos` (нужен macOS runner)
- [ ] **Linux AppImage** — `flutter build linux` + AppImage packaging
- [ ] **Auto-deploy** через `gh release create` с assets

### Фаза 6 — Продвинутые фичи

- [ ] **Offline QR** — показать QR подписки когда нет сети (как PWA offline)
- [ ] **Deep link `kobra://add/<url>`** — добавление подписки через ссылку  
  (уже `kobra://` в AndroidManifest ✅)
- [ ] **Routing profiles** — пресеты из Remnawave (fast/adblock/youtube-clean)  
  через кастомный заголовок `kobra-routing-profiles`
- [ ] **Server health indicator** — показывать задержку нод (уже есть в Hiddify)
- [ ] **Quick toggle** — виджет на Android (уже есть, проверить)

---

## Файлы для изменения (карта)

| Файл | Назначение |
|------|-----------|
| `lib/core/model/constants.dart` | URLs, имена ✅ |
| `lib/core/model/environment.dart` | Sentry DSN ✅ |
| `lib/features/profile/model/profile_entity.dart` | SubscriptionInfo ✅ |
| `lib/features/profile/data/profile_parser.dart` | kobra-* headers ✅ |
| `lib/features/profile/widget/profile_tile.dart` | Показывать renewUrl кнопку |
| `lib/features/profile/widget/profile_tile_main.dart` | RenewBanner |
| `lib/features/home/widget/` | ServiceInfoWidget |
| `assets/images/logo.svg` | Заменить на KOBRA лого |
| `assets/images/tray_icon.png/ico` | Иконка трея |
| `android/app/src/main/res/mipmap-*/` | Android иконки |
| `android/app/src/main/AndroidManifest.xml` | App name, scheme ✅ |
| `ios/Runner/Info.plist` | Bundle display name ✅ |
| `windows/runner/Runner.rc` | ProductName ✅ |
| `linux/my_application.cc` | Window title ✅ |
| `arb/*.arb` | Локализация — убрать "Hiddify" |

---

## Стратегия апстрима

Hiddify активно развивается. Чтобы получать обновления (новые протоколы, фиксы):

```bash
# Получить обновления от Hiddify
git fetch upstream
git merge upstream/main

# Наши изменения в отдельных файлах — конфликтов немного
# constants.dart, profile_entity.dart, profile_parser.dart — минимальный diff
```

Держать наши изменения **минимальными** — только добавления, не переписывания.
Тогда merge upstream = почти без конфликтов.

---

## Идеи (парковка)

- **kobra-announcement** заголовок: текст анонса → баннер на главном экране  
  (как `announce` виджет в FlClashX)
- **kobra-routing** заголовок: `fast|stealth|adblock|youtube` → показывать в UI какой профиль активен
- **kobra-badge** заголовок: кастомный бейдж в иконке трея (например "PRO")
- **kobra-app-version-min**: минимальная версия KOBRA app для работы подписки →  
  принудительное обновление
- **HWID передача** (из FlClashX): при запросе подписки добавлять header  
  `X-Device-ID: <uuid>` → панель знает на каком устройстве активирована подписка  
  (нужно для ограничения числа устройств)
- Локализация: RU/EN как приоритет (убрать FA/ZH/JA из дефолта)

---

## Связанные ресурсы

| Ресурс | Путь/URL |
|--------|---------|
| FlClashX (справочник для Remnawave-фич) | `kobra-soft/kobra-app/reference/FlClashX/` |
| GUI.for.SingBox (справочник) | `kobra-soft/kobra-app/reference/GUI.for.SingBox/` |
| Hiddify upstream | `git remote upstream` |
| KOBRA PLAN.md | `/mnt/shared/projects/kobra/PLAN.md` |
| Remnawave response rules | `management/docs/KOBRA-MASTER-PLAN.md §3` |
