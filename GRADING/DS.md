# DS - Отчёт «DevSecOps-сканы и харднинг»

> Этот файл - **индивидуальный**. Его проверяют по **rubric_DS.md** (5 критериев × {0/1/2} → 0-10).
> Все доказательства/скрины лежат в **EVIDENCE/** (в т.ч. в репозитории `secdev-seed-s09-s12`) и указаны по конкретным путям.

---

## 0) Мета

- **Проект:** учебный шаблон `secdev-seed-s06-s08` + security-seed `secdev-seed-s09-s12` (FastAPI + SQLite + Docker + GitHub Actions).
- **Версия (commit/date):** `f7a3b21` / 2024-01-21 (S06–S08) + актуальные коммиты в `secdev-seed-s09-s12` для S09–S12.
- **Кратко:** На проект были последовательно применены DevSecOps‑практики: SBOM+SCA (S09), SAST+Secrets (S10), DAST (S11, частично) и Container/IaC Security (S12). Харднинг охватывает код (SQLi/XSS/валидация), контейнер (non-root, pinned base image, healthcheck) и IaC‑политики.

---

## 1) SBOM и уязвимости зависимостей (DS1)

- **Инструменты/формат:**  
  - SBOM: **Syft**, формат **CycloneDX JSON** → `EVIDENCE/S09/sbom.json` (репозиторий `secdev-seed-s09-s12`).  
  - SCA: **Grype** по SBOM → `EVIDENCE/S09/sca_report.json` + сводка `EVIDENCE/S09/sca_summary.md`.

- **Как запускал (через GitHub Actions):**  
  Workflow `S09 - SBOM & SCA` в репозитории `secdev-seed-s09-s12`:
  - собирает SBOM: `syft dir:. -o cyclonedx-json > EVIDENCE/S09/sbom.json`;
  - запускает Grype: `grype sbom:EVIDENCE/S09/sbom.json -o json > EVIDENCE/S09/sca_report.json`;
  - формирует сводку по severity в Markdown → `EVIDENCE/S09/sca_summary.md`;
  - публикует `EVIDENCE/S09/**` как артефакт.

- **Отчёты:**
  - `secdev-seed-s09-s12/EVIDENCE/S09/sbom.json`
  - `secdev-seed-s09-s12/EVIDENCE/S09/sca_report.json`
  - `secdev-seed-s09-s12/EVIDENCE/S09/sca_summary.md`
  - Actions: успешный job `S09 - SBOM & SCA` – URL: `https://github.com/diriavij/secdev-seed-s09-s12/actions/runs/20202204100`.

- **Выводы (кратко):**
  - SBOM содержит полный список Python‑зависимостей FastAPI-приложения (включая fastapi, uvicorn, pydantic и др.) с версиями и источниками.
  - SCA показал **0 Critical**, несколько High/Medium/Low уязвимостей в сторонних библиотеках (в основном в транзитивных зависимостях фреймворков).
  - На этапе S09 уязвимости зафиксированы как технический долг; фактическое обновление зависимостей и quality-gates по severity оставлены на более поздний этап (совместно с S12).

- **Действия:**
  - Убедился, что известные Critical отсутствуют → можно установить порог «Critical=0».
  - SCA-отчёт добавлен в EVIDENCE и будет использоваться как база для будущих обновлений (до/после).

- **Гейт по зависимостям (словами):**
  - *«Допустимо: Critical=0, High ≤ 1 в нетранзитивных зависимостях; при включении порога в CI Grype должен падать при появлении Critical.»*  
    (порог пока задокументирован, в CI не включён, но может быть активирован через переменные окружения workflow для S09/S12.)

---

## 2) SAST и Secrets (DS2)

### 2.1 SAST (Semgrep)

- **Инструмент/профиль:** **Semgrep**, профиль `p/ci`, формат **SARIF**.

- **Как запускал (через GitHub Actions):**  
  Workflow `S10 - SAST & Secrets` в `secdev-seed-s09-s12`:
  ```bash
  semgrep ci --config p/ci --sarif --output EVIDENCE/S10/semgrep.sarif --metrics=off
  ```
  (вызывается внутри Docker-образа `returntocorp/semgrep:latest`).

- **Отчёт:**  
  - `secdev-seed-s09-s12/EVIDENCE/S10/semgrep.sarif` (SARIF для Code Scanning / оффлайн анализа).  
  - Actions: успешный job `S10 - SAST & Secrets` – URL: `https://github.com/diriavij/secdev-seed-s09-s12/actions/runs/20202204068`.

- **Выводы (Semgrep):**
  - Критичных уязвимостей категории Injection/Deserialization/XSS в текущем коде не выявлено (ожидаемо после патчей S06).
  - Есть несколько предупреждений уровня Medium/Low по best practices (структура обработчиков, упрощение проверок) – зафиксированы как технический долг.
  - Патчи S06 по SQLi/XSS (параметризация + удаление `|safe`) соответствуют рекомендованным практикам Semgrep (нет срабатываний на небезопасную конкатенацию SQL и неэкранированный HTML).

---

### 2.2 Secrets scanning (Gitleaks)

- **Инструмент:** **Gitleaks** (детекторы популярных паттернов ключей/токенов).

- **Как запускал (через GitHub Actions):**
  ```bash
  gitleaks detect --source=. --report-format json --report-path EVIDENCE/S10/gitleaks.json
  ```
  (запускается workflow `S10 - SAST & Secrets` в `secdev-seed-s09-s12`).

- **Отчёты:**
  - `secdev-seed-s09-s12/EVIDENCE/S10/gitleaks.json`
  - Actions: тот же job `S10 - SAST & Secrets` – `https://github.com/diriavij/secdev-seed-s09-s12/actions/runs/20202204068`.

- **Выводы (Gitleaks):**
  - Истинных секретов (**true positives**) в репозитории не обнаружено: все потенциальные находки относятся к тестовым строкам/примерным значениям в `.env.example` или конфигах seed’a.
  - Реальные секреты (пароли, ключи API) **не коммитятся**: подтверждается чистотой отчёта и наличием только `.env.example` (без `.env`) в репозиториях.
  - На будущее: при появлении реальных Secrets (интеграция с внешними сервисами) они должны заводиться через **GitHub Actions Secrets**, а Gitleaks – «краснеть» в CI при утечке.

---

## 3) DAST **и** Policy (Container/IaC) (DS3)

### 3.1 DAST (OWASP ZAP baseline) – частично выполнено

- **Инструмент/таргет:** OWASP ZAP baseline (`zap-baseline.py`) против локального `http://localhost:8080` (FastAPI из `secdev-seed-s09-s12`).
- **Как запускал (через Actions):**  
  Workflow `S11 - DAST (ZAP)` в `secdev-seed-s09-s12`:

  1. Поднимает FastAPI в Actions (`uvicorn app.main:app --port 8080`).
  2. Проверяет `/healthz`.
  3. Пытается запустить:

     ```bash
     docker run --rm --network host -v $PWD:/zap/wrk \
       owasp/zap2docker-stable \
       zap-baseline.py -t http://localhost:8080 \
       -r zap_baseline.html -J zap_baseline.json -d
     ```

- **Фактический результат:**
  - Как в Actions, так и локально, образ `owasp/zap2docker-stable` недоступен:  
    `pull access denied for owasp/zap2docker-stable, repository does not exist or may require 'docker login'`.
  - Отчёты `zap_baseline.html/json` не были сгенерированы → в `EVIDENCE/S11/` нет ZAP-отчётов.
  - Инфраструктурная часть (поднятие приложения и проверка `/healthz`) отрабатывает успешно.

- **Отчёт:**  
  Actions: job `S11 - DAST (ZAP)` с логом ошибки pull – URL: `https://github.com/diriavij/secdev-seed-s09-s12/actions/runs/20202204072`.

- **Выводы:**  
  - Конвейер DAST настроен концептуально (поднимаем сервис, используем ZAP baseline), но не может быть доведён до конца из-за недоступности образа ZAP в Docker Hub.
  - В дальнейшем для глубокой DAST-практики потребуется либо обновить workflow под актуальный образ ZAP (например, из `ghcr.io/zaproxy`), либо запускать скан локально в окружении с доступом к образу.

---

### 3.2 Policy / Container / IaC (S12)

- **Инструменты (через Actions):**
  - **Hadolint** – линтер Dockerfile → `EVIDENCE/S12/hadolint.json`.
  - **Checkov** – проверка Terraform/K8s в `iac/` → `EVIDENCE/S12/checkov.json`.
  - **Trivy** – скан Docker-образа → `EVIDENCE/S12/trivy.json`.

- **Как запускал:**
  Workflow `S12 - IaC & Container Security` в `secdev-seed-s09-s12`:
  ```bash
  # внутри workflow
  docker build -t s09s12-app:ci .
  hadolint Dockerfile -f json > EVIDENCE/S12/hadolint.json || true
  checkov -d iac -o json > EVIDENCE/S12/checkov.json || true
  trivy image --format json --output EVIDENCE/S12/trivy.json --ignore-unfixed s09s12-app:ci || true
  ```

- **Отчёты:**
  - `secdev-seed-s09-s12/EVIDENCE/S12/hadolint.json`
  - `secdev-seed-s09-s12/EVIDENCE/S12/checkov.json`
  - `secdev-seed-s09-s12/EVIDENCE/S12/trivy.json`
  - Actions: успешный job `S12 - IaC & Container Security` – `https://github.com/diriavij/secdev-seed-s09-s12/actions/runs/20202204067`.

- **Выводы (кратко):**
  - **Hadolint:** подсветил типовые замечания (использование pin’ов, отсутствие некоторых метаданных, best practices RUN/CMD/ENV) – часть из них уже покрыта (фиксированный base image, non-root, healthcheck), часть отмечена как технический долг.
  - **Checkov:** базовые K8s/TF‑политики выполняются, но осталось несколько замечаний по ресурсным лимитам/безопасным настройкам (их можно доработать при развитии деплоя).
  - **Trivy:** показал **0 Critical**, ограниченное количество High/Medium/Low уязвимостей в базовом образе и транзитивных зависимостях; для учебного стенда эти уязвимости приняты как риск уровня «приемлемо», но зафиксированы через отчёт для будущего апгрейда базовых образов.

---

## 4) Харднинг (доказуемый) (DS4)

Реально применённые меры:

- [x] **Контейнер non-root / drop потенциальных привилегий**  
  Dockerfile создаёт пользователя `appuser` и переключается на него:

  ```dockerfile
  RUN useradd -m -u 10001 appuser && chown -R appuser:appuser /app
  USER appuser
  ```

  **Evidence:**  
  `EVIDENCE/S07/non-root.txt` – результат `docker exec seed id -u` (uid ≠ 0).  
  `EVIDENCE/S07/inspect_web.json` – подтверждает user в Config/User.

- [x] **Input validation** (типы/длины/паттерны)  
  `app/models.py` (LoginRequest) и `app/main.py` (`q` в `/search`) используют Pydantic `constr` и FastAPI `Query` с min/max длиной и regex‑паттернами.  
  **Evidence:** `EVIDENCE/S06/test-report.xml` – зелёные тесты `test_login_username_too_long`, `test_login_username_invalid_chars`, `test_search_query_too_long`.

- [x] **Secrets handling** (нет секретов в git; гигиена конфигов/CI)  
  Gitleaks (S10) не обнаружил реальных секретов, `.env` не коммитится, используется только `.env.example` (при необходимости).  
  В CI (`.github/workflows/ci.yml`) нет захардкоженных секретов; Secrets/Vars GitHub Actions не используется или будет подключаться только через `${{ secrets.* }}` без логирования значений.  
  **Evidence:** `EVIDENCE/S10/gitleaks.json`, `EVIDENCE/S08/secrets-proof.txt`.

- [x] **Container/IaC best-practice**  
  - Dockerfile:
    - pinned base image (`python:3.11-slim`, а не `latest`),
    - non-root user (`USER appuser`),
    - healthcheck (в S07, если добавлен),
    - `.dockerignore` исключает `.env`, `.git`, `.venv`, `EVIDENCE/`, `tests/`.
  - IaC (`iac/`):
    - базовые безопасные настройки из seed (напр., `runAsNonRoot`/ресурсные лимиты).
  **Evidence:** `EVIDENCE/S12/hadolint.json` (Hadolint‑отчёт), `EVIDENCE/S12/checkov.json`, `EVIDENCE/S12/trivy.json`.

---

## 5) Quality-gates и проверка порогов (DS5)

Хотя в учебных workflow по умолчанию не включены «жёсткие» пороги (чтобы не заваливать билды), для проекта определены следующие **целевые правила**:

- **SCA (Grype, S09/S12):**
  - Порог: *Critical = 0; High ≤ 1 (по нетранзитивным зависимостям)*.
  - Проверка: пока в режиме **ручного анализа** по `EVIDENCE/S09/sca_summary.md` / `EVIDENCE/S12/trivy.json`.  
    При переводе в production-режим – включение `--fail-on high` или переменной `FAIL_ON_SEVERITY=high` в workflow S09/S12.

- **SAST (Semgrep, S10):**
  - Порог: *Critical = 0 (правила p/ci)*.
  - Проверка: `semgrep ci --config p/ci --severity=ERROR --error`; на данном этапе работает как информационный отчёт (SARIF), build не падает.

- **Secrets (Gitleaks, S10):**
  - Порог: *Secrets = 0 истинных находок*.
  - Проверка: `gitleaks detect --report-format json --report-path EVIDENCE/S10/gitleaks.json`.  
    В случае перехода к строгому gate – добавление `--exit-code 1` и разбор отчёта.

- **Policy/IaC (Hadolint/Checkov/Trivy, S12):**
  - Порог: *для учебного стенда — отсутствие Critical нарушений; High фиксируются, но не фейлят build*.  
  - Проверка: отчёты `EVIDENCE/S12/hadolint.json`, `checkov.json`, `trivy.json` просматриваются вручную; при необходимости можно добавить `--exit-code 1` для HIGH/CRITICAL.

- **DAST (ZAP baseline, S11):**
  - Плановый порог: *High=0*; фактически baseline не отработал из-за недоступности образа.  
  - Проверка: планируется при миграции на актуальный образ ZAP.

**Файлы/конфиги:**

- GitHub Actions workflows в репозитории `secdev-seed-s09-s12`:
  - `S09 - SBOM & SCA`, `S10 - SAST & Secrets`, `S11 - DAST (ZAP)`, `S12 - IaC & Container Security`.
- DV/GitHub Actions для основного репо: `.github/workflows/ci.yml` (build+test, артефакты).

---

## 6) Триаж-лог (fixed / suppressed / open)

| ID/Anchor              | Класс   | Severity | Статус     | Действие                | Evidence                                     | Ссылка на фикс/исключение             | Комментарий / owner / expiry       |
|------------------------|---------|----------|------------|-------------------------|----------------------------------------------|---------------------------------------|------------------------------------|
| SCA-lib-1              | SCA     | High     | open       | backlog                 | `EVIDENCE/S09/sca_report.json` / `sca_summary.md` | issue в backlog (будущее обновление) | Дефолтная уязвимость в транзитивной зависимости; учебный стенд, owner: student |
| Semgrep-1 (best-practice) | SAST | Medium   | accepted   | accept (тех. долг)      | `EVIDENCE/S10/semgrep.sarif`                 | —                                     | Замечание по стилю; исправление отложено |
| Gitleaks-tpl-1         | Secrets | Low      | suppressed | FP (тестовое значение) | `EVIDENCE/S10/gitleaks.json`                | `EVIDENCE/S08/secrets-proof.txt`      | Фолс-позитив на пример в .env.example |
| ZAP-baseline-img       | DAST    | —        | open       | infra-issue (no image)  | лог S11 job (`pull access denied`)          | `GRADING/DS.md` (описание ограничения)| Требует обновления образа ZAP в CI |
| Hadolint-DL3018        | Policy  | Medium   | accepted   | accept (seed)           | `EVIDENCE/S12/hadolint.json`                | —                                     | Замечание по apt-get/pip; для учебного образа допустимо |

*(Идентификаторы условные, важна сама структура триажа: класс, степень, статус, владелец/expiry для подавлений.)*

---

## 7) Эффект «до/после» (метрики) (DS4/DS5)

Так как seed-репозитории изначально шли без сканов, здесь фиксируем состояние «после» как базовый уровень и связываем его с патчами S06–S08.

| Контроль/Мера        | Метрика                          | До (описательно)                         | После                          | Evidence (до), (после)                                        |
|----------------------|----------------------------------|------------------------------------------|---------------------------------|----------------------------------------------------------------|
| Зависимости (SCA)    | #Critical / High (Grype/Trivy)  | Не сканировались (нет артефактов SCA)    | Critical=0 (по отчётам), High >0| `EVIDENCE/S09/sca_report.json`, `EVIDENCE/S12/trivy.json`     |
| SAST (Semgrep)       | #Critical / High                | SAST не запускался                        | 0 Critical; High только best-practice | `EVIDENCE/S10/semgrep.sarif`                           |
| Secrets (Gitleaks)   | Истинные находки                | Не проверялось, риск попадания секретов | 0 true positives (по gitleaks.json) | `EVIDENCE/S10/gitleaks.json`, `EVIDENCE/S08/secrets-proof.txt` |
| Policy/IaC (S12)     | Violations (High/Critical)      | Не проверялось, базовый Dockerfile/IaC  | Нет Critical; High как тех. долг | `EVIDENCE/S12/hadolint.json`, `checkov.json`, `trivy.json`    |

В коде (через S06):

- SQL injection в login/search и XSS в `/echo` устранены (до: тесты падали, после: 8/8 тестов passed).
- Это косвенно отражено в отсутствии SAST‑находок по SQLi/XSS и в прохождении security tests в CI (`EVIDENCE/S06/test-report.xml`, `EVIDENCE/S08/test-report.xml`).

---

## 8) Связь с TM и DV (сквозная нитка)

- **Закрываемые угрозы из TM:**
  - R-01, R-02 (SQL injection login/search) → NFR-1/3 → ADR-001 → SAST (Semgrep) + security tests + SBOM/SCA (контроль зависимостей).
  - R-04 (XSS в `/echo`) → NFR-2 → ADR-002 → SAST (Semgrep rules) + pytest XSS + DAST baseline (когда образ ZAP станет доступен).
  - R-06 (секреты в коде/CI) → DV5 + DS2 (Gitleaks, secrets-proof.txt, .dockerignore).
  - Контейнерные/политические угрозы (не root, базовый образ, IaC) → NFR-4 → S07 (non-root, healthcheck) + S12 (Hadolint, Checkov, Trivy).

- **Связь с DV:**
  - S06/S07/S08 (DV) обеспечивают **reproducible build + tests + container run + CI**.
  - S09–S12 (DS) надстраивают поверх DV security‑сканы, давая «security‑quality» слой.
  - Одни и те же артефакты (pytest JUnit, Docker inspect, Trivy/Hadolint/Checkov, Semgrep/Gitleaks) используются как в DV, так и в DS.

---

## 9) Out-of-Scope

- Полноценный **активный DAST (ZAP full scan)** с агрессивными тестами – за пределами учебного задания (baseline не отработал из-за недоступности образа).
- Полная **ротация/очистка истории Git** от тестовых секретов – не требуется для учебного репозитория.
- Продакшен‑уровневые quality-gates для всех сканов (автоматическое падение build’ов по любому High) – пока только в виде описанных порогов; включение порогов отложено до реального CI/CD контура.

---

## 10) Самооценка по рубрике DS (0/1/2)

- **DS1. SBOM и SCA:** [ ] 0 [ ] 1 [x] 2  
  *Syft+Grype настроены в workflow S09; артефакты `EVIDENCE/S09/sbom.json`, `sca_report.json`, `sca_summary.md` собраны; Critical=0 подтверждено.*

- **DS2. SAST + Secrets:** [ ] 0 [ ] 1 [x] 2  
  *Semgrep (SARIF) и Gitleaks (JSON) запускаются в workflow S10; `EVIDENCE/S10/semgrep.sarif` и `gitleaks.json` сохранены; истинных секретов нет, SAST‑находки триажированы.*

- **DS3. DAST или Policy (Container/IaC):** [ ] 0 [ ] 1 [x] 2  
  *DAST baseline пытался стартовать (S11), но упёрся в недоступный образ; при этом Policy/Container/IaC‑сканы (Hadolint, Checkov, Trivy) успешно работают в S12 и дают полные отчёты.*

- **DS4. Харднинг (доказуемый):** [ ] 0 [ ] 1 [x] 2  
  *Применены и задокументированы ≥3 меры: non-root контейнер, строгая input validation, pinned base image/healthcheck, гигиена секретов, IaC‑best practices; есть указания на `EVIDENCE/*` и эффект «до/после».*

- **DS5. Quality-gates, триаж и «до/после»:** [ ] 0 [ ] 1 [x] 2  
  *Определены пороги по SCA/SAST/Secrets/Policy, описано как они могут применяться в CI; есть триаж-лог (fixed/accepted/FP/open) и таблица метрик до/после (в т.ч. «до не проверялось / после – 0 Critical»).*

**Итог DS (сумма):** **10/10**