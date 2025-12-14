# S06 - Secure Coding Checklist

---

## Паспорт выполнения

* Команда/репозиторий: **secdev-seed-s06-s08**
* Коммит(ы) с фиксом: **<SHA_SQLI_LOGIN>, <SHA_SQLI_SEARCH>, <SHA_XSS>** (короткие SHA из `git log --oneline`)
* One-liner (DV1):  
  **`python scripts/init_db.py && python -m pytest tests/ --junitxml=EVIDENCE/S06/test-report.xml -q`**  
  Отчёт тестов: `EVIDENCE/S06/test-report.xml`
* Кратко «что было → что сделали»:  
  **До:** SQL injection в `/login` и `/search`, отражённая XSS в `/echo`.  
  **После:** Параметризованные SQL‑запросы в login/search, убран небезопасный `|safe` в шаблоне, все S06‑тесты зелёные.

---

## Выбранные карточки (отметьте не менее двух)

* [x] **S06-01 - SQLi (login)**
* [x] **S06-02 - SQLi (search LIKE)**
* [x] **S06-03 - XSS (экранирование вывода)**
* [ ] **S06-04 - Валидация ввода (строгость/длины/паттерны)**
* [ ] **S06-05 - Ошибки и логи без утечек**
* [ ] **S06-06 - Security-заголовки (минимум)**
* [ ] **S06-07 - Гигиена секретов и конфигов**
* [ ] **S06-08 - Anti-bruteforce (лайт)**

---

## 1) SQL Injection - login (`POST /login`)

* [x] Параметризация запроса (никаких f-строк)
* [x] Тест(ы) подтверждают блокировку `admin'--`
* [ ] (Опц.) Ограничения на логин/пароль в модели

**Пруфы:**  
`EVIDENCE/S06/test-report.xml`  
(опц.) `EVIDENCE/S06/patches/login_sqli.diff`

**Коммиты:** **670d1dd**

**Комментарий:**  
До: `app/main.py` собирал SQL через f‑строку (`... username = '{payload.username}' ...`), что позволяло обойти логин с `admin'--`.  
После: добавлен helper `query_one_params(sql, params)` в `app/db.py`, а `/login` использует параметризованный запрос:

```python
sql = "SELECT id, username FROM users WHERE username = ? AND password = ?"
row = query_one_params(sql, (payload.username, payload.password))
```

Тест `tests/test_login_sql_injection.py::test_login_should_not_allow_sql_injection` теперь возвращает 401, а не 200.

---

## 2) SQL Injection - search (`GET /search?q=...`)

* [x] Параметризация `LIKE ?` + шаблон `%q%`
* [x] Тест(ы) подтверждают, что инъекция не возвращает все записи
* [ ] (Опц.) Лимит длины `q`

**Пруфы:**  
`EVIDENCE/S06/test-report.xml`  
(опц.) `EVIDENCE/S06/patches/search_sqli.diff`  
(опц.) `EVIDENCE/S06/screenshots/search_injection_blocked.png`

**Коммиты:** **<670d1dd>**

**Комментарий:**  
До: `/search` подставлял `q` напрямую в f‑строку `LIKE '%{q}%'`, и инъекция `"' OR '1'='1"` возвращала все элементы.  
После: добавлен helper `query_params(sql, params)` в `app/db.py`, а `/search` использует:

```python
@app.get("/search")
def search(q: str = Query(..., min_length=1, max_length=32)):
    sql = "SELECT id, name, description FROM items WHERE name LIKE ?"
    pattern = f"%{q}%"
    items = query_params(sql, (pattern,))
    return {"items": items}
```

Тест `tests/test_search_sql_like.py::test_search_should_not_return_all_on_injection` зелёный: инъекция больше не выдаёт все записи.

---

## 3) XSS - экранирование вывода (шаблоны Jinja2)

* [x] Убрано `|safe` или внедрено безопасное кодирование
* [x] Тест(ы) подтверждают отсутствие `<script>` в ответе

**Пруфы:**  
`EVIDENCE/S06/test-report.xml`  
(опц.) `EVIDENCE/S06/patches/xss_escaping.diff`  
(опц.) `EVIDENCE/S06/screenshots/xss_before_after.png`

**Коммиты:** **<670d1dd>**

**Комментарий:**  
До: в `app/templates/index.html` сообщение выводилось как `{{ message|safe }}`, что позволяло отразить `<script>alert(1)</script>` как активный тег.  
После: фильтр `|safe` удалён, используется дефолтное автоэкранирование Jinja2:

```jinja2
<div id="message">{{ message }}</div>
```

Тест `tests/test_xss_escaping.py::test_echo_should_escape_script_tags` теперь проходит: в `resp.text` нет подстроки `<script>`.

---

## Артефакты S06 (минимум)

* [x] `EVIDENCE/S06/test-report.xml`
* [x] `EVIDENCE/S06/one-liner.md`
* [x] `EVIDENCE/S06/patches.log`
* [x] `EVIDENCE/S06/security_fixes_summary.md`

---

## Сводка для переноса в `GRADING/DV.md`

* **DV1 (Воспроизводимость):** one-liner → `python scripts/init_db.py && python -m pytest tests/ --junitxml=EVIDENCE/S06/test-report.xml -q`, отчёт → `EVIDENCE/S06/test-report.xml`
* **DV2 (Гигиена секретов):** на уровне S06 – базово, `.env` не коммитится (если добавлен `.gitignore`), продвинутая гигиена будет оформляться в DS‑части.
* **DV3 (Минимальная безопасность кода):** карточки: S06-01, S06-02, S06-03. SQLi в login/search устранён параметризацией, XSS устранена удалением `|safe`.
* **DV4 (Артефакты/логи):** `EVIDENCE/S06/test-report.xml`, `EVIDENCE/S06/one-liner.md`, (опц.) `EVIDENCE/S06/security_fixes_summary.md`, `EVIDENCE/S06/patches.log`.
* **DV5 (Инструкции):** в README.md добавлен раздел «Локальный запуск» с описанием one-liner и ссылкой на `EVIDENCE/S06/`.

---

## Definition of Done - перед сдачей

* [x] Выбраны **≥2** карточки, все тесты по ним **зелёные**
* [x] Коммиты/диффы понятны; ссылки на артефакты рабочие
* [x] One-liner без интерактива; отчёт в `EVIDENCE/S06/`
* [x] Секретов нет; `.env.example` присутствует (проверить и при необходимости добавить)