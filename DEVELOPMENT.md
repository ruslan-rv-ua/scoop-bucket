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

Створи `.github/workflows/release.yml` у репозиторії застосунку:

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

env:
  # Єдине місце де треба змінити ім'я при використанні цього шаблону.
  # Регістр важливий: саме так буде названо ZIP-архів.
  APP_NAME: MyApp

permissions:
  contents: write

jobs:
  build:
    runs-on: windows-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # TODO: додай сюди кроки збірки свого застосунку.
      # Де буде результат збірки — залежить від проєкту, жодних вимог немає.

      - name: Get version and app info
        id: info
        shell: bash
        run: |
          VERSION=${GITHUB_REF#refs/tags/v}
          APP_ID=$(echo "${{ env.APP_NAME }}" | tr '[:upper:]' '[:lower:]')
          echo "VERSION=$VERSION" >> $GITHUB_OUTPUT
          echo "APP_ID=$APP_ID" >> $GITHUB_OUTPUT

      - name: Create release package
        id: package
        shell: pwsh
        run: |
          $appName = "${{ env.APP_NAME }}"
          $version = "${{ steps.info.outputs.VERSION }}"
          $packageDir = $appName
          $zipName = "$appName-$version-windows-x64.zip"

          New-Item -ItemType Directory -Force -Path $packageDir

          # TODO: скопіюй сюди файли які мають потрапити в ZIP.
          # Шляхи залежать від твого проєкту — бери звідки треба.
          # Copy-Item "path\to\file.exe" -Destination $packageDir
          # Copy-Item "path\to\config.json" -Destination $packageDir

          Compress-Archive -Path $packageDir -DestinationPath $zipName -Force

          $hash = (Get-FileHash -Path $zipName -Algorithm SHA256).Hash.ToLower()
          $hash | Out-File -FilePath "$zipName.sha256" -NoNewline -Encoding ascii

          echo "HASH=$hash" >> $env:GITHUB_OUTPUT
          echo "ZIP_NAME=$zipName" >> $env:GITHUB_OUTPUT

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          files: |
            ${{ env.APP_NAME }}-*.zip
            ${{ env.APP_NAME }}-*.zip.sha256
          generate_release_notes: true
          prerelease: ${{ contains(github.ref, '-alpha') || contains(github.ref, '-beta') || contains(github.ref, '-rc') }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      # Пропускається для pre-release тегів (-alpha, -beta, -rc)
      - name: Update Scoop bucket
        if: ${{ !contains(github.ref, '-alpha') && !contains(github.ref, '-beta') && !contains(github.ref, '-rc') }}
        uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.SCOOP_BUCKET_TOKEN }}
          repository: ruslan-rv-ua/scoop-bucket
          event-type: update-${{ steps.info.outputs.APP_ID }}
          client-payload: |
            {
              "app": "${{ steps.info.outputs.APP_ID }}",
              "version": "${{ steps.info.outputs.VERSION }}",
              "hash": "${{ steps.package.outputs.HASH }}",
              "url": "https://github.com/${{ github.repository }}/releases/download/v${{ steps.info.outputs.VERSION }}/${{ steps.package.outputs.ZIP_NAME }}"
            }
```

`APP_NAME` використовується скрізь у workflow: як ім'я ZIP-архіву і як ідентифікатор застосунку при відправці події в scoop-bucket (приводиться до lowercase автоматично).

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
- [ ] Створено `release.yml` у репозиторії застосунку, змінено `APP_NAME`
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
