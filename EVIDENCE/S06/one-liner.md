# S06 - Secure Coding One-liner

## One-liner команда для полного цикла сборки и тестирования (DV1):

```bash
python3 scripts/init_db.py && python3 -m pytest tests/ --junitxml=EVIDENCE/S06/test-report.xml -v
```

## Требования:
- Python 3.9+ (рекомендуется 3.11+)
- Зависимости установлены: `pip install -r requirements.txt`
- (Рекомендуется) активированное виртуальное окружение `.venv`

## Что делает команда:
1. `python3 scripts/init_db.py` — инициализирует SQLite базу данных `app.db` с тестовыми данными (пользователи, товары).
2. `python3 -m pytest tests/ --junitxml=EVIDENCE/S06/test-report.xml -v` — запускает все тесты из каталога `tests/` и генерирует XML‑отчёт в `EVIDENCE/S06/test-report.xml`.

## Результат:
- Отчёт тестов: `EVIDENCE/S06/test-report.xml` (формат JUnit XML).
- Код выхода:
  - `0` — если все тесты прошли успешно (8 passed)
  - `≠0` — если есть упавшие тесты или ошибки запуска.
