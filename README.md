# LIMS — Logbook Internship Management System

Sistem pengurusan logbook latihan industri berasaskan Laravel untuk pelajar dan penyelia.

## System Overview

| Actor          | Responsibilities                                                                                                |
| -------------- | --------------------------------------------------------------------------------------------------------------- |
| **Student**    | Submit daily log entries with attachments, generate AI summaries, track weekly progress, set personal reminders |
| **Supervisor** | Review/approve/reject log entries, assign tasks to students, view analytics & performance dashboard             |
| **Admin**      | Placeholder (Coming Soon)                                                                                       |

**Live URL:** https://lim-system.my

---

## Tech Stack

| Layer        | Technology                                                                                          |
| ------------ | --------------------------------------------------------------------------------------------------- |
| **Backend**  | PHP 8.2+, Laravel 12, Eloquent ORM, Composer                                                        |
| **Database** | MariaDB 10.11 / MySQL 8.0 (Production: Azure Docker / Development: XAMPP)                          |
| **Frontend** | Blade, Bootstrap 5.3, Tailwind CSS 4, jQuery 3.7, DataTables 1.13, SweetAlert2 11, Font Awesome 6.4 |
| **Build**    | Node.js 18+, Vite 7, Laravel Vite Plugin 2.0, Axios, Concurrently                                   |
| **Cloud**    | Cloudinary (file storage), Gemini 2.5 Flash-Lite (AI summaries)                                     |
| **Email**    | Mailtrap SDK (local dev), Brevo SMTP (production)                                                   |
| **Server**   | Azure Virtual Machine (Ubuntu 24.04 LTS, Docker Compose: Nginx, PHP 8.3, MariaDB 10.11, phpMyAdmin)  |

---

## Installation

```bash
# 1. Clone & install
git clone https://github.com/IAreShut/LIMS.git
cd LIMS
composer install
npm install

# 2. Setup environment
cp .env.example .env
php artisan key:generate

# 3. Database (XAMPP)
#    Start MySQL → Create database `lims_db`
#    Update .env: DB_DATABASE=lims_db, DB_USERNAME=root, DB_PASSWORD=

# 4. Migrate & seed
php artisan migrate
php artisan db:seed

# 5. Run development server
composer dev
# (Runs: artisan serve + queue:listen + pail + npm run dev)

# 6. Run queue and schedule
php artisan queue:work
php artisan schedule:work
```

**Test Accounts:**
| Role | Email | Password |
|---|---|---|
| Supervisor | supervisor@test.com | password123 |
| Student | student@test.com | password123 |
| Admin | admin@test.com | password123 |

---

## Architecture

```
CLIENT (Browser) ─── HTTPS ───▶ Laravel 12 Application ───▶ MySQL
                                   │
                              Controllers (per-role):
                                Auth, Student\{Dashboard,LogEntry,Progress,Profile,Notification},
                                Supervisor\{Dashboard,Review,Task,Analytics,Profile,AssignStudent}
                                   │
                              Eloquent Models:
                                User, Internship, LogEntry, LogAttachment,
                                Notification, Task, SupervisorAssignment
                                   │
                              Services:
                                Gemini API (AI summary), Cloudinary (files),
                                LimsDatabaseChannel + Mail (notifications)
```

**Architecture style:** Monolithic MVC, role-based route grouping, server-rendered Blade with AJAX interactivity. No API layer, no microservices.

---

## Module Summaries

### Authentication

- Login via email **or** matrix_id (auto-detect)
- Registration: strict pre-assignment check via `supervisor_assignments` table
- No email verification, no role middleware

### Log Entry (Core)

- Status lifecycle: `draft → pending → approved/rejected`
- Draft = editable, Pending = locked for supervisor review
- Attachments: Cloudinary (production) or local storage (development)
- AI summary: optional Gemini-generated professional summary from task description + images

### Review (Supervisor)

- Lists pending entries from assigned students only
- Approve (no comment) or Reject (mandatory feedback reason)

### Progress Tracking

- Weekly progress matrix grid (week 1–N)
- Status per week: Approved / Pending / Rejected / In Progress / Empty
- Drill-down into daily entries per week

### Task & Reminders

- Supervisor assigns tasks to **all** students (bulk)
- Student creates personal reminders
- Types: `sv_task` (supervisor) / `personal_reminder` (self)
- Notifications sent via database + email

### Notifications

- Custom `LimsDatabaseChannel` (not Laravel's default)
- Dual delivery: DB table + email
- Real-time polling (30s interval) via SweetAlert2 toasts
- Scheduled: Daily reminder every weekday at 5 PM

### Supervisor-Student Assignment

- Pre-assigned via `supervisor_assignments` lookup table (admin-managed)
- Registration auto-fills faculty/programme/class from assignment data
- AssignStudent page shows registered profiles + task metrics table
- At-risk detection: rejected logs or overdue tasks

---

## Design Patterns & Conventions

| Pattern                 | Implementation                                                                                                                               |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **Controller-per-role** | Separate namespaces for Student & Supervisor controllers                                                                                     |
| **Authorisation**       | Inline checks in controllers (no role middleware)                                                                                            |
| **Validation**          | Inline in controllers (no Form Request classes)                                                                                              |
| **File upload**         | Dual-path: `env('CLOUDINARY_URL')` → Cloudinary / local storage                                                                              |
| **Transactions**        | `DB::beginTransaction/commit/rollback` for multi-step writes                                                                                 |
| **Phone format**        | `+60XXXXXXXXX` (Malaysian normalisation)                                                                                                     |
| **CSS/JS**              | Page-specific files in `public/css/` & `public/js/` (not Vite-bundled); only `resources/css/app.css` & `resources/js/app.js` go through Vite |

---

## Deployment

### Azure Docker (Current Production)

* **Server IP:** `40.80.80.249`
* **SSH Login:** `ssh azureuser@40.80.80.249`

#### Essential Docker Commands (Run inside `/var/www/LIMS`):

```bash
# 🚀 Start / Build All Containers
sudo docker compose up -d --build

# 🛑 Stop All Containers
sudo docker compose down

# 📊 Check Status of Containers
sudo docker compose ps

# 📜 View Live Logs
sudo docker compose logs -f
```

#### Useful Laravel Commands inside Docker:

```bash
# 🧹 Clear Configuration & Cache
sudo docker exec -it lims-app php artisan config:clear
sudo docker exec -it lims-app php artisan cache:clear

# 🔑 Fix Storage Permissions
sudo docker exec -it lims-app chmod -R 777 storage bootstrap/cache

# 🔗 Storage Link
sudo docker exec -it lims-app php artisan storage:link

# 🗄️ Database Import / Export
sudo docker exec -i lims-db mysql -u root -plims-system lims_db < /tmp/lims_db.sql
sudo docker exec lims-db mysqldump -u root -plims-system lims_db > /tmp/backup.sql

# 🔍 View Laravel Log
sudo docker exec -it lims-app tail -f storage/logs/laravel.log
```

#### 🔄 How to Pull & Update Azure Server After New Git Commit:

```bash
# 1. SSH into Azure Server:
ssh azureuser@40.80.80.249

# 2. Pull latest code & restart app container:
cd /var/www/LIMS
git pull origin main
sudo docker compose restart app

# 3. If you added new Composer packages or DB migrations:
sudo docker exec -it lims-app composer install --no-dev --optimize-autoloader
sudo docker exec -it lims-app php artisan migrate --force
sudo docker exec -it lims-app php artisan config:clear
```


