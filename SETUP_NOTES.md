# Setup Notes (verified on this machine)

Project imported from `C:\Users\Admin\Documents\student-management-using-django-main.zip`
into `C:\Users\Admin\Documents\Workspace dsh hardness\student-management-using-django-main`.

## Key finding: `requirements.txt` does NOT work on this machine

`requirements.txt` pins `Django==3.1.1` (Oct 2020). This machine only has
**Python 3.13.15**, and Python 3.12+ **removed the `distutils` module**.

Django 3.1.1 imports `distutils.version.LooseVersion`, so it fails immediately:

```
File ".../django/utils/version.py", line 6, in <module>
    from distutils.version import LooseVersion
ModuleNotFoundError: No module named 'distutils'
ImportError: Couldn't import Django.
```

There is **no Python 3.11 or older installed** (`py -0p` lists only 3.13), so the
original pinned stack cannot be run as-is.

## Working configuration (verified end-to-end)

Run with a modern Django instead. This was tested and works:

```powershell
cd "C:\Users\Admin\Documents\Workspace dsh hardness\student-management-using-django-main"
python -m venv venv
.\venv\Scripts\python.exe -m pip install django dj-database-url whitenoise requests pillow
$env:MY_SECRET_KEY="any-dev-value-here"
.\venv\Scripts\python.exe manage.py migrate
.\venv\Scripts\python.exe manage.py runserver
```

Installed and verified versions:

| Package | Version |
|---|---|
| Django | 6.1.2 |
| dj-database-url | 3.1.2 |
| whitenoise | 6.12.0 |
| requests | 2.34.2 |
| Pillow | 12.3.0 |

No source code changes were required for this to work.

### Verified results

- `manage.py check` -> **System check identified no issues (0 silenced).**
- `manage.py migrate` -> all migrations apply cleanly.
- `manage.py collectstatic` -> 3667 static files copied.
- Server boots; `GET /` (login page) returns **200**.
- All **35 pages** across all three roles return 200 with no exceptions
  (HOD/admin 19 pages, Staff 9 pages, Student 7 pages).

## Issues found

### Fixed

1. **`SECRET_KEY` was `None` by default.**
   `settings.py` read `os.environ.get('MY_SECRET_KEY')` with no fallback, so the
   app crashed on startup unless `MY_SECRET_KEY` was set. The README does not
   mention this. Now falls back to a dev-only key; production should still set
   `MY_SECRET_KEY` in the environment.

2. **Broken Ionicons stylesheet on 2 pages.**
   `{% static %}` was wrapped around an absolute external URL, producing a
   mangled path instead of loading the CDN:
   - `main_app/templates/main_app/login.html` line 14
   - `main_app/templates/registration/base.html` line 15

   ```html
   <!-- before (broken): /static/https%3A/code.ionicframework.com/... -->
   <link rel="stylesheet" href="{% static 'https://code.ionicframework.com/ionicons/2.0.1/css/ionicons.min.css' %}">
   ```

   `main_app/templates/main_app/base.html` line 14 was already correct, and
   `index.html` line 14 likewise. Ionicons is not vendored locally, so the fix
   was to drop the `{% static %}` wrapper.

3. **`STATICFILES_STORAGE` was obsolete.**
   Removed in Django 5.1 and silently ignored, so WhiteNoise compression was not
   actually active. Added the modern `STORAGES` setting; both are declared so the
   project works on old and new Django.

### Still open (low priority)

4. **`db.sqlite3` and `.env`-style secrets are gitignored**; `dj_database_url`
   prints `WARNING: No DATABASE_URL environment variable set` on every command.
   Harmless, but noisy.

5. **`ALLOWED_HOSTS = ['*']` with `DEBUG = True`**, plus default credentials
   (`admin@admin.com` / `admin`) documented in the README and hardcoded reCAPTCHA
   and Firebase keys in `main_app/views.py`. Fine for local study; do not expose
   this publicly as-is.

5. **Hardcoded third-party keys in source** (Google reCAPTCHA site key in
   `views.py` line 33, Firebase config in `views.py`). Also note the reCAPTCHA
   check calls Google's API on every login, so login needs working internet.

## Test credentials seeded locally

Created in the local `db.sqlite3` for testing:
`admin@admin.com` / `admin`, `staff@staff.com` / `staff`,
`student@student.com` / `student` (plus one Course, Session and Subject).

`db.sqlite3` is gitignored, so this is local-only.