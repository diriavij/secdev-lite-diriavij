# DV - Мини-проект «DevOps-конвейер»

> Этот файл - **индивидуальный**. Его проверяют по **rubric_DV.md** (5 критериев × {0/1/2} → 0-10).
> Все доказательства/скрины кладите в **EVIDENCE/** и ссылайтесь на конкретные файлы/якоря.

---

## 0) Мета

- **Проект:** Учебный шаблон `secdev-seed-s06-s08` (FastAPI + SQLite + pytest)
- **Версия (commit/date):** `f7a3b21` / 2024-01-21 (после S06–S08)
- **Кратко:** Локально и в CI собираемый Python-проект: one-liner для тестов, Docker-образ для контейнеризации, GitHub Actions workflow для автоматического прогона pytest и сборки артефактов.

---

## 1) Воспроизводимость локальной сборки и тестов (DV1)

**One-liner для локального запуска S06:**

```bash
python3 scripts/init_db.py && python3 -m pytest tests/ --junitxml=EVIDENCE/S06/test-report.xml -v
```

**Интерпретация:**

1. `python3 scripts/init_db.py` – создаёт и заполняет SQLite `app.db` тестовыми данными.
2. `python3 -m pytest tests/ --junitxml=EVIDENCE/S06/test-report.xml -v` – запускает все тесты из `tests/`, пишет JUnit-отчёт в `EVIDENCE/S06/test-report.xml`.

**Версии инструментов (фиксация):**

```bash
python --version     # Python 3.11+
pip --version
pip freeze > EVIDENCE/pip-freeze.txt
pytest --version     # pytest 7.x
```

(файл `EVIDENCE/pip-freeze.txt` используется как freeze-снимок окружения для воспроизводимости).

**Шаги локального запуска:**

1. Создать виртуальное окружение и установить зависимости:

   ```bash
   python3 -m venv .venv
   . .venv/bin/activate      # Windows: .venv\Scripts\activate
   pip install -r requirements.txt
   ```

2. Выполнить one-liner:

   ```bash
   python3 scripts/init_db.py && python3 -m pytest tests/ --junitxml=EVIDENCE/S06/test-report.xml -v
   ```

3. Посмотреть результаты в `EVIDENCE/S06/test-report.xml` (8 passed).

**Evidence DV1:**

- `EVIDENCE/S06/test-report.xml`
- `EVIDENCE/S06/one-liner.md`
- `EVIDENCE/pip-freeze.txt`

---

## 2) Контейнеризация (и гигиена секретов в контейнере, DV2)

**Dockerfile:** `Dockerfile` в корне репозитория.

Ключевые моменты:

- Базовый образ: `python:3.11-slim` (фиксированный тег, без `latest`).
- Установка зависимостей из `requirements.txt` с `--no-cache-dir`.
- Копируются только необходимые файлы: `app/`, `scripts/`, `requirements.txt`.
- Создаётся непривилегированный пользователь `appuser` (uid 10001); права на `/app`.
- `USER appuser` – приложение не работает от root.
- Добавлен `HEALTHCHECK` по `http://127.0.0.1:8000/`.
- CMD: `python scripts/init_db.py && uvicorn app.main:app --host 0.0.0.0 --port 8000`.

**Сборка/запуск локально:**

```bash
docker build -t secdev-seed:latest . 2>&1 | tee EVIDENCE/S07/build.log

docker run --rm -p 8000:8000 --name seed secdev-seed:latest \
  > EVIDENCE/S07/run.log 2>&1 &
sleep 5

# Проверка HTTP-кода корня
python - <<'PY' > EVIDENCE/S07/http_root_code.txt
import urllib.request
print(urllib.request.urlopen("http://127.0.0.1:8000/", timeout=5).getcode())
PY

# Инспект и non-root пруф
docker ps --filter "name=seed" --format "{{.Names}} {{.Status}}" > EVIDENCE/S07/ps.txt
docker inspect seed > EVIDENCE/S07/inspect_web.json
docker exec seed sh -c "id -u" > EVIDENCE/S07/non-root.txt
```

**Гигиена секретов:**

- В образ **не попадают**:
  - `.env` (в `.dockerignore`),
  - `.git/`, `EVIDENCE/`, `tests/`, `.venv/`.
- Конфигурация приложения в seed не требует реальных секретов; в будущем планируется использовать `.env.example` + `os.getenv`.
- При запуске в реальном окружении секреты передаются только через переменные окружения / секреты CI/CD, а не через `COPY .env`.

**Evidence DV2:**

- `EVIDENCE/S07/build.log`
- `EVIDENCE/S07/image-size.txt`
- `EVIDENCE/S07/run.log`
- `EVIDENCE/S07/ps.txt`
- `EVIDENCE/S07/http_root_code.txt` (должно быть `200`)
- `EVIDENCE/S07/inspect_web.json`
- `EVIDENCE/S07/non-root.txt` (uid ≠ 0)
- `.dockerignore` – исключает `.env`, `.git`, `.venv`, `EVIDENCE/`, `tests/`.

---

## 3) CI: базовый pipeline и стабильный прогон (DV3)

**Платформа CI:** GitHub Actions  
**Файл конфига CI:** `.github/workflows/ci.yml`

**Основные стадии:**

1. **on:** `push`, `pull_request` (и `workflow_dispatch` – ручной запуск).
2. `Checkout` (actions/checkout@v4).
3. `Setup Python 3.11` (actions/setup-python@v5).
4. `Cache pip` (actions/cache@v4, кэш `~/.cache/pip`).
5. `Install deps` – `pip install -r requirements.txt`.
6. `Init DB` – `python scripts/init_db.py`.
7. `Run tests` – `pytest -q --junitxml=EVIDENCE/S08/test-report.xml`.
8. `Upload artifacts` – actions/upload-artifact@v4 (`name: evidence-s08`, `path: EVIDENCE/S08/**`).

**Фрагмент ci.yml (ключевые шаги):**

```yaml
name: ci

on:
  push:
  pull_request:

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Cache pip
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-

      - name: Install deps
        run: pip install -r requirements.txt

      - name: Init DB
        run: python scripts/init_db.py

      - name: Run tests
        run: |
          mkdir -p EVIDENCE/S08
          pytest -q --junitxml=EVIDENCE/S08/test-report.xml
        continue-on-error: false

      - name: Upload artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: evidence-s08
          path: |
            EVIDENCE/S08/**
          if-no-files-found: warn
```

**Стабильность:**

- После исправления уязвимостей S06 все 8 тестов проходят и локально, и в CI.
- Последние раны CI зелёные; в случае падения тестов артефакты всё равно загружаются (`if: always()`).

**Evidence DV3:**

- `EVIDENCE/S08/ci-run.txt` – URL последнего удачного рана.
- `EVIDENCE/S08/test-report.xml` –JUnit-отчёт CI.
- `EVIDENCE/S08/ci-log.txt` – краткая выписка шагов job.
- `EVIDENCE/S08/junit-validate.txt` – проверка, что JUnit не пустой.

---

## 4) Артефакты и логи конвейера (DV4)

### S06 – локальные тесты secure coding

- `EVIDENCE/S06/test-report.xml` – pytest (8 тестов passed).
- `EVIDENCE/S06/one-liner.md` – документированная one-liner команда.
- `EVIDENCE/S06/security_fixes_summary.md` – описание исправлений SQLi/XSS/валидации.

### S07 – Docker/контейнерный запуск

- `EVIDENCE/S07/build.log` – полный лог сборки образа `secdev-seed:latest`.
- `EVIDENCE/S07/image-size.txt` – размер образа.
- `EVIDENCE/S07/run.log` – лог запуска контейнера `seed`.
- `EVIDENCE/S07/ps.txt` – статус контейнера.
- `EVIDENCE/S07/http_root_code.txt` – HTTP-код корня (`200`).
- `EVIDENCE/S07/inspect_web.json` – метаданные контейнера.
- `EVIDENCE/S07/non-root.txt` – uid процесса внутри контейнера (не root).

### S08 – CI / GitHub Actions

- `EVIDENCE/S08/ci-run.txt` – URL рана CI.
- `EVIDENCE/S08/test-report.xml` – JUnit отчёт тестов из CI.
- `EVIDENCE/S08/ci-log.txt` – выписка ключевых шагов CI.
- `EVIDENCE/S08/junit-validate.txt` – проверка валидности отчёта.
- `EVIDENCE/S08/secrets-proof.txt` – подтверждение отсутствия утечек секретов.

**Индекс артефактов DV:**

| Тип     | Файл в `EVIDENCE/`             | Комментарий                          |
|---------|---------------------------------|--------------------------------------|
| CI-лог  | `S08/ci-log.txt`               | Краткий лог шагов GitHub Actions     |
| Лок.лог | `S06/test-report.xml`          | Результаты локального pytest         |
| Docker  | `S07/build.log`, `S07/run.log` | Сборка/запуск контейнера             |
| Freeze  | `pip-freeze.txt`               | Локальное окружение Python           |
| Grep    | `S08/secrets-proof.txt`        | Гигиена секретов в CI                |

---

## 5) Секреты и переменные окружения (DV5 – гигиена)

**Шаблон окружения:**

- В текущем учебном проекте реальные секреты (API токены, пароли к внешним сервисам) не используются.
- Планируется использовать файл `.env.example` как шаблон (без значений) и читать значения через `os.getenv` в коде/конфиге.

**Хранение и передача в CI:**

- В текущем workflow GitHub Actions **секреты не используются** – все шаги основаны на публичных командах (checkout, pip install, pytest).
- В будущем при необходимости секреты будут храниться в **Actions → Settings → Secrets and variables** (repository secrets).
- Значения секретов **не будут** записываться в лог CI или в файлы в `EVIDENCE/`.

**Проверка отсутствия секретов в коде/логе:**

```bash
# Пример простой проверки
git grep -nE 'SECRET|TOKEN|password=|AKIA' || true
```

Вывод можно сохранить в `EVIDENCE/grep-secrets.txt` (пустой/без находок).

**Evidence DV5:**

- `.dockerignore` – исключает `.env` и другие чувствительные файлы из образа.
- `EVIDENCE/S08/secrets-proof.txt` – явное подтверждение, что секреты не используются/не утекли в YAML/логах.
- `EVIDENCE/S07/build.log` – без утечки конфигурации/секретов.

---

## 6) Индекс артефактов DV

| Тип     | Файл в `EVIDENCE/`             | Дата/время   | Коммит/версия | Runner/OS    |
|---------|--------------------------------|--------------|---------------|--------------|
| CI-лог  | `S08/ci-log.txt`               | 2024-01-21   | `f7a3b21`     | `gha-ubuntu` |
| JUnit   | `S08/test-report.xml`          | 2024-01-21   | `f7a3b21`     | `gha-ubuntu` |
| Лок.тесты | `S06/test-report.xml`        | 2024-01-21   | `f7a3b21`     | `local`      |
| Docker  | `S07/build.log`, `S07/run.log` | 2024-01-20   | `f7a3b21`     | `local`      |
| Freeze  | `pip-freeze.txt`               | 2024-01-19   | `f7a3b21`     | `local`      |
| Secrets | `S08/secrets-proof.txt`        | 2024-01-21   | `f7a3b21`     | `gha-ubuntu` |

---

## 7) Связь с TM и DS (hook)

- **TM:**  
  Конвейер (one-liner + Docker + CI) поддерживает требования TM:
  - NFR-1/3 (SQLi/валидация) – тесты S06 и CI гарантируют, что патчи не сломались.
  - NFR-2 (XSS) – pytest XSS-сценарии входят в CI.
  - NFR-4 (non-root) – Dockerfile и `EVIDENCE/S07/non-root.txt` подтверждают рантайм-безопасность.

- **DS:**  
  В будущем S09–S12:
  - SAST/Secrets (Semgrep, GitLeaks) → отчёты в `EVIDENCE/S10/`.
  - SBOM/SCA (Syft, Grype) → `EVIDENCE/S09/`.
  - DAST (ZAP) → `EVIDENCE/S11/`.
  - Container scan (Trivy) → `EVIDENCE/S12/`.

---

## 8) Самооценка по рубрике DV (0/1/2)

- **DV1. Воспроизводимость локальной сборки и тестов:** [ ] 0 [ ] 1 [x] 2  
  *One-liner работает, версии зафиксированы, тесты воспроизводимы локально.*

- **DV2. Контейнеризация (Docker/Compose) и гигиена секретов в контейнере:** [ ] 0 [ ] 1 [x] 2  
  *Образ `secdev-seed:latest` собирается, контейнер отвечает 200 на `/`, работает не от root; `.env`/секреты не попадают в образ.*

- **DV3. CI: базовый pipeline и стабильный прогон:** [ ] 0 [ ] 1 [x] 2  
  *GitHub Actions запускается на push/PR, выполняет install → init DB → pytest → upload artifacts; последние раны зелёные.*

- **DV4. Артефакты и логи конвейера:** [ ] 0 [ ] 1 [x] 2  
  *Собраны и задокументированы артефакты S06/S07/S08 в `EVIDENCE/`; есть индекс и ссылки.*

- **DV5. Секреты и конфигурация окружения (гигиена):** [ ] 0 [ ] 1 [x] 2  
  *Секреты не захардкожены; в текущем workflow не используются; предусмотрен шаблон `.env.example`; `.env` не коммитится; это подтверждено в `secrets-proof.txt`.*

**Итог DV (сумма):** **10/10**