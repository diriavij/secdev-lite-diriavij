# S08 - CI Checklist

## Паспорт выполнения

* Репозиторий: **secdev-seed-s06-s08**
* Платформа CI: **GitHub Actions**
* Версия Python (раннер): **3.11**
* Ссылка на последний удачный ран: см. `EVIDENCE/S08/ci-run.txt`
* Основной JUnit-отчёт: `EVIDENCE/S08/test-report.xml` (скачан из артефактов CI)

---

## A. Триггеры и раннеры

* [x] `on: push` включён.
* [x] `on: pull_request` включён.
* [x] `paths-ignore` настроен для `EVIDENCE/**` и `README.md`.
* [x] Раннер: `ubuntu-latest`.

---

## B. Python & cache pip

* [x] Используется `actions/setup-python@v5` c Python 3.11.
* [x] `actions/cache@v4` настроен на `~/.cache/pip`.
* [ ] (опц.) Подтверждён speedup на втором ране (можно зафиксировать позже).

---

## C. Build → Init DB → Pytest

* [x] Установка зависимостей: `pip install -r requirements.txt`.
* [x] Инициализация БД: `python scripts/init_db.py`.
* [x] Тесты: `pytest -q --junitxml=EVIDENCE/S08/test-report.xml`.
* [ ] (опц.) Покрытие: `--cov=app --cov-report xml:coverage.xml` (пока не делали).

---

## D. Артефакты CI

* [x] Артефакты загружаются всегда (`if: always()`).
* [x] В артефакте есть `EVIDENCE/S08/test-report.xml`.
* [x] Локально в репозитории сохранены:
  * `EVIDENCE/S08/ci-run.txt`
  * `EVIDENCE/S08/test-report.xml`
  * `EVIDENCE/S08/artifacts.txt`
  * `EVIDENCE/S08/ci-log.txt`
  * `EVIDENCE/S08/junit-validate.txt`

---

## E. Гигиена секретов (DV2)

* [x] Секреты не захардкожены в YAML/логах.
* [x] В текущем workflow секреты GitHub Actions не используются.
* [x] В репозитории нет `.env`; при необходимости будет использоваться `.env.example`.

См. `EVIDENCE/S08/secrets-proof.txt`.

---

## F. Сводка для переноса в `GRADING/DV.md`

* **DV2 (секреты):** секреты GitHub Actions в данном workflow не используются, значений в YAML/логах нет; `.env` не коммитится, только `.env.example` (при необходимости).
* **DV4 (артефакты/логи):**
  * URL рана → `EVIDENCE/S08/ci-run.txt`
  * JUnit → `EVIDENCE/S08/test-report.xml`
  * CI лог (кратко) → `EVIDENCE/S08/ci-log.txt`
  * Проверка JUnit → `EVIDENCE/S08/junit-validate.txt`
* **DV5 (инструкции):** в README добавлен раздел «Как устроен CI» (см. ниже), отсылающий к GitHub Actions и артефактам `evidence-s08`.

---

## Definition of Done

* [x] CI триггерится на push/PR.
* [x] Шаги: checkout → setup-python → cache pip → install deps → init DB → pytest → upload artifacts.
* [x] В `EVIDENCE/S08/` лежат `ci-run.txt`, `test-report.xml`, `ci-log.txt`, `junit-validate.txt`, `secrets-proof.txt`.
* [x] `GRADING/DV.md` и README обновлены (DV2/DV4/DV5).
