# scoop-bucket: документація для розробника

## Загальна схема

```
репозиторій застосунку
  └─ push тегу v*
       └─ release.yml
            ├─ збірка
            ├─ <App>-v<версія>-x64.exe + сайдкар .sha256
            └─ GitHub Release
                                        ┆  (нічого не перетинає межу між репозиторіями)
scoop-bucket
  └─ excavator.yml   (кнопка в Actions або розклад раз на добу)
       ├─ читає checkver у кожному bucket/{app}.json
       ├─ бачить новіший Release → будує URL за autoupdate, бере хеш із сайдкара
       └─ комітить оновлений маніфест у main
            └─ ci.yml (перевірка маніфесту)
                 └─ при помилці — автоматичний revert
```

Раніше застосунки самі писали в bucket через `repository_dispatch` і тримали для цього
PAT у своїх секретах. Це прибрано: Excavator редагує репозиторій, у якому живе, тож йому
вистачає власного `github.token`, і жоден токен не треба ротувати.

## Файли у цьому репозиторії

```
scoop-bucket/
├─ bucket/
│   ├─ audiocaptor.json        ← маніфест пакету (bare .exe)
│   ├─ axygen-shot.json
│   ├─ browserselector.json
│   ├─ marka.json
│   ├─ pathmaster.json         ← зразок маніфесту з autoupdate через сайдкар
│   └─ quick-snippets.json
├─ examples/
│   └─ release-template.yml    ← шаблон release.yml для репозиторію застосунку
└─ .github/
    └─ workflows/
        ├─ ci.yml                     ← перевірка маніфестів + revert при помилці
        ├─ excavator.yml              ← автооновлення маніфестів (checkver/autoupdate)
        └─ update-scoop-manifest.yml  ← ручний важіль: виставити версію/URL/хеш
```

---

## Як додати новий застосунок

### 1. У репозиторії застосунку

Скопіюй [`examples/release-template.yml`](examples/release-template.yml) у
`.github/workflows/release.yml` і зроби три речі:

1. **`APP_NAME`** — ім'я exe без розширення. З нього складається ім'я артефакту
   `<APP_NAME>-v<версія>-x64.exe`.
2. **Версія проєкту** — у кроці `Gate — the tag names the project version` заміни TODO на
   читання версії зі свого джерела правди (Cargo.toml, package.json, файл VERSION). Тег,
   що не збігається з версією, відмовляється до збірки.
3. **Збірка і шлях до exe** — TODO-блок «ЗБІРКА» і рядок `Copy-Item` у `Stage the artifact`.

Форма артефакту фіксована: **голий exe і сайдкар** `<hex64> *<ім'я файлу>`. Без ZIP —
scoop перейменовує файл фрагментом `#/<APP_NAME>.exe`, а хеш бере з сайдкара.

Секретів у репозиторії застосунку **не потрібно**: `gh release create` працює на
`github.token` самого запуску.

### 2. Тестовий реліз

Постав тег із суфіксом — `v1.0.0-rc.1`. Release позначиться pre-release, а `releases/latest`
не зрушить, тож bucket нічого не побачить. Перевір, що на сторінці Release лежать exe і
`.sha256`, завантаж і запусти exe.

### 3. Маніфест у bucket

Excavator **оновлює** маніфести, але **не створює**: перший `bucket/{appname}.json`
(ім'я файлу — lowercase) засівається руками, після справжнього (не rc) релізу.
Зразок — [`bucket/pathmaster.json`](bucket/pathmaster.json):

```json
{
  "version": "1.0.0",
  "description": "Опис застосунку",
  "homepage": "https://github.com/ruslan-rv-ua/MyApp",
  "license": "MIT",
  "architecture": {
    "64bit": {
      "url": "https://github.com/ruslan-rv-ua/MyApp/releases/download/v1.0.0/MyApp-v1.0.0-x64.exe#/MyApp.exe",
      "hash": "<64 hex із сайдкара>"
    }
  },
  "bin": "MyApp.exe",
  "shortcuts": [["MyApp.exe", "MyApp"]],
  "persist": "data",
  "checkver": "github",
  "autoupdate": {
    "architecture": {
      "64bit": {
        "url": "https://github.com/ruslan-rv-ua/MyApp/releases/download/v$version/MyApp-v$version-x64.exe#/MyApp.exe",
        "hash": { "url": "$url.sha256" }
      }
    }
  }
}
```

Що тут несуче:

- **`#/MyApp.exe`** — фрагмент перейменовує завантажений файл; `bin` і `shortcuts` через це
  не міняються від версії до версії.
- **`checkver: "github"`** — Excavator дивиться на `releases/latest` репозиторію з `homepage`.
- **`autoupdate.hash.url: "$url.sha256"`** — `$url` це URL без фрагмента, тож це сайдкар поруч
  з exe; scoop сам читає з нього рядок `<hex64> *<ім'я>`.
- **`persist`** — усе, що застосунок пише поряд із собою й що має пережити оновлення. Під
  scoop ці теки лежать у версійному каталозі, який `scoop cleanup` видаляє; `persist` робить
  із них junction у `~\scoop\persist\<app>\`. Кілька тек — масив: `["data", "recordings"]`.

Перевір локально: `scoop install .\bucket\myapp.json` ставить застосунок прямо з файлу;
`scoop uninstall myapp` після. Далі — розділ у README bucket, коміт і push у `main`; `ci.yml`
валідує маніфест через `scoop info` і відкотить коміт, якщо структура бита.

### 4. Кожен наступний реліз

Тег → Release → **Actions → Excavator → Run workflow** у цьому репозиторії (або дочекатись
добового запуску о 04:20 UTC). Лог Excavator називає кожен застосунок і що з ним сталось;
`THROW_ERROR: 1` робить помилку checkver червоним прогоном, а не тихим зеленим.

---

## Ручне оновлення маніфесту

Коли Excavator не підходить — відкотити на попередню версію, поставити реліз, якого
`checkver` не бачить, полагодити хеш:

- GitHub → scoop-bucket → **Actions → Set a manifest by hand (override) → Run workflow**
- Заповнити поля: `app`, `version`, `hash`, `url` (URL можна з фрагментом `#/…`)

Workflow завантажує файл і звіряє хеш **до** того, як щось змінювати.

---

## Checklist для нового застосунку

- [ ] Скопійовано `examples/release-template.yml` у репозиторій застосунку; змінено
      `APP_NAME`, читання версії, кроки збірки й шлях до exe
- [ ] Тестовий реліз `v1.0.0-rc.1`: на сторінці Release є exe і `.sha256`
- [ ] Справжній реліз `v1.0.0`
- [ ] `bucket/{appname}.json` засіяно руками з хешем із сайдкара; `scoop install .\bucket\{appname}.json` працює
- [ ] Розділ у README bucket
- [ ] Наступний реліз: Excavator оновив маніфест сам, CI bucket зелений

---

## Можливі проблеми

**CI падає з "Invalid manifest" і робить revert** — перевір структуру JSON, особливо поле
`architecture."64bit"`. Запусти `scoop info` локально. Після виправлення — коміт у `main`
або ручний workflow.

**Excavator зелений, але маніфест не оновився** — прочитай лог: він перелічує кожен застосунок.
Найчастіші причини: реліз позначений pre-release (`releases/latest` не зрушив), сайдкара
немає або він не у форматі `<hex64> *<ім'я>`, `homepage` у маніфесті не вказує на репозиторій
із релізами.

**Excavator червоний з "Hash mismatch" / "Could not find hash"** — сайдкар лежить не поруч
з exe або має інше ім'я, ніж `<url без фрагмента>.sha256`. Перевір `autoupdate.hash.url`.

**Помилка "Manifest file not found" у ручному workflow** — ім'я файлу в `bucket/` має бути
строго lowercase і збігатись із полем `app`.
