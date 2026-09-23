# AI Assignment Review

Автоматическая проверка **pull request** студента в вашем учебном репозитории по критериям
из Markdown-файла (rubric) с помощью AI-провайдера с OpenAI-совместимым API
(по умолчанию — [Kodik Router](https://api.kodikrouter.ru/v1)).

Action уже опубликован как **`anst-foto/ai-review`**. Копировать или собирать его не нужно —
достаточно один раз настроить свой учебный репозиторий: добавить rubric, секрет с ключом и workflow.

Оценка (балл, вердикт «пройдено/не пройдено», сводка по критериям) появляется в
**GitHub Actions Summary**. PR автоматически **не комментируется** — при желании используйте
выходы Action дальше в workflow.

## Как это работает

1. При событии PR Action читает event payload GitHub и определяет номер PR.
2. Через GitHub API получает метаданные PR и **base commit SHA** — коммит, в который вливается PR.
3. Читает rubric из **base-версии** репозитория (по `base_sha`), чтобы критерии нельзя было подменить изменениями ученика.
4. Собирает список изменённых файлов PR (имя, статус, patch) через GitHub API.
5. Формирует промпт: rubric + сериализованный diff, помеченный как ненадёжное содержимое ученика, чтобы модель не выполняла инструкции из него.
6. Отправляет запрос провайдеру (`temperature=0`, `response_format: {"type":"json_object"}`) и валидирует ответ по схеме.
7. Считает `passed = (модель считает работу выполненной) И (балл >= passing-score)`.

Что Action **не** делает: не выполняет `checkout`, не запускает код ученика, не комментирует PR
автоматически и не усекает слишком большие diff-ы (вместо этого завершается ошибкой, чтобы не
выдать фиктивную оценку).

## Настройка (один раз)

Настройка состоит из трёх шагов в вашем учебном репозитории.

### 1. Создайте rubric

Добавьте файл критериев, например `.github/assignment-rubric.md`. Пример —
[`examples/rubric.md`](examples/rubric.md):

```markdown
# Критерии проверки задания

## Условие
Кратко опишите ожидаемый результат и ограничения задания здесь.

## Критерии и баллы
Общая сумма — 100 баллов.

- **Корректность (50 баллов):** решение реализует требования задания.
- **Качество кода (25 баллов):** понятные имена, разумная структура, нет явного дублирования.
- **Обработка граничных случаев (15 баллов):** обработаны пустые и некорректные входные данные.
- **Соответствие ограничениям (10 баллов):** соблюдены явно указанные ограничения задания.

## Обязательные требования
Перечислите требования, без которых работа не может считаться выполненной.
Если обязательное требование не выполнено, модель поставит `passed: false`.

## Формат обратной связи
Для каждого критерия указывайте статус `pass`, `partial`, `fail` или `not_applicable`
и кратко поясняйте оценку. Ссылайтесь только на то, что видно в diff; не утверждайте, что запускали тесты.
```

> Rubric читается из **base-коммита PR**, поэтому студент не может изменить критерии в самом PR.

### 2. Настройте секреты

В репозитории **Settings → Secrets and variables → Actions** создайте секрет
`KODIK_API_KEY` с ключом вашего провайдера. `GITHUB_TOKEN` GitHub подставляет сам.

### 3. Добавьте workflow

Создайте файл `.github/workflows/ai-review.yml`:

```yaml
name: AI assignment review

on:
  pull_request_target:
    types: [opened, synchronize, reopened, ready_for_review]

permissions:
  contents: read
  pull-requests: read

jobs:
  review:
    if: contains(fromJSON('["OWNER","MEMBER","COLLABORATOR"]'), github.event.pull_request.author_association)
    runs-on: ubuntu-latest
    steps:
      - name: Set up Python
        uses: actions/setup-python@a26af69be951a213d495a4c3e4e4022e16d87065 # v5.6.0
        with:
          python-version: "3.12"

      - name: Grade assignment
        id: grade
        uses: anst-foto/ai-review@057a4b6cd3d5ef85c7df4e1a856e5cf96c72f86a
        with:
          api-key: ${{ secrets.KODIK_API_KEY }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          rubric-path: .github/assignment-rubric.md
```

В примере Action закреплён по SHA релиза `v0.1`. Почему это безопаснее тега —
см. «Безопасность».

## Входы

| Вход | Обязательный | По умолчанию | Назначение |
| --- | --- | --- | --- |
| `api-key` | Да | — | Ключ OpenAI-совместимого API. Передавайте через GitHub Secret. |
| `github-token` | Да | — | Токен для чтения PR и rubric. Достаточно `contents: read` и `pull-requests: read`. |
| `rubric-path` | Да | — | Путь к Markdown-файлу критериев относительно корня репозитория (из base-коммита). |
| `model` | Нет | `qwen/qwen3.8-27b:free` | Идентификатор модели провайдера. |
| `base-url` | Нет | `https://api.kodikrouter.ru/v1` | Базовый URL OpenAI-совместимого API. |
| `passing-score` | Нет | `70` | Минимальный балл от 0 до 100 для прохождения. |
| `max-diff-chars` | Нет | `50000` | Максимальный размер итогового промпта (rubric + системный текст + diff). При превышении Action завершается ошибкой и данных не усекает. Минимум — 1000. |
| `pull-request-number` | Нет | из event payload | Номер PR; нужен только при ручном запуске (`workflow_dispatch`). |

## Выходы

Доступны в следующих шагах через `steps.<id>.outputs.<имя>`:

| Выход | Назначение |
| --- | --- |
| `score` | Балл от 0 до 100. |
| `passed` | `true`, только если модель отметила работу как выполненную **и** балл не ниже `passing-score`. |
| `summary` | Markdown-сводка по критериям. |

Пример использования выходов:

```yaml
      - name: Grade assignment
        id: grade
        uses: anst-foto/ai-review@057a4b6cd3d5ef85c7df4e1a856e5cf96c72f86a
        with:
          api-key: ${{ secrets.KODIK_API_KEY }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          rubric-path: .github/assignment-rubric.md

      - name: Show result
        run: |
          echo "Score: ${{ steps.grade.outputs.score }}/100"
          echo "Passed: ${{ steps.grade.outputs.passed }}"
```

## Ручной запуск (`workflow_dispatch`)

Проверку можно запускать вручную из **Actions → Run workflow**, добавив триггер с входным
параметром номера PR и передав его в `pull-request-number`:

```yaml
on:
  workflow_dispatch:
    inputs:
      pr-number:
        description: "Номер PR для проверки"
        required: true

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - name: Set up Python
        uses: actions/setup-python@a26af69be951a213d495a4c3e4e4022e16d87065 # v5.6.0
        with:
          python-version: "3.12"

      - name: Grade assignment
        id: grade
        uses: anst-foto/ai-review@057a4b6cd3d5ef85c7df4e1a856e5cf96c72f86a
        with:
          api-key: ${{ secrets.KODIK_API_KEY }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          rubric-path: .github/assignment-rubric.md
          pull-request-number: ${{ inputs.pr-number }}
```

При `workflow_dispatch` payload события не содержит `pull_request.number`, поэтому без
`pull-request-number` Action завершится ошибкой «requires a pull_request event or the
`pull-request-number` input». При обычном `pull_request` / `pull_request_target` номер
определяется автоматически.

## Безопасность

Workflow в примере использует `pull_request_target`, чтобы иметь доступ к секретам и читать
diff PR из форка. Это чувствительный триггер: он выполняется в контексте базового репозитория
с доступом к секретам, хотя PR приходит из внешнего форка.

Чтобы оставаться в безопасном режиме:

- **Не** выполняйте `checkout` ветки `head` PR и **не** устанавливайте зависимости из PR — иначе запустится код ученика.
- Ограничьте права минимумом: `contents: read` и `pull-requests: read`.
- **Закрепите** Action по полному commit SHA, а не по изменяемому тегу `@v1`: владелец тега может подменить код, которому передан ключ.
- Помните, что diff задания отправляется выбранному AI-провайдеру. Не используйте решение для репозиториев, данные которых нельзя передавать этому провайдеру.

### Ограничение расхода квоты

Платный вызов провайдера происходит на каждый подходящий PR. Чтобы посторонние авторы форков
не сжигали квоту, пример ограничивает проверку участниками репозитория через `if:` по полю
`author_association` (`OWNER`, `MEMBER`, `COLLABORATOR`). Расширяйте список осознанно; для
сторонних участников можно запускать проверку вручную и выборочно — например, по метке
(`labeled`), а не на каждый `push`.

### Про обычный `pull_request`

Если использовать `pull_request` вместо `pull_request_target`, секреты недоступны для PR из
форка, и AI-проверка не сможет вызвать API. Используйте `pull_request_target` только после
соблюдения мер безопасности выше.

## Ограничения и поведение при ошибках

- Проверяется только текстовый diff. GitHub API может не вернуть patch для бинарных или очень больших файлов — в этом случае Action завершится ошибкой.
- GitHub может сократить очень большие patch-и без явного признака усечения — это ограничение Action надёжно обнаружить не может. Разбивайте такие PR на меньшие текстовые части.
- Файлы без текстового diff (чистые переименования, новые пустые файлы, удалённые файлы) пропускаются без ошибки.
- Невалидный ответ модели, недоступный/пустой rubric, отсутствие текстовых изменений или превышение лимита промпта приводят к ошибке вместо фиктивной оценки.
- Оценка формируется с `temperature=0` для воспроизводимости; код и тесты ученика не запускаются.