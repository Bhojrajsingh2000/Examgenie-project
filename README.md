# ExamGenie
**Automated Question Paper Generation System for Schools & Colleges**
Built with Python, Django and MySQL.

This is the full working implementation matching the project report: role-based
access (Admin / Teacher / Exam Coordinator), a question bank with bulk import,
a blueprint-driven paper generation engine with PDF export, and the complete
**Teacher-to-Admin Paper Processing workflow** (submission → admin transcription
→ notification → pay-to-download via a Payment Gateway).

---

## 1. Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.10+, Django 4.2/5.x |
| Database | MySQL 8.x (via `mysqlclient`) |
| Frontend | Django Templates, Bootstrap 5 |
| PDF Export | WeasyPrint |
| Bulk Import | openpyxl / pandas |
| Payment Gateway | Razorpay (swap in `payments/gateway.py` for Stripe/PayU) |


## 3. Setup Instructions

### 3.1 Prerequisites
- Python 3.10+
- MySQL 8.x server running locally or remotely
- `pip`, `virtualenv`

### 3.2 System packages (Ubuntu/Debian) needed to build `mysqlclient` and `WeasyPrint`
```bash
sudo apt-get install default-libmysqlclient-dev pkg-config \
     libpango-1.0-0 libpangocairo-1.0-0 libcairo2 build-essential python3-dev
```

### 3.3 Create a virtual environment & install dependencies
```bash
python3 -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 3.4 Configure environment variables
```bash
cp .env.example .env
# then edit .env with your MySQL credentials and (optionally) Razorpay test keys
```

### 3.5 Create the MySQL database
```sql
CREATE DATABASE examgenie_db CHARACTER SET utf8mb4;
CREATE USER 'examgenie_user'@'localhost' IDENTIFIED BY 'your-password';
GRANT ALL PRIVILEGES ON examgenie_db.* TO 'examgenie_user'@'localhost';
```

### 3.6 Run migrations & create an admin account
```bash
python manage.py migrate
python manage.py createsuperuser        # or set role=ADMIN via /admin/ afterwards
```

### 3.7 Run the development server
```bash
python manage.py runserver
```
Visit `http://127.0.0.1:8000/`.

> Note: `RAZORPAY_KEY_ID` / `RAZORPAY_KEY_SECRET` can be left blank during local
> development — `payments/gateway.py` automatically falls back to a dev mode
> that simulates a successful order + signature so the full pay-to-download
> flow can be tested without real payment gateway credentials.

## 4. Running the Automated Tests
The project ships with 16 automated tests covering the paper generation
engine, PDF export, the Teacher-to-Admin workflow and the payment flow.

```bash
python manage.py test tests.test_question_bank tests.test_papergen tests.test_submissions tests.test_payments
```

## 5. Key Workflows

### 5.1 Automated Paper Generation (Coordinator)
1. Admin sets up Classes → Subjects → Chapters.
2. Teachers add Questions to the bank (or bulk-import via CSV/XLSX).
3. Coordinator defines a **Blueprint** (marks/difficulty/type distribution) under `/blueprints/`.
4. Coordinator clicks **Generate** under `/papers/generate/`, choosing how many sets (A/B/C...).
5. The engine (`papergen/engine.py`) auto-selects questions, avoiding recently-used ones,
   and `papergen/pdf_export.py` renders a print-ready question paper PDF + a separate answer key PDF.

### 5.2 Teacher-to-Admin Paper Processing (report section 6.1)
1. **Teacher Login & Submission** — Teacher uploads a handwritten paper PDF at `/submissions/submit/`.
2. **Admin Processing** — Admin is notified, reviews the PDF at `/submissions/admin/pending/`,
   transcribes it, and uploads the finalized digital PDF.
3. **Notification** — Teacher is notified the paper is ready.
4. **Pay-to-Download** — Teacher pays via the integrated Payment Gateway at `/payments/initiate/<id>/`;
   on success, a secure, time-limited download link is issued (`payments/views.py: secure_download`).

## 6. Default User Roles
Created via Django Admin (`/admin/`) or the in-app "Add User" form (Admin only):
- **ADMIN** — manages classes/subjects, processes submissions, views all reports.
- **TEACHER** — manages own question bank, submits handwritten papers, pays to download.
- **COORDINATOR** — defines blueprints, generates papers, views generation reports.

## 7. Production Notes
- Set `DEBUG=False` and a real `SECRET_KEY` in `.env`.
- Serve static/media via Nginx; run the app with `gunicorn examgenie.wsgi:application`.
- Configure the Razorpay **webhook** URL (`/payments/webhook/`) in the gateway dashboard and
  verify the `X-Razorpay-Signature` header using `RAZORPAY_KEY_SECRET` before trusting the payload
  (a placeholder for this is left in `payments/views.py: payment_webhook`).
- Consider moving `media/` to S3/cloud storage for multi-server deployments.
