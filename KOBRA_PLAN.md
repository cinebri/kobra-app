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

## Архитектура подписочного шаблона (singbox.json)

### Ключевое решение: роутинг живёт в шаблоне, не в UI

Вся логика обхода блокировок задаётся в `reference/templates/kobra/singbox.json` —
Remnawave отдаёт его как содержимое подписки. Пользователь получает готовый пресет,
ничего не настраивает сам. Это исключает конфликты между настройками приложения
и конфигом подписки.

### Три слоя обхода для российских ресурсов

| Слой | Механизм | Что покрывает |
|------|----------|---------------|
| DNS | `dns.rules`: geosite-ru → dns-direct (Yandex `tls://77.88.8.8`) | Резолвинг RU-доменов без прокси |
| Приложения | `tun.exclude_package` (~100 пакетов) | Весь трафик Android-приложений мимо VPN |
| Домены/IP | `route.rules`: geosite-ru + geoip-ru → direct | Веб-сайты и IP, не покрытые exclude_package |

**Вывод**: приложения обходятся через `exclude_package`, сайты — через `geosite-ru`.
Оба слоя нужны, потому что браузеры (Chrome, Firefox) не байпасятся через exclude_package —
трафик к ruтuple ru-сайтам проходит через них.

### DNS-стратегия

- **dns-remote** — Cloudflare DoH (`https://cloudflare-dns.com/dns-query`) через прокси  
  Нейтральный быстрый DNS для иностранного трафика. Идёт через `KOBRA.INK`, поэтому
  блокировка CF в РФ не имеет значения — запрос уходит с IP ноды.
- **dns-direct** — Yandex DoT (`tls://77.88.8.8`) напрямую  
  Резолвинг российских доменов без прокси (получаем российские IP, не CDN-обёртки).
- **Порядок**: geosite-private → dns-local, geosite-ru + yandex + vk + mailru → dns-direct, всё остальное → dns-remote.

**Почему не AdGuard DoH**: у нас есть ноды без рекламы (adblock на стороне ноды) и ноды
с рекламой. DNS adblock на клиенте давал бы adblock ВСЕМ независимо от выбранной ноды —
ломал бы это разделение. Adblock = ответственность ноды (server-side `geosite:category-ads-all → block`).

### LAN bypass

`route_exclude_address` в TUN inbound: весь RFC-1918 и link-local трафик идёт мимо VPN.  
Раньше был отключён — включили явно (список 10.0.0.0/8, 192.168.0.0/16 и т.д.).

### Структура Proxy Groups

```
KOBRA.INK (selector)
  ├─ ⚡️ Fastest  (urltest, интервал 5m, tolerance 50ms)
  ├─ 🚀 Fast      (selector, outbounds: null → Remnawave подставит fast-ноды)
  ├─ 🔄 Fallback  (selector, outbounds: null → Remnawave подставит все ноды)
  └─ 🕶 Stealth   (selector, outbounds: null → Remnawave подставит stealth-ноды)
```

`outbounds: null` — плейсхолдер, который Remnawave заменяет на реальные прокси при генерации подписки.  
Фильтрация по типу ноды (fast/stealth) — на стороне Remnawave через tags/groups, не в Sing-box шаблоне  
(у Sing-box нет Mihomo-аналога `include-all: true` + `filter:`).

### Rule-sets

Используем бинарные `.srs` от MetaCubeX (`meta-rules-dat`). Sing-box скачивает только явно
объявленные rule-sets при первом запуске, кешируются в `kobra.db`:

| Tag | Файл | Назначение |
|-----|------|-----------|
| `geosite-private` | `geosite/private.srs` | LAN/loopback → dns-local / direct |
| `geoip-private` | `geoip/private.srs` | Частные IP → direct |
| `geosite-ru` | `geosite/category-ru.srs` | РФ-домены (широкая категория) |
| `geoip-ru` | `geoip/ru.srs` | Российские IP-диапазоны (RIPE) |
| `geosite-yandex` | `geosite/yandex.srs` | Все домены Яндекса (полный список) |
| `geosite-vk` | `geosite/vk.srs` | Домены VK/ВКонтакте |
| `geosite-mailru` | `geosite/mailru.srs` | Домены Mail.ru Group |

**Почему yandex/vk/mailru отдельно**: `category-ru.srs` — это агрегат категории из v2fly.
Он может включать эти сервисы частично. Отдельные `.srs` файлы содержат полный список
доменов сервиса и гарантируют корректный DNS-резолвинг через Yandex DoT.

Ozon и Wildberries: нет в v2fly как отдельные geosite — покрываются через `geoip-ru`
(их серверы находятся в российских IP-диапазонах) + `exclude_package` для Android-приложений.

**Проверено на нодах** (май 2026): Xray 26.3.27 внутри remnanode Docker-контейнера.
Все 6 тегов `geosite:yandex`, `vk`, `mailru`, `ozon`, `wildberries`, `category-ru` присутствуют
в `geosite.dat` — тест конфигом `xray -test` вернул `Configuration OK`.

MetaCubeX `.srs` файлы (`yandex.srs`, `vk.srs`, `mailru.srs`) возвращают HTTP 200 — существуют.

### Экспериментальные функции

- **Clash API** на `127.0.0.1:9090` с YACD UI — для дебага и ручного переключения нод
- **Cache file** `kobra.db` — кеш rule-sets и fakeip

### Файл шаблона и синхронизация с панелью

`reference/templates/kobra/singbox.json` — единственный источник правды.  
При изменении стратегии обхода → правим только его → обновляем в Remnawave.

**Как синхронизировать шаблон в панель** (Remnawave API):
```python
import json, urllib.request, base64
TOKEN = "<REMNAWAVE_API_TOKEN>"   # из reference/mcp-remnawave/.env
BASE  = "https://pnl.kobra.ink"
UUID  = "9433b135-9023-4046-841b-f71fbdd793ed"   # SINGBOX/KOBRA

template = json.load(open("reference/templates/kobra/singbox.json"))
body = json.dumps({"uuid": UUID, "templateJson": template}).encode()
req = urllib.request.Request(f"{BASE}/api/subscription-templates",
    data=body, method="PATCH",
    headers={"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"})
urllib.request.urlopen(req)
```

**Бэкапы** текущего состояния панели: `reference/templates/kobra/backup/*.bak`  
Снимать перед каждым изменением. Соответствие:

| Файл шаблона | UUID в панели | Тип |
|-------------|--------------|-----|
| `singbox.json` | `9433b135-9023-4046-841b-f71fbdd793ed` | SINGBOX/KOBRA |
| `mihomo.yaml` | `6aeb15c7-24c5-4ec2-9cb8-fca4cda1c567` | MIHOMO/KOBRA |
| `clash.yaml` | `97b8a000-1d83-49a8-a6f1-ce20a386158f` | CLASH/KOBRA |
| `stash.yaml` | `29ff581f-48e9-4044-b4ba-c04de93c4e5e` | STASH/KOBRA |
| `xray-json.json` | `bba1acb0-7472-4d07-8e0b-a09b30eac62e` | XRAY_JSON/KOBRA |

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
- **Routing profiles через kobra-* заголовок** — `kobra-routing-profile: adblock|gaming|streaming`  
  → приложение применяет пресет overlay поверх базового singbox.json
- **Sing-box vs Mihomo фильтрация групп** — проверить, поддерживает ли Remnawave name-based  
  фильтрацию нод при генерации Sing-box подписки (аналог Mihomo `filter: "(?i)Fast"`)
- **dart run build_runner build** — запустить локально после изменений в `profile_entity.dart`  
  (freezed-кодген не выполнен на сервере, Flutter SDK не установлен)
- **UA ребрендинг** — при переименовании UA с `HiddifyNext` → `KobraVPN` обновить response rule  
  в Remnawave (панель → Subscription Settings → Response Rules → Sing-box clients regex)

---

## Remnawave — инфраструктура и интеграция

### Архитектура серверной части

| Компонент | Роль |
|-----------|------|
| **Remnawave** (панель) | Web-бэкенд, генерирует подписки из шаблонов. Xray **внутри не запускается** |
| **Remnanode** (ноды) | Docker-контейнер с Xray 26.3.27 + `geosite.dat`/`geoip.dat` с хоста |
| **Подписка** | Remnawave берёт шаблон из БД, подставляет `outbounds: null` → реальные прокси |
| **`.srs` файлы** | Скачиваются Sing-box **клиентом** с GitHub при первом запуске, не нодами |

Ноды используют `.dat`-формат (Xray). Клиент — `.srs`-формат (Sing-box). Это разные стеки.

### User-Agent Hiddify / kobra-app

Hiddify отправляет при запросе подписки:
```
HiddifyNext/{version} ({OS}) like ClashMeta v2ray sing-box
```

Response rule в Remnawave (Sing-box clients) обновлён:
```
^(?:sfa|sfi|sfm|sft|karing|singbox|HiddifyNext)
```

Без `HiddifyNext` в регексе kobra-app получал бы **Base64 fallback** вместо Sing-box конфига.  
**Проверить после ребрендинга UA**: если сменим `HiddifyNext` → `KobraVPN` в `app_info_entity.dart`,
нужно обновить регекс в панели соответственно.

Файл UA: `lib/core/model/app_info_entity.dart`, строка `String get userAgent`.

---

## Связанные ресурсы

| Ресурс | Путь/URL |
|--------|---------|
| FlClashX (справочник для Remnawave-фич) | `kobra-soft/kobra-app/reference/FlClashX/` |
| GUI.for.SingBox (справочник) | `kobra-soft/kobra-app/reference/GUI.for.SingBox/` |
| Hiddify upstream | `git remote upstream` |
| KOBRA PLAN.md | `/mnt/shared/projects/kobra/PLAN.md` |
| Remnawave response rules | `management/docs/KOBRA-MASTER-PLAN.md §3` |
