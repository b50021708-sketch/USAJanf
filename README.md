# United Together — independent demo website

A Flask website with an English landing page, voluntary signup form, live PostgreSQL-backed counter, duplicate-email protection, CSRF token, and basic rate limits.

## Run locally

Python 3.10+ recommended.

```bash
python -m venv .venv
# macOS/Linux/Termux:
source .venv/bin/activate
# Windows PowerShell:
# .venv\Scripts\Activate.ps1

pip install -r requirements.txt
```

Set environment variables before starting the app:

```bash
# macOS/Linux/Termux: use your actual PostgreSQL connection URL
export DATABASE_URL='postgresql://USER:PASSWORD@HOST:5432/DATABASE'
export SECRET_KEY="$(python -c 'import secrets; print(secrets.token_hex(32))')"
python app.py
```

Open http://127.0.0.1:5000.

## Deploy publicly on Render

1. Create a PostgreSQL database in Render.
2. Create a Web Service from this project in a GitHub repository.
3. Use `pip install -r requirements.txt` as the build command.
4. Use `gunicorn app:app --bind 0.0.0.0:$PORT` as the start command.
5. Set `DATABASE_URL` to the PostgreSQL **internal** connection URL for the database and set `SECRET_KEY` to a long random secret. `render.yaml` can configure most service settings.
6. Deploy. Render provides the public HTTPS URL.

## Notes

- This is an independent demonstration project, not an official U.S. government website.
- The counter shows actual rows registered in this project's database; it does not represent a real-world population or official participation count.
- Collect only information you genuinely need, publish a privacy notice before public launch, and review applicable privacy requirements.
- In-memory rate limiting is per process and resets on restart; use a shared Redis-backed limiter for multi-instance production deployments.
- Do not use Flask's development server for public production hosting.
