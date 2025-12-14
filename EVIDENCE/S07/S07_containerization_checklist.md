# S07 - Containerization Checklist

> Цель: аккуратно «упаковать» one-liner из S06 в контейнер (Docker), собрать проверяемые артефакты и обновить DV.

---

## Паспорт выполнения

* Команда/репозиторий: **secdev-seed-s06-s08**
* Тег образа: **secdev-seed:latest**
* Вариант запуска: ☑ Docker  ☐ Compose
* One-liner из S06 (для справки):  
  `python3 scripts/init_db.py && python3 -m pytest tests/ --junitxml=EVIDENCE/S06/test-report.xml -v`
* Ссылки на артефакты S07: см. «Индекс пруфов» ниже.

---

## A. Образ (build)

* [x] **Pinned base image** — `python:3.11-slim` (фиксированный тег, без `latest`).
* [x] **`.dockerignore`** присутствует и исключает мусор (`__pycache__/`, `.venv/`, `.git/`, `EVIDENCE/`, `tests/`, `.env`).
* [x] **WORKDIR, COPY, RUN**: зависимости ставятся из `requirements.txt` с `--no-cache-dir`, копируются только `app/`, `scripts/`, `requirements.txt`.
* [x] **Сборка воспроизводима**:

  ```bash
  docker build -t secdev-seed:latest .
  ```

* [x] **Лог сборки сохранён** → `EVIDENCE/S07/build.log`.
* [x] **Размер образа зафиксирован** → `EVIDENCE/S07/image-size.txt`.

**DV:** DV4 (артефакты), DV5 (описание сборки в README).

---

## B. Запуск (run)

* [x] Порт проброшен (`-p 8000:8000`).
* [x] Контейнер поднимается командой:

  ```bash
  docker run --rm
  ```