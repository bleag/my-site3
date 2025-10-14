# Отчет о настройке GitHub Pages и CI/CD

## 1. Что сделано (кратко)
- Созданы страницы: `/ru/index.html`, `/en/index.html`, `/index.html`.
- Настроен workflow GitHub Actions `.github/workflows/publish-on-release.yml`, который:
  - Запускается при событии `release.published`.
  - Клонирует ветку `gh-pages` (создает, если отсутствует).
  - Копирует статические файлы в папку `v{tag}` на ветке `gh-pages`.
  - Также обновляет папку `latest` (чтобы было удобно перейти на последнюю версию).
  - Коммитит и пушит изменения, сохраняя прошлые версии.

## 2. Структура файлов (в репозитории)
```
/ru/index.html
/en/index.html
/index.html
.github/workflows/publish-on-release.yml
```

## 3. Как работает публикация (пошагово)
1. Вы создаете Release в GitHub (Tags -> Release), например с тегом `1.0` (или `v1.0`).
2. GitHub Actions получает webhook `release.published` и запускает workflow.
3. Workflow:
   - Проверяет репозиторий.
   - Получает тег релиза `github.event.release.tag_name`.
   - Клонирует ветку `gh-pages`.
   - Копирует файлы сайта в директорию `v<tag>` (например `v1.0`).
   - Добавляет/коммитит/пушит изменения в `gh-pages`.
4. GitHub Pages настроенная на ветку `gh-pages` раздаёт статические файлы. Сайт будет доступен по адресу:
```
https://<ваш-пользователь>.github.io/<ваш-репозиторий>/v1.0/
```

> Примечание: если ваш релиз использует название без префикса `v` (например `1.0`), workflow всё равно создаёт каталог `v1.0`. Если хотите изменить формат — отредактируйте шаг `TARGET_DIR` в workflow.

## 4. Доступ к прошлым версиям
Поскольку каждый релиз публикуется в отдельную папку (`v<tag>`), все предыдущие версии сохраняются в ветке `gh-pages` и остаются доступными, например:
- `/v0.9/`
- `/v1.0/`
- `/v2.3/`

Также добавлена папка `latest`, которая всегда перезаписывается последним релизом — удобно для ссылки на актуальную версию:
```
https://<user>.github.io/<repo>/latest/
```

## 5. Настройка GitHub Pages (инструкции и куда сделать скриншоты)
1. Откройте репозиторий на GitHub -> Settings -> Pages.
   - Выберите ветку: **gh-pages**.
   - Путь: **/** (root).
   - Сохраните.
   - **Скриншот 1:** сделайте снимок настроек Pages (ветка + путь). Подпишите: «Выбрана ветка gh-pages; страницы будут собираться/раздаваться из корня ветки».
2. Если в разделе Pages доступна настройка источника сборки — убедитесь, что выбрана ветка `gh-pages` и корень.
   - **Скриншот 2:** страница информации о публикации (URL сайта) — подпишите, что это итоговый URL для доступа.

## 6. Скриншоты CI/CD и пояснение шагов (куда вставить скриншоты)
Для отчета приложите:
- **Скриншот 3:** Логи выполненного workflow (Actions -> ваш workflow -> run) — подпишите, что показан запуск при релизе.
- Описание шагов workflow (ниже — детально).

### Описание шагов workflow (как в `.github/workflows/publish-on-release.yml`)
- **Checkout repository** — получает код.
- **Set variables** — определяет `RELEASE_TAG`.
- **Prepare site build** — место для сборки (если у вас SSG — npm run build и т.п.).
- **Clone (or create) gh-pages branch** — клонирует gh-pages ветку (или создаёт её).
- **Copy site into versioned folder** — копирует статические файлы в `v{tag}`, а также обновляет `latest`.
- **Commit & Push** — отправляет изменения в `gh-pages`.

> **Скриншот 4:** экран с конкретным шагом `Copy site into versioned folder` и его логами.

## 7. Скриншоты web-приложения
- Откройте любую опубликованную версию, сделайте скриншот страницы, чтобы в адресной строке виден URL вида:
  ```
  https://<user>.github.io/<repo>/v1.0/
  ```
  - **Скриншот 5:** ru/index.html, виден адрес.
  - **Скриншот 6:** en/index.html, виден адрес.

## 8. Код pipeline
Ниже — содержимое файла `.github/workflows/publish-on-release.yml` (тот же, что в репозитории):

```yaml
name: Publish site on release

on:
  release:
    types: [published]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set variables
        run: |
          echo "REPO_NAME=${{ github.event.repository.name }}" >> $GITHUB_ENV
          echo "REPO_OWNER=${{ github.repository_owner }}" >> $GITHUB_ENV
          # tag name or release name
          echo "RELEASE_TAG=${{ github.event.release.tag_name }}" >> $GITHUB_ENV
          if [ -z "${{ github.event.release.tag_name }}" ]; then
            echo "RELEASE_TAG=${{ github.event.release.name }}" >> $GITHUB_ENV
          fi
          echo "TARGET_DIR=v${{ github.event.release.tag_name }}" >> $GITHUB_ENV

      - name: Prepare site build (copy static files)
        run: |
          # If you have a build step (npm run build, hugo, etc.) run it here.
          # This example uses the repository root files as the site.
          SITE_DIR=$(pwd)
          echo "SITE_DIR=${SITE_DIR}"
          ls -la

      - name: Clone (or create) gh-pages branch
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"
          # clone gh-pages branch if exists, otherwise create orphan
          git clone --single-branch --branch gh-pages https://x-access-token:${GH_TOKEN}@github.com/${{ github.repository }} gh-pages ||           (git clone https://x-access-token:${GH_TOKEN}@github.com/${{ github.repository }} gh-pages && cd gh-pages && git checkout --orphan gh-pages)
          cd gh-pages
          git pull origin gh-pages || true

      - name: Copy site into versioned folder
        run: |
          cd gh-pages
          mkdir -p "v${{ github.event.release.tag_name }}"
          # Remove old content for this version and copy new files
          rm -rf "v${{ github.event.release.tag_name }}"/*
          cp -R ../ru ./v${{ github.event.release.tag_name }}/ru
          cp -R ../en ./v${{ github.event.release.tag_name }}/en
          cp ../index.html ./v${{ github.event.release.tag_name }}/index.html
          # optional: update a "latest" symlink-like folder by copying as 'latest'
          rm -rf latest
          mkdir -p latest
          cp -R ../ru ./latest/ru
          cp -R ../en ./latest/en
          cp ../index.html ./latest/index.html
          git status --porcelain
          git add -A
          if git diff --cached --quiet; then
            echo "No changes to commit"
          else
            git commit -m "Publish site for release ${{ github.event.release.tag_name }}"
            git push origin gh-pages
          fi

      - name: Done
        run: echo "Site published to gh-pages branch under /v${{ github.event.release.tag_name }}/"
```

(Замените `name: Publish site on release

on:
  release:
    types: [published]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set variables
        run: |
          echo "REPO_NAME=${{ github.event.repository.name }}" >> $GITHUB_ENV
          echo "REPO_OWNER=${{ github.repository_owner }}" >> $GITHUB_ENV
          # tag name or release name
          echo "RELEASE_TAG=${{ github.event.release.tag_name }}" >> $GITHUB_ENV
          if [ -z "${{ github.event.release.tag_name }}" ]; then
            echo "RELEASE_TAG=${{ github.event.release.name }}" >> $GITHUB_ENV
          fi
          echo "TARGET_DIR=v${{ github.event.release.tag_name }}" >> $GITHUB_ENV

      - name: Prepare site build (copy static files)
        run: |
          # If you have a build step (npm run build, hugo, etc.) run it here.
          # This example uses the repository root files as the site.
          SITE_DIR=$(pwd)
          echo "SITE_DIR=${SITE_DIR}"
          ls -la

      - name: Clone (or create) gh-pages branch
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"
          # clone gh-pages branch if exists, otherwise create orphan
          git clone --single-branch --branch gh-pages https://x-access-token:${GH_TOKEN}@github.com/${{ github.repository }} gh-pages ||           (git clone https://x-access-token:${GH_TOKEN}@github.com/${{ github.repository }} gh-pages && cd gh-pages && git checkout --orphan gh-pages)
          cd gh-pages
          git pull origin gh-pages || true

      - name: Copy site into versioned folder
        run: |
          cd gh-pages
          mkdir -p "v${{ github.event.release.tag_name }}"
          # Remove old content for this version and copy new files
          rm -rf "v${{ github.event.release.tag_name }}"/*
          cp -R ../ru ./v${{ github.event.release.tag_name }}/ru
          cp -R ../en ./v${{ github.event.release.tag_name }}/en
          cp ../index.html ./v${{ github.event.release.tag_name }}/index.html
          # optional: update a "latest" symlink-like folder by copying as 'latest'
          rm -rf latest
          mkdir -p latest
          cp -R ../ru ./latest/ru
          cp -R ../en ./latest/en
          cp ../index.html ./latest/index.html
          git status --porcelain
          git add -A
          if git diff --cached --quiet; then
            echo "No changes to commit"
          else
            git commit -m "Publish site for release ${{ github.event.release.tag_name }}"
            git push origin gh-pages
          fi

      - name: Done
        run: echo "Site published to gh-pages branch under /v${{ github.event.release.tag_name }}/"` содержимым файла `publish-on-release.yml` — он приложен в архиве.)

---

## 9. Примечания по безопасности и токенам
Workflow использует автоматически предоставляемый `GITHUB_TOKEN` — он имеет права пушить в репозиторий; если у вас ограничены правила ветки, настройте разрешения для workflows или используйте персональный токен с нужными правами (хранить в Secrets).

## 10. Как протестировать локально
1. Создайте тестовый релиз в GitHub (Release -> Create release) с тегом `1.0`.
2. Проверьте Actions → workflow запустится.
3. После завершения зайдите в Settings → Pages и посмотрите URL или напрямую откройте:
```
https://<user>.github.io/<repo>/v1.0/
```

---

## 10. Что приложено в архиве
- Статические страницы (ru/, en/, index.html).
- Workflow GitHub Actions.
- Этот файл отчета (`report.md`) и README.

--- 

Пожалуйста, распакуйте архив и используйте инструкции для подключения к вашему репозиторию GitHub.