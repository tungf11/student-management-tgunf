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

## Known issues found (not yet fixed)

1. **`SECRET_KEY` is `None` by default.**
   `settings.py` reads `os.environ.get('MY_SECRET_KEY')` with no fallback, so the
   app crashes unless you set `MY_SECRET_KEY`. The README does not mention this.

2. **Broken Ionicons stylesheet on 3 pages.**
   `{% static %}` is wrapped around an absolute external URL, which produces a
   mangled path instead of loading the CDN:
   - `main_app/templates/main_app/login.html` line 14
   - `main_app/templates/main_app/base.html` line 14
   - `main_app/templates/registration/base.html` line 15

   ```html
   <!-- broken -->
   <link rel="stylesheet" href="{% static 'https://code.ionicframework.com/ionicons/2.0.1/css/ionicons.min.css' %}">
   ```
   `index.html` line 14 already does it correctly (plain URL). Ionicons is not
   vendored locally, so the fix is to drop the `{% static %}` wrapper.

3. **`STATICFILES_STORAGE` is obsolete.**
   Set in `settings.py` line 166; removed in Django 5.1. It is silently ignored,
   so WhiteNoise compression/manifest is not actually active.

4. **`db.sqlite3` and `.env`-style secrets are gitignored**; `dj_database_url`
   prints `WARNING: No DATABASE_URL environment variable set` on every command.
   Harmless, but noisy.

5. **Hardcoded third-party keys in source** (Google reCAPTCHA site key in
   `views.py` line 33, Firebase config in `views.py`). Also note the reCAPTCHA
   check calls Google's API on every login, so login needs working internet.

## Test credentials seeded locally

Created in the local `db.sqlite3` for testing:
`admin@admin.com` / `admin`, `staff@staff.com` / `staff`,
`student@student.com` / `student` (plus one Course, Session and Subject).

`db.sqlite3` is gitignored, so this is local-only.