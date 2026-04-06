Viva La Diva Bakery — Web təhlükəsizliyi layihəsi
Onlayn çörəkçi mağazası ətrafında qurulmuş tamstack nümunə: statik frontend, Flask REST API və PostgreSQL verilənlər bazası.
İmkanlar
İstifadəçi qeydiyyatı və girişi (sessiya əsaslı)
Menyu səhifəsi — məhsullar verilənlər bazasından yüklənir
Sifariş sistemi — daxil olmuş istifadəçi üçün
Admin paneli — bütün sifarişləri görmək üçün
Təhlükəsizlik lab rejimi — SQLi, IDOR nümayişi
Texnologiyalar
Tərəf
Texnologiya
Backend
Python 3, Flask
Verilənlər bazası
PostgreSQL (psycopg2)
Frontend
HTML, CSS, JavaScript
Deploy
Replit

Tələblər
Python 3.10+
PostgreSQL server
Quraşdırma (lokal)
Repozitoriyanı klonlayın:

git clone https://github.com/SIZIN-USERNAME/websec-bakery.git

cd websec-bakery

Kitabxanaları qurun:

pip install -r backend/requirements.txt

Layihə kökündə .env yaradın (.env.example-a baxın):

FLASK_SECRET_KEY=uzun-tesadufi-acar

DATABASE_URL=postgresql://istifadeci:parol@localhost:5432/bakery_db

WEBSEC_LAB=1

DEV_INSECURE_SQL=1

DEV_AUTH_BYPASS=1

Verilənlər bazasını qurun:

psql -U postgres -d bakery_db -f backend/setup.sql
İşə salma
python backend/app.py

Brauzerdə: http://127.0.0.1:5000/
Sağlamlıq yoxlaması: GET /api/health
Layihə strukturu
bakery/

├── backend/

│   ├── app.py              # Flask tətbiqi və API

│   ├── requirements.txt

│   └── setup.sql           # PostgreSQL cədvəlləri

├── frontend/               # HTML, CSS, JS

├── docs/

│   └── report-screenshots/ # Demo şəkilləri

├── .env.example

├── README.md

└── REPORT.md
Replit-də deploy
Layihə Replit platformasında yerləşdirilib:

Canlı URL: https://1ae2911b-ea60-4936-a316-39ae152e59ce-00-2v7mhych7njmt.sisko.replit.dev/

Replit Agent avtomatik olaraq:

Bütün kitabxanaları qurdu
PostgreSQL verilənlər bazasını yaratdı
setup.sql ilə cədvəlləri yaratdı
Flask serverini işə saldı
API endpoint-ləri
Method
URL
Təsvir
POST
/api/register
Qeydiyyat
POST
/api/login
Giriş (SQLi lab)
POST
/api/logout
Çıxış
GET
/api/me
Cari istifadəçi
GET
/api/products
Məhsullar (SQLi lab)
POST
/api/orders
Sifariş yarat
GET
/api/orders
Sifarişlərim
GET
/api/orders/<id>
Tək sifariş (IDOR lab)
GET
/api/admin/orders
Admin paneli
GET
/api/health
Sağlamlıq yoxlaması

Təhlükəsizlik qeydi
Bu kod bazası tədris məqsədilədir. Lab rejimindəki zəifliklər (DEV_INSECURE_SQL, DEV_AUTH_BYPASS) istehsal mühitində söndürülməlidir.
Universitet hesabatı
Ətraflı texniki hesabat (arxitektura, kod nümunələri, lab təsviri): REPORT.md
Demo şəkilləri: docs/report-screenshots/

