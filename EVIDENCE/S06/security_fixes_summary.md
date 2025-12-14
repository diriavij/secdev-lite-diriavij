# S06 Security Fixes Summary

## Применённые исправления безопасности

### 1. SQL Injection — Login (S06-01)

**Проблема:**  
В `app/main.py` SQL‑запрос для `/login` собирался через f‑строку:

```python
# До (уязвимо):
sql = f"SELECT id, username FROM users WHERE username = '{payload.username}' AND password = '{payload.password}'"
row = query_one(sql)
```

Это позволяло обойти аутентификацию с `username="admin'-- "` за счёт SQL‑комментария.

**Исправление:**  

- В `app/db.py` добавлена функция безопасного выполнения запросов с параметрами:

  ```python
  def query_one_params(sql: str, params: tuple):
      with get_conn() as conn:
          row = conn.execute(sql, params).fetchone()
          return dict(row) if row else None
  ```

- В `app/main.py` (`/login`) SQL‑запрос переписан на параметризованный:

  ```python
  # После (безопасно):
  sql = "SELECT id, username FROM users WHERE username = ? AND password = ?"
  row = query_one_params(sql, (payload.username, payload.password))
  ```

- Коммит: **f7a3b21**
- Тесты:
  - `tests/test_login_sql_injection.py::test_login_sql_injection_blocked` ✅
  - `tests/test_login_sql_injection.py::test_login_valid_credentials` ✅

---

### 2. SQL Injection — Search (S06-02)

**Проблема:**  
В `app/main.py` для `/search` использовалась f‑строка в `LIKE`:

```python
# До (уязвимо):
sql = f"SELECT id, name, description FROM items WHERE name LIKE '%{q}%'"
items = query(sql)
```

Инъекция `"' OR '1'='1"` в `q` приводила к выдаче всех записей.

**Исправление:**

- В `app/db.py` добавлена функция для параметризованных запросов, возвращающих список:

  ```python
  def query_params(sql: str, params: tuple):
      with get_conn() as conn:
          rows = conn.execute(sql, params).fetchall()
          return [dict(row) for row in rows]
  ```

- В `app/main.py` endpoint `/search` переписан на безопасную параметризацию:

  ```python
  @app.get("/search")
  def search(q: str | None = Query(
      default=None,
      min_length=1,
      max_length=100
  )):
      if q is None:
          return {"items": []}
      sql = "SELECT id, name, description FROM items WHERE name LIKE ?"
      pattern = f"%{q}%"
      items = query_params(sql, (pattern,))
      return {"items": items}
  ```

- Добавлено ограничение длины параметра `q` через FastAPI `Query(min_length=1, max_length=100)`, чтобы защититься от слишком длинных запросов.

- Коммит: **f7a3b21**
- Тесты:
  - `tests/test_search_sql_like.py::test_search_like_injection_blocked` ✅
  - `tests/test_search_sql_like.py::test_search_valid_query` ✅

---

### 3. XSS — небезопасный вывод (S06-03)

**Проблема:**  
В шаблоне `app/templates/index.html` сообщение выводилось как raw‑HTML:

```jinja2
<!-- До (уязвимо): -->
<div id="message">{{ message|safe }}</div>
```

Это позволяло отразить `<script>alert(1)</script>` как активный скрипт (reflected XSS через `/echo`).

**Исправление:**

- Убран фильтр `|safe`, задействовано автоэкранирование Jinja2:

  ```jinja2
  <!-- После (безопасно): -->
  <div id="message">{{ message }}</div>
  ```

- Дополнительно (defence in depth) в обработчике `/echo` можно экранировать ввод:

  ```python
  @app.get("/echo")
  def echo(request: Request, msg: str = ""):
      safe_msg = msg.replace("<", "&lt;").replace(">", "&gt;")
      return templates.TemplateResponse("index.html", {"request": request, "message": safe_msg})
  ```

- Коммит: **b5f6a33**
- Тесты:
  - `tests/test_xss_escaping.py::test_echo_escapes_script_tags` ✅
  - `tests/test_xss_escaping.py::test_echo_escapes_img_onerror` ✅

---

## Итоговая статистика тестов

```text
========================= 8 passed in 1.87s =========================
collected 8 items

tests/test_login_sql_injection.py::test_login_sql_injection_blocked PASSED [12%]
tests/test_login_sql_injection.py::test_login_valid_credentials PASSED [25%]
tests/test_search_sql_like.py::test_search_like_injection_blocked PASSED [37%]
tests/test_search_sql_like.py::test_search_valid_query PASSED [50%]
tests/test_xss_escaping.py::test_echo_escapes_script_tags PASSED [62%]
tests/test_xss_escaping.py::test_echo_escapes_img_onerror PASSED [75%]
tests/test_validation.py::test_login_username_too_long PASSED [87%]
tests/test_validation.py::test_login_username_invalid_chars PASSED [100%]
```

## Карточки безопасности (реализованы)

- ✅ **S06-01** — SQL Injection (login) — параметризация запросов
- ✅ **S06-02** — SQL Injection (search LIKE) — безопасная обработка LIKE‑паттернов
- ✅ **S06-03** — XSS (экранирование вывода) — удаление `|safe`, автоэкранирование

## Покрытие угроз STRIDE

- **T (Tampering):** SQL injection предотвращена через параметризацию.
- **I (Information Disclosure):** XSS предотвращена через экранирование вывода.
- **D (Denial of Service):** Ограничения на длину входов снижают риск DoS через большие payload’ы.

## Артефакты

- `EVIDENCE/S06/test-report.xml` — JUnit XML отчёт тестов (8 passed)
- `EVIDENCE/S06/one-liner.md` — документация команды сборки и тестов
- `EVIDENCE/S06/security_fixes_summary.md` — данный файл
- `EVIDENCE/S06/patches.log` — список коммитов

## Связь с требованиями (TM.md)

- **NFR-2 (Предотвращение SQL инъекций)** → реализовано через S06-01, S06-02 (login и search).
- **ADR-002 (Параметризованные запросы)** → применён в login и search.
- **R-02 (SQL injection, L×I=15)** → риск закрыт ✅.

## Проверяемость (DV.md)

- **DV1:** one-liner работает без интерактива и создаёт `EVIDENCE/S06/test-report.xml`.
- **DV3:** реализованы 4 карточки безопасности с тестами (SQLi login, SQLi search, XSS, validation).
- **DV4:** все артефакты S06 лежат в `EVIDENCE/S06/` и перечислены в DV.md.