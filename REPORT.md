# Web təhlükəsizliyi fənni — layihə hesabatı

**Layihə adı:** Viva La Diva Bakery (WebSec)  
**Məzmun:** Onlayn çörəkçi mağazası kontekstində veb tətbiq, PostgreSQL, Flask API və təhlükəsizlik lab ssenariləri

---

## Mündəricat

1. [Giriş və məqsəd](#1-giriş-və-məqsəd)
2. [Arxitektura və texnologiyalar](#2-arxitektura-və-texnologiyalar)
3. [Layihənin qurulması](#3-layihənin-qurulması)
4. [Backend: əsas komponentlər](#4-backend-əsas-komponentlər)
5. [Frontend](#5-frontend)
6. [Nəzərdə tutulmuş təhlükəsizlik "boşluqları" (lab)](#6-nəzərdə-tutulmuş-təhlükəsizlik-boşluqları-lab)
7. [Təhlükəsizlik yamaları və yaxşı təcrübələr](#7-təhlükəsizlik-yamaları-və-yaxşı-təcrübələr)
8. [Deploy (Replit)](#8-deploy-replit)
9. [Demo şəkilləri](#9-demo-şəkilləri)
10. [Nəticə və tövsiyələr](#10-nəticə-və-tövsiyələr)
11. [İstinadlar](#11-istinadlar)

---

## 1. Giriş və məqsəd

Bu layihə veb təhlükəsizliyi kursu çərçivəsində hazırlanmışdır. Məqsəd:

* real veb stack (Flask + PostgreSQL + statik frontend) üzərində **giriş/qeydiyyat**, **sessiya**, **REST API** qurmaq;
* **tədris məqsədli** zəiflikləri (SQL injection, IDOR, admin bypass, sessiya ilə bağlı ssenarilər) nümayiş etdirmək;
* müqayisəli olaraq **parametrləşdirilmiş sorğular** və **giriş yoxlamaları** ilə fərqi göstərmək;
* müəyyən **UI** üzərindən sızıntıların istifadəçiyə necə təqdim oluna biləcəyini təsvir etmək (simulyasiya).

Layihə **istehsal üçün hazır deyil**; təhlükəsizlik lab rejimi mühit dəyişənləri ilə idarə olunur.  
Mövzu kimi **Viva La Diva Bakery** — onlayn çörəkçi mağazası seçilmişdir.

---

## 2. Arxitektura və texnologiyalar

### 2.1. Ümumi sxem

```mermaid
flowchart LR
  subgraph client [Brauzer]
    HTML[HTML/CSS/JS]
  end
  subgraph server [Flask]
    API["/api/*"]
    Static[Statik səhifələr]
  end
  subgraph data [PostgreSQL]
    DB[(users, products, orders)]
  end
  HTML --> API
  HTML --> Static
  API --> DB
```

### 2.2. Texnologiyalar

| Təbəqə | Texnologiya |
| --- | --- |
| Backend | Python 3, Flask |
| Verilənlər bazası | PostgreSQL (`psycopg2`, `RealDictCursor`) |
| Frontend | Vanilla JS, bir səhifəli HTML |
| Konfiqurasiya | `.env`, `python-dotenv` |
| Deploy | Replit (daxili PostgreSQL ilə) |

---

## 3. Layihənin qurulması

### 3.1. Qovluq strukturu

```
bakery/
├── backend/
│   ├── app.py            # Əsas Flask tətbiqi
│   ├── requirements.txt
│   └── setup.sql         # PostgreSQL cədvəlləri
├── frontend/             # HTML, CSS, JS
├── docs/
│   └── report-screenshots/
├── .env.example
└── REPORT.md
```

### 3.2. Mühit dəyişənləri (`.env`)

```env
FLASK_SECRET_KEY=uzun-tesadufi-acar
DATABASE_URL=postgresql://istifadeci:parol@localhost:5432/bakery_db
WEBSEC_LAB=1
DEV_INSECURE_SQL=1
DEV_AUTH_BYPASS=1
```

### 3.3. Verilənlər bazası

`setup.sql` faylı aşağıdakı cədvəlləri yaradır:

```sql
CREATE TABLE users (
    id            SERIAL PRIMARY KEY,
    email         VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name     VARCHAR(255),
    role          VARCHAR(50) DEFAULT 'user'
);

CREATE TABLE products (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    description TEXT,
    price       NUMERIC(10,2) NOT NULL,
    category    VARCHAR(100)
);

CREATE TABLE orders (
    id           SERIAL PRIMARY KEY,
    user_id      INTEGER REFERENCES users(id),
    product_name VARCHAR(255) NOT NULL,
    quantity     INTEGER DEFAULT 1,
    total        NUMERIC(10,2),
    status       VARCHAR(50) DEFAULT 'pending'
);
```

### 3.4. Replit-də deploy

Layihə Replit platformasında yerləşdirilib. Replit Agent avtomatik olaraq:
- Flask, psycopg2, werkzeug kitabxanalarını qurdu
- PostgreSQL verilənlər bazasını yaratdı
- `setup.sql` faylını icra edərək cədvəlləri yaratdı
- Flask serverini işə saldı

---

## 4. Backend: əsas komponentlər

### 4.1. Flask tətbiqi

```python
# backend/app.py (fragment)
app = Flask(__name__)
app.secret_key = os.environ.get("FLASK_SECRET_KEY") or os.urandom(24)
```

### 4.2. Lab rejimi bayraqları

`WEBSEC_LAB`, `DEV_INSECURE_SQL`, `DEV_AUTH_BYPASS` kimi dəyişənlər zəif SQL və parol yoxlamasını idarə edir:

```python
# backend/app.py — lab bayraqları
WEBSEC_LAB       = True   # Lab rejimi aktiv (default)
DEV_INSECURE_SQL = True   # SQLi zəifliyi aktiv
DEV_AUTH_BYPASS  = True   # Parol yoxlaması keçilir
```

### 4.3. PostgreSQL qoşulması

```python
# backend/app.py — get_db()
def get_db():
    import psycopg2
    from psycopg2.extras import RealDictCursor
    return psycopg2.connect(DATABASE_URL, cursor_factory=RealDictCursor)
```

### 4.4. Statik faylların verilməsi

Path traversal əleyhinə yoxlama ilə:

```python
# backend/app.py — serve_file()
@app.route("/<path:path>")
def serve_file(path):
    safe = (FRONTEND_DIR / path).resolve()
    try:
        safe.relative_to(FRONTEND_DIR.resolve())
    except ValueError:
        return "Forbidden", 403
    return send_from_directory(FRONTEND_DIR, path)
```

---

## 5. Frontend

* **Giriş/Qeydiyyat**: `fetch("/api/login")`, `fetch("/api/register")` ilə API çağırışları
* **Menyu səhifəsi**: `/api/products`-dan məhsullar yüklənir; axtarış xanası SQLi giriş nöqtəsidir
* **Sifariş forması**: Daxil olmuş istifadəçi üçün sifariş yaradılır
* **SQLi nəticəsi**: Serverdən gələn xəta mesajı qırmızı qutuda `textContent` ilə ekranda göstərilir (XSS-dən qaçmaq üçün)

---

## 6. Nəzərdə tutulmuş təhlükəsizlik "boşluqları" (lab)

Aşağıdakılar **tədris məqsədilə** kodda mövcuddur və mühit dəyişənləri ilə söndürülə bilər.

---

### 6.1. SQL Injection — Authentication Bypass

**Səbəb:** İstifadəçi girişi birbaşa SQL mətninə birləşdirilir.

```python
# backend/app.py — _user_lookup_concat (SQLi lab)
def _user_lookup_concat(email: str):
    sql = (
        "SELECT * FROM users WHERE email = '"
        + email    # ← TƏHLÜKƏLİ: birbaşa birləşdirmə!
        + "'"
    )
    cur.execute(sql)
```

**Hücum payloadu** (email sahəsinə yazılır):
```
' OR '1'='1
```

**İcra olunan sorğu:**
```sql
SELECT * FROM users WHERE email = '' OR '1'='1'
```

**Nəticə:** Şərt həmişə `TRUE` qaytarır → sistem istifadəçini **parols uz içəri buraxır**.

![SQLi Auth Bypass](docs/report-screenshots/01-sqli-auth-bypass.png)  
*Şəkil 1. Login səhifəsi — `' OR '1'='1` yazılıb, sistem parols uz Admin User kimi daxil etdi*

**Təhlükəsiz alternativ:**
```python
cur.execute(
    "SELECT * FROM users WHERE email = %s",
    (email,)   # ← parametrləşdirilmiş sorğu
)
```

---

### 6.2. SQL Injection — Verilənlər bazası xətasının sızması

**Səbəb:** Yanlış formatlı payload PostgreSQL sintaksis xətası yaradır və bu xəta ekranda göstərilir.

**Hücum payloadu** (menyu axtarış xanasına yazılır):
```
%' UNION SELECT 1,email,password_hash,4,5,6 FROM users--
```

**Ekranda görünən xəta:**
```
DATABASE ERROR (SQLI LAB)
each UNION query must have the same number of columns
LINE 1: ... FROM products WHERE name ILIKE '%%' UNION SELECT 1,email,pa...
SQL FRAGMENT: %' UNION SELECT 1,email,password_hash,4,5,6 FROM users--
```

**Nəticə:** Hücumçu verilənlər bazası strukturu haqqında məlumat alır.

![SQLi DB Error](docs/report-screenshots/02-sqli-db-error.png)  
*Şəkil 2. Menyu axtarışı — yanlış UNION payload ilə DB xəta mesajı ekranda görünür*

---

### 6.3. SQL Injection — UNION Attack (məlumat oğurluğu)

**Səbəb:** Düzgün sütun sayı ilə UNION sorğusu verilənlər bazasından istifadəçi məlumatlarını çıxarır.

**Hücum payloadu:**
```
%' UNION SELECT id,email,password_hash,0,role FROM users--
```

**Nəticə:** Admin istifadəçisinin email və parol hash-ı menyu kartı kimi ekranda görünür:

```
Ad:         admin@vivaladiva.com
Təsvir:     pbkdf2:sha256:260000$example$hashedpassword
Kateqoriya: ADMIN
```

![SQLi UNION Attack](docs/report-screenshots/03-sqli-union-attack.png)  
*Şəkil 3. UNION attack uğurlu — admin email və parol hash-ı menyu kartı kimi görünür*

**Təhlükəsiz alternativ:**
```python
# Parametrləşdirilmiş sorğu ilə UNION attack mümkün deyil:
cur.execute(
    "SELECT * FROM products WHERE name ILIKE %s",
    (f"%{search}%",)
)
```

---

### 6.4. IDOR — Başqasının sifarişinə icazəsiz giriş

**Səbəb:** IDOR (Insecure Direct Object Reference) — API sorğusunda `user_id` yoxlanılmır.

```python
# backend/app.py — ZƏİF versiya
@app.route("/api/orders/<int:oid>", methods=["GET"])
def api_order_by_id(oid):
    # user_id yoxlanılmır — hər kəs istənilən sifarişi görə bilər!
    cur.execute("SELECT * FROM orders WHERE id = %s", (oid,))
```

**Hücum ssenarisi:**
1. `test@test.com` ilə daxil olunur
2. Brauzerdə URL yazılır: `/api/orders/1`
3. Nəticə: `admin@vivaladiva.com`-un sifarişi görünür!

```json
{
  "ok": true,
  "order": {
    "id": 1,
    "user_id": 1,
    "product_name": "Chocolate Lava Cake",
    "total": "15.00"
  }
}
```

![IDOR](docs/report-screenshots/04-idor.png)  
*Şəkil 4. IDOR — `test@test.com` ilə daxil olub `/api/orders/1` açıldı, admin sifarişi görünür*

**Təhlükəsiz alternativ:**
```python
# WHERE şərtinə user_id əlavə olunur:
cur.execute(
    "SELECT * FROM orders WHERE id = %s AND user_id = %s",
    (oid, session["user_id"])
)
```

---

### 6.5. Zəif admin idarəetməsi

**Səbəb:** Admin yoxlaması yalnız sessiyadakı email-ə əsaslanır — real RBAC yoxdur.

```python
# backend/app.py — api_admin_orders
if session.get("email") != "admin@vivaladiva.com":
    return jsonify({"ok": False, "error": "Admin hüququnuz yoxdur."}), 403
```

**Problem:** Əgər hücumçu SQLi ilə admin emaili ilə sessiya yarada bilsəydi, admin paneli açılardı. Real sistemdə verilənlər bazasında rol cədvəli olmalıdır.

**Təhlükəsiz alternativ:**
```python
cur.execute("SELECT role FROM users WHERE id = %s", (session["user_id"],))
row = cur.fetchone()
if row["role"] != "admin":
    return jsonify({"ok": False, "error": "İcazəniz yoxdur."}), 403
```

---

### 6.6. Lab üçün parol yoxlamasının keçilməsi

`DEV_AUTH_BYPASS` aktiv olanda parol hash yoxlanılmır:

```python
# backend/app.py — api_login
if DEV_INSECURE_SQL and DEV_AUTH_BYPASS:
    pw_ok = True   # ← LAB: parol yoxlanılmır!
else:
    pw_ok = check_password_hash(row["password_hash"], password)
```

Bu, yalnız lab mühitində SQLi payload ilə sınaq üçün məntiqlidir; istehsalda **söndürülməlidir**.

---

## 7. Təhlükəsizlik yamaları və yaxşı təcrübələr

| Boşluq | Zəif versiya | Təhlükəsiz versiya |
| --- | --- | --- |
| SQL Injection | String birləşdirməsi | Parametrləşdirilmiş sorğu (`%s`) |
| Auth Bypass | Parol yoxlanılmır | `werkzeug.check_password_hash()` |
| IDOR | Yalnız ID yoxlanılır | ID + `user_id` birlikdə yoxlanılır |
| Admin Bypass | Email ilə yoxlama | RBAC + verilənlər bazası rol cədvəli |
| Path Traversal | Yoxlama yox | `relative_to()` ilə yoxlama |
| XSS | `innerHTML` istifadəsi | `textContent` ilə göstərmə |

* **Parametrləşdirilmiş sorğular** — `user_get_by_email`, `user_insert` və s. lab söndürüldükdə istifadə olunur
* **Parol hash** — `werkzeug.security`
* **Sessiya** — `Secure`, `HttpOnly`, `SameSite` çərəz parametrləri
* **Statik fayl path traversal** — `relative_to` yoxlaması
* **Frontend** — SQLi demo üçün məlumat `textContent` ilə verilir (XSS qarşısının alınması)

---

## 8. Deploy (Replit)

Layihə Replit platformasında yerləşdirilib. Replit-in daxili PostgreSQL-dən istifadə edilir.

* Replit Agent layihənin zip faylını import etdi
* `requirements.txt`-dən asılılıqları avtomatik qurdu
* `setup.sql` ilə verilənlər bazasını yaratdı
* Flask serverini port 5000-də işə saldı

**Canlı URL:** https://1ae2911b-ea60-4936-a316-39ae152e59ce-00-2v7mhych7njmt.sisko.replit.dev/

**`/api/health` endpoint cavabı:**
```json
{
  "ok": true,
  "app": "Viva La Diva Bakery",
  "lab_mode": true,
  "insecure_sql": true,
  "auth_bypass": true
}
```

![Deploy](docs/report-screenshots/05-deploy.png)  
*Şəkil 5. Replit — layihə işləyir, /api/health endpoint cavabı*

---

## 9. Demo şəkilləri

| № | Təsvir | Fayl |
| --- | --- | --- |
| 1 | SQLi Auth Bypass — parols uz giriş | `docs/report-screenshots/01-sqli-auth-bypass.png` |
| 2 | SQLi — DB xəta mesajı ekranda | `docs/report-screenshots/02-sqli-db-error.png` |
| 3 | SQLi UNION Attack — admin email + hash | `docs/report-screenshots/03-sqli-union-attack.png` |
| 4 | IDOR — başqasının sifarişi | `docs/report-screenshots/04-idor.png` |
| 5 | Replit deploy — /api/health | `docs/report-screenshots/05-deploy.png` |
| 6 | Əsas səhifə | `docs/report-screenshots/06-home.png` |
| 7 | Menyu səhifəsi | `docs/report-screenshots/07-menu.png` |

---

## 10. Nəticə və tövsiyələr

Layihə Flask + PostgreSQL üzərində tamstack veb tətbiq və **tədris məqsədli** zəiflik nümunələrini birləşdirir. Real mühitdə:

* `DEV_INSECURE_SQL`, `DEV_AUTH_BYPASS`, `WEBSEC_LAB` **söndürülməli** və ya silinməlidir;
* bütün SQL sorğuları **parametrləşdirilməli**;
* admin paneli **RBAC**, audit log ilə möhkəmləndirilməlidir;
* sızmaların **ekranda göstərilməsi** aradan qaldırılmalıdır;
* IDOR üçün hər sorğuda **`user_id` yoxlanılmalıdır**.

---

## 11. İstinadlar

* Flask: https://flask.palletsprojects.com/
* PostgreSQL: https://www.postgresql.org/docs/
* OWASP Top 10: https://owasp.org/www-project-top-ten/
* OWASP SQL Injection: https://owasp.org/www-community/attacks/SQL_Injection
* OWASP IDOR: https://owasp.org/www-chapter-ghana/assets/slides/IDOR.pdf
* Replit: https://replit.com/

---

*Hesabatın hazırlanma tarixi: 2026. Bu sənəd tədris məqsədilədir; kod nümunələri layihə fayllarından götürülüb.*
