Лабораторная работа №2. Настройка CI/CD пайплайна на GitHub Actions
Студент: Сергей Шнитко
Репозиторий: https://github.com/temmie4real/arpolab1

Цель работы
Изучение принципов построения декларативных сценариев непрерывной интеграции (CI) на платформе GitHub Actions. Приобретение навыков автоматизации процессов верификации структуры проекта (Sanity Check), безопасного управления секретами репозитория и настройки процессов автоматического зеркалирования исходного кода в резервную инфраструктуру.

Ход работы
Шаг 1: Создание резервного репозитория для зеркалирования
Зайдя на свой GitHub, создал второй, пустой репозиторий и назвал его arpolab2. При создании не добавлял файлы README или .gitignore, чтобы GitHub Actions мог беспрепятственно зеркалировать туда код. Скопировал URL-адрес резервного репозитория.

(docs/10.jpg)

Шаг 2: Генерация токена доступа (PAT Token)
В правом верхнем углу GitHub нажал на своё фото профиля → Settings. В самом низу левого меню выбрал Developer settings. Перешёл в Personal access tokens → Tokens (classic). Нажал Generate new token (classic). Написал название Backup-Token и поставил галочки напротив пунктов repo (доступ к репозиториям) и workflow. Нажал Generate token внизу страницы и обязательно скопировал появившийся длинный буквенно-цифровой код (он показывается только один раз).

(docs/11.jpg)

Шаг 3: Сохранение токена в секреты основного проекта
Перешёл на страницу основного репозитория arpolab1. Открыл вкладку Settings (верхняя панель). В левом меню развернул вкладку Secrets and variables и нажал Actions. Нажал большую зелёную кнопку New repository secret. В поле Name ввёл строго заглавными буквами: BACKUP_TOKEN. В поле Value вставил скопированный на Шаге 2 длинный токен. Нажал Add secret.

(docs/12.jpg)

Шаг 4: Написание единого YAML пайплайна
Открыл свою среду разработки в корне проекта игры. Создал папку с именем .github, внутри неё создал папку workflows (имена папок критически важны, строго с маленькой буквы и через точку: .github/workflows/). Внутри папки workflows создал файл main.yml и вставил в него универсальный код автоматизации, заменив nbrouka/2d_platformer и nbrouka/2d_platformer_backup на свои значения:

yaml
# Название всего автоматического процесса
name: Unity 2D Platformer CI/CD

# Условие запуска: реагировать на любой push в главную ветку main
on:
  push:
    branches: [ "main" ]

jobs:
  # ЗАДАЧА 1: Быстрая диагностика структуры проекта (Sanity Check)
  sanity_check:
    runs-on: ubuntu-latest # Запуск на бесплатном облачном сервере Linux
    steps:
      - name: Checkout code
        uses: actions/checkout@v4 # Скачиваем код проекта на облачный сервер

      - name: Check Unity Directories
        run: |
          echo "=== Проверка наличия метаданных Unity ==="
          if [ -d "ProjectSettings" ]; then echo "ProjectSettings найден."; else echo "Ошибка!" && exit 1; fi
          if [ -d "Packages" ]; then echo "Packages найден."; else echo "Ошибка!" && exit 1; fi

      - name: Locate 2D Platformer Scripts
        run: |
          echo "=== Поиск C# скриптов в папке Assets ==="
          find Assets/ -name "Controller.cs" -print
          find Assets/ -name "BuildManager.cs" -print

  # ЗАДАЧА 2: Автоматическое зеркалирование в резервный репозиторий
  mirror_repo:
    needs: sanity_check # Начнется только после того, как успешно пройдет первая задача
    if: github.repository == 'temmie4real/arpolab1' # Только в оригинальном репо, чтобы избежать цикла в backup
    runs-on: ubuntu-latest
    steps:
      - name: Checkout full history
        uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Скачиваем полную историю коммитов для корректного зеркалирования
          persist-credentials: false # Отключаем credential helper - чтобы GITHUB_TOKEN не перекрывал PAT

      - name: Verify token is set
        env:
          BACKUP_TOKEN: ${{ secrets.BACKUP_TOKEN }}
        run: |
          if [ -z "$BACKUP_TOKEN" ]; then
            echo "=== Ошибка: BACKUP_TOKEN is empty! Create a PAT with scopes [repo, workflow] ==="
            exit 1
          else
            echo "=== BACKUP_TOKEN is set (length: ${#BACKUP_TOKEN}) ==="
          fi

      - name: Push to Backup Repository
        env:
          BACKUP_TOKEN: ${{ secrets.BACKUP_TOKEN }}
        run: |
          echo "=== Начало процесса зеркалирования ==="
          git push --force https://x-access-token:${BACKUP_TOKEN}@github.com/temmie4real/arpolab2.git HEAD:main
          echo "== Код успешно продублирован =="
(docs/13.jpg)

Шаг 5: Отправка пайплайна на GitHub
Сохранил файл main.yml (Ctrl + S), создал новую ветку LR2 и добавил изменения в git:

git checkout -b LR2
git add .github/workflows/main.yml
git commit -m "feat: added CI/CD pipeline with sanity check and mirroring"
git push origin LR2

Зайдя в репозиторий на GitHub, нажал кнопку Compare & pull request. Создал PR, где base: main, а compare: LR2. После проверки нажал кнопку Merge pull request прямо на GitHub, объединив код LR2 с веткой main.

(docs/14.jpg)

Шаг 6: Проверка результатов и отчёт
Открыл основной репозиторий на сайте GitHub и перешёл во вкладку Actions. Увидел запущенный рабочий процесс Unity 2D Platformer CI/CD. Кликнул по нему — там отображаются две зелёные галочки напротив задач sanity_check и mirror_repo.

(docs/15.jpg)

Зайдя в резервный репозиторий arpolab2, обнаружил, что он перестал быть пустым — GitHub Actions сам полностью скопировал туда весь код, структуру папок и историю коммитов.

(docs/16.jpg)

Обновил файл README.md в основном репозитории, дополнив его шагами из ЛР№2 и скриншотами. Зафиксировал отчёт в истории Git отдельным коммитом и отправил в ветку main:

git add README.md
git commit -m "docs: Lab report #2 added"
git push origin main

Выводы
В ходе работы освоил построение декларативных CI-сценариев на GitHub Actions, настройку триггеров и зависимостей между задачами (needs), безопасное управление секретами через Repository Secrets, а также автоматическое зеркалирование исходного кода в резервный репозиторий с использованием Personal Access Token.