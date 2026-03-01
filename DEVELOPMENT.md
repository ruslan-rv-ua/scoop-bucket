# scoop-bucket: документація для розробника

## Загальна схема

```
репозиторій застосунку
  └─ push тегу v*
       └─ release.yml
            ├─ збірка
            ├─ пакування у ZIP
            ├─ публікація GitHub Release
            └─ repository_dispatch → scoop-bucket
                   └─ update-scoop-manifest.yml
                        ├─ створює гілку update/{app}-{version}
                        ├─ оновлює bucket/{app}.json
                        └─ відкриває Pull Request
                               └─ ci.yml (перевірка маніфесту)
                                    └─ auto-merge якщо CI ✓
```

## Файли у цьому репозиторії

```
scoop-bucket/
├─ bucket/
│   ├─ browserselector.json   ← маніфест пакету
│   └─ quick-snippets.json
└─ .github/
    └─ workflows/
        ├─ ci.yml                     ← перевірка маніфестів + автомерж
        └─ update-scoop-manifest.yml  ← універсальний оновлювач
```

---

## Як додати новий застосунок

### 1. У репозиторії scoop-bucket

Додай маніфест `bucket/{appname}.json` (назва файлу — lowercase):

```json
{
  "version": "1.0.0",
  "description": "Опис застосунку",
  "homepage": "https://github.com/ruslan-rv-ua/MyApp",
  "license": "MIT",
  "architecture": {
    "64bit": {
      "url": "https://github.com/ruslan-rv-ua/MyApp/releases/download/v1.0.0/MyApp-1.0.0-windows-x64.zip",
      "hash": "sha256тут",
      "extract_dir": "MyApp"
    }
  },
  "bin": "MyApp.exe",
  "checkver": {
    "github": "https://github.com/ruslan-rv-ua/MyApp"
  }
}
```

> ⚠️ Поле `extract_dir` має збігатись з назвою директорії яку скрипт пакування створює всередині ZIP. У шаблоні нижче це `$packageDir = $appName`, тобто `MyApp`. Якщо зміниш — оновлюй в обох місцях.

Більше нічого у scoop-bucket робити не треба — `update-scoop-manifest.yml` вже готовий і підхопить будь-який новий маніфест автоматично.

### 2. У репозиторії застосунку

Скопіюй [`examples/release-template.yml`](examples/release-template.yml) у `.github/workflows/release.yml` репозиторію застосунку.

Після копіювання зроби три речі:

1. **`APP_NAME`** — зміни на ім'я свого застосунку (єдиний обов'язковий рядок, всі інші частини workflow генеруються з нього автоматично).
2. **Кроки збірки** — заміни TODO-блок «ЗБІРКА» на кроки збірки свого проєкту. У шаблоні є готові коментовані приклади для MSYS2/make, Rust і Node.js.
3. **Файли у ZIP** — у кроці `Create release package` заміни TODO-рядки `Copy-Item` на реальні шляхи до файлів свого застосунку.

`APP_NAME` використовується скрізь у workflow: як ім'я ZIP-архіву і як ідентифікатор застосунку при відправці події в scoop-bucket (приводиться до lowercase автоматично).

> **Що містить шаблон:**
> - `workflow_dispatch` з двома полями: `version` (рядок, обов'язковий) і `prerelease` (прапорець, необов'язковий)
> - При ручному запуску версія береться з `inputs.version`, при тег-тригері — з тегу автоматично
> - `draft: false` — реліз публікується одразу, без чернетки
> - Pre-release: автоматично за суфіксом тегу (`-alpha`/`-beta`/`-rc`) **і** за ручним прапорцем
> - Умова `Update Scoop bucket` враховує обидва способи позначення pre-release

### 3. Як створити SCOOP_BUCKET_TOKEN

`SCOOP_BUCKET_TOKEN` — це персональний токен доступу (Personal Access Token), який дозволяє workflow у репозиторії застосунку писати в інший репозиторій — `scoop-bucket`. Стандартний `GITHUB_TOKEN` для цього не підходить, бо він має доступ тільки до поточного репозиторію.

**Крок 1: відкрий сторінку створення токена**

GitHub → аватар у правому верхньому куті → **Settings** → ліве меню, прокрути вниз → **Developer settings** → **Personal access tokens** → **Fine-grained tokens** → **Generate new token**

**Крок 2: заповни форму**

| Поле | Значення |
|------|----------|
| Token name | `scoop-bucket-dispatch` (або будь-яка зрозуміла назва) |
| Expiration | на свій розсуд; рекомендую 1 рік — GitHub нагадає про оновлення |
| Resource owner | `ruslan-rv-ua` |
| Repository access | **Only select repositories** → вибрати `scoop-bucket` |

У секції **Permissions → Repository permissions** встанови:
- **Contents**: Read and write
- **Metadata**: Read-only (додається автоматично)

Решта прав — залишай без змін (No access).

**Крок 3: збережи токен**

Натисни **Generate token**. GitHub покаже значення токена — рядок виду `github_pat_...`.

> ⚠️ Це єдиний момент коли GitHub показує повне значення токена. Після закриття сторінки побачити його знову буде неможливо. Скопіюй його зараз.

**Один токен — для всіх застосунків.** Токен надає доступ до `scoop-bucket`, а не до конкретного застосунку — тому він один для всіх. При додаванні нового застосунку просто додаєш той самий токен як секрет у новий репозиторій.

> ⚠️ Збережи токен у менеджері паролів одразу після генерації. Якщо загубиш — доведеться генерувати новий і вручну оновлювати секрет `SCOOP_BUCKET_TOKEN` у кожному репозиторії застосунку.

### 4. Налаштування секрету у репозиторії застосунку

Скопійований токен треба зберегти як секрет у репозиторії застосунку, щоб workflow міг його використати.

GitHub → репозиторій застосунку → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

| Поле | Значення |
|------|----------|
| Name | `SCOOP_BUCKET_TOKEN` |
| Secret | вставити скопійоване значення токена (`github_pat_...`) |

Натисни **Add secret**.

Після цього `${{ secrets.SCOOP_BUCKET_TOKEN }}` у `release.yml` автоматично підставить потрібне значення під час запуску workflow. Саме значення токена ніколи не з'являється у логах.

### 5. Увімкнути автомерж у scoop-bucket

- GitHub → scoop-bucket → **Settings → General → Pull Requests**
- Увімкнути **Allow auto-merge**

Без цього CI буде перевіряти PR, але мержити доведеться вручну.

---

## Як виглядає процес після push тегу

```
git tag v1.2.3
git push origin v1.2.3
```

1. `release.yml` запускається на `windows-latest`
2. Збирає застосунок
3. Пакує `MyApp-1.2.3-windows-x64.zip` + рахує SHA256
4. Публікує GitHub Release з автогенерованими release notes
5. Надсилає `repository_dispatch` з `event-type: update-myapp` у scoop-bucket
6. `update-scoop-manifest.yml` створює гілку `update/myapp-1.2.3`
7. Оновлює `bucket/myapp.json` (version, url, hash)
8. Відкриває PR: _"Update myapp to 1.2.3"_
9. `ci.yml` перевіряє JSON-синтаксис і структуру через `scoop info`
10. Якщо ✓ — автомерж у main через squash commit

Для тегів з `-alpha`, `-beta`, `-rc` — GitHub Release позначається як pre-release, крок оновлення bucket пропускається.

---

## Ручне оновлення маніфесту

Якщо треба оновити маніфест без релізу (наприклад виправити щось):

- GitHub → scoop-bucket → **Actions → Update Scoop manifest → Run workflow**
- Заповнити поля: `app`, `version`, `hash`, `url`

---

## Checklist для нового застосунку

- [ ] Створено `bucket/{appname}.json` у scoop-bucket
- [ ] Скопійовано [`examples/release-template.yml`](examples/release-template.yml) як `.github/workflows/release.yml` у репозиторій застосунку, змінено `APP_NAME`, замінено TODO-кроки збірки і файли ZIP
- [ ] Додано секрет `SCOOP_BUCKET_TOKEN` у репозиторій застосунку
- [ ] Увімкнено **Allow auto-merge** у scoop-bucket (якщо ще не зроблено)
- [ ] Зроблено тестовий реліз з тегом типу `v0.1.0-alpha` (bucket не чіпає, але перевіряє збірку)
- [ ] Зроблено повноцінний реліз, перевірено що PR у scoop-bucket відкрився і змержився

---

## Можливі проблеми

**CI падає з "Invalid manifest"** — перевір структуру JSON, особливо поле `architecture."64bit"`. Запусти `scoop info` локально.

**PR не мержиться автоматично** — перевір що в scoop-bucket увімкнено Allow auto-merge і що `ci.yml` має `pull-requests: write`.

**`repository_dispatch` не спрацьовує** — перевір що токен `SCOOP_BUCKET_TOKEN` не прострочений і має доступ до scoop-bucket з правами Contents: write.

**Помилка "Manifest file not found"** — назва файлу у `bucket/` має бути строго lowercase і збігатись з `APP_NAME` приведеним до lowercase у `release.yml`.
