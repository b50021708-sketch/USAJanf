import os
import re
import secrets
from datetime import datetime, timezone

import psycopg
from psycopg.rows import dict_row
from flask import Flask, request, jsonify, render_template_string, session, redirect, url_for, flash
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

app = Flask(__name__)
app.config["SECRET_KEY"] = os.environ.get("SECRET_KEY")
if not app.config["SECRET_KEY"]:
    raise RuntimeError("Set a strong SECRET_KEY environment variable.")

DATABASE_URL = os.environ.get("DATABASE_URL")
if not DATABASE_URL:
    raise RuntimeError("Set DATABASE_URL to your PostgreSQL connection URL.")

# Do not trust forwarded client IP headers unless your deployment proxy is configured.
limiter = Limiter(
    get_remote_address,
    app=app,
    default_limits=["120 per hour"],
    storage_uri="memory://",
)

def db_connect():
    return psycopg.connect(DATABASE_URL, row_factory=dict_row, connect_timeout=10)

def ensure_db():
    with db_connect() as db:
        db.execute("""
            CREATE TABLE IF NOT EXISTS signups (
                id BIGSERIAL PRIMARY KEY,
                name VARCHAR(80) NOT NULL,
                email VARCHAR(254) NOT NULL UNIQUE,
                created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
            )
        """)

PAGE = r"""
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="theme-color" content="#101c36">
<title>United Together | Community Pledge</title>
<style>
:root{--navy:#101c36;--blue:#244c91;--red:#c43b4b;--muted:#65738b}
*{box-sizing:border-box}
body{margin:0;background:#f3f5f9;color:var(--navy);font-family:Arial,Helvetica,sans-serif}
header{background:var(--navy);color:#fff;padding:20px 7%;display:flex;justify-content:space-between;align-items:center;gap:16px}
.brand{font-size:21px;font-weight:800;letter-spacing:.5px}.brand span{color:#f26b78}
.tag{font-size:11px;border:1px solid #52617a;padding:7px 10px;border-radius:20px}
.stripe{height:5px;background:linear-gradient(90deg,#c43b4b 0 33.3%,white 33.3% 66.6%,#244c91 66.6%)}
.hero{background:linear-gradient(115deg,#101c36,#193e77 75%,#244c91);color:white;text-align:center;padding:76px 20px 65px}
.eyebrow{color:#ffb6be;font-weight:bold;letter-spacing:3px;font-size:12px}
h1{font-size:clamp(36px,6vw,65px);margin:18px 0;letter-spacing:-2px}
.hero p{max-width:620px;margin:0 auto;color:#d9e2f2;line-height:1.8}
.counter{background:white;max-width:720px;width:calc(100% - 36px);margin:-28px auto 35px;position:relative;border-radius:15px;box-shadow:0 12px 35px #15264718;padding:25px;text-align:center}
.counter-label{color:#67758d;font-size:12px;font-weight:bold;letter-spacing:2px}
#count{font-size:clamp(36px,7vw,54px);font-weight:800;color:var(--blue);margin:10px 0;font-variant-numeric:tabular-nums}
.live{color:#25865a;font-size:12px}.dot{display:inline-block;width:8px;height:8px;background:#31ad72;border-radius:50%;margin-right:5px}
main{max-width:1000px;margin:0 auto;padding:10px 20px 60px}
.columns{display:grid;grid-template-columns:1fr 1fr;gap:35px;align-items:start}
h2{font-size:28px}.description{color:var(--muted);line-height:1.8}
.point{display:flex;gap:12px;margin:20px 0;align-items:flex-start}
.icon{background:#e3eaf8;border-radius:10px;padding:11px}
.point p{margin:5px 0;color:var(--muted);line-height:1.6}
.card{background:white;border:1px solid #e7eaf0;border-radius:15px;padding:26px;box-shadow:0 10px 30px #1526470b}
label.field{display:block;font-size:13px;font-weight:bold;margin:17px 0 8px}
input[type=text],input[type=email]{width:100%;padding:13px;border:1px solid #d9dfeb;border-radius:8px;font-size:15px}
input:focus{outline:2px solid #7398df;border-color:transparent}
.consent{display:flex;align-items:flex-start;gap:10px;color:var(--muted);font-size:12px;line-height:1.6;margin:18px 0}
button{width:100%;background:var(--red);border:0;color:white;border-radius:8px;padding:15px;font-weight:bold;font-size:15px;cursor:pointer}
button:hover{background:#a92e3e}
.message{background:#edf7f0;color:#22633d;padding:12px;border-radius:8px;margin-bottom:14px;font-size:13px}
.error{background:#fff0f0;color:#a52d35}
footer{text-align:center;padding:25px 15px;color:#738097;background:#e9edf4;font-size:12px;line-height:1.8}
@media(max-width:700px){.columns{grid-template-columns:1fr}header{padding:17px 5%}.hero{padding-top:60px}.tag{font-size:9px}}
</style>
</head>
<body>
<header>
  <div class="brand">UNITED<span> TOGETHER</span></div>
  <div class="tag">INDEPENDENT DEMO PROJECT</div>
</header>
<div class="stripe"></div>
<section class="hero">
  <div class="eyebrow">ONE COMMUNITY · ONE FUTURE</div>
  <h1>United We Move Forward.</h1>
  <p>A community pledge celebrating participation, shared responsibility, and a better future. Every voice counts.</p>
</section>
<section class="counter">
  <div class="counter-label">COMMUNITY PARTICIPANTS</div>
  <div id="count" aria-live="polite">—</div>
  <div class="live"><span class="dot"></span>Live database count</div>
</section>
<main>
<div class="columns">
  <section>
    <h2>Be part of the community.</h2>
    <p class="description">Join this independent demonstration project. Registration is voluntary. The counter reflects registrations stored by this website.</p>
    <div class="point"><div class="icon">🤝</div><div><strong>Community</strong><p>Bring people together around shared goals.</p></div></div>
    <div class="point"><div class="icon">📊</div><div><strong>Live statistics</strong><p>Watch the total update automatically.</p></div></div>
    <div class="point"><div class="icon">🔐</div><div><strong>Voluntary registration</strong><p>Only the information you submit is stored.</p></div></div>
  </section>
  <section class="card">
    <h2 style="margin-top:0">Join Us</h2>
    <p class="description">Complete the form below.</p>
    {% for category, message in get_flashed_messages(with_categories=true) %}
      <div class="message {{ 'error' if category == 'error' else '' }}">{{ message }}</div>
    {% endfor %}
    <form action="/join" method="post">
      <input type="hidden" name="csrf_token" value="{{ csrf_token }}">
      <label class="field" for="name">Your name</label>
      <input id="name" name="name" type="text" maxlength="80" required autocomplete="name" placeholder="Enter your name">
      <label class="field" for="email">Email address</label>
      <input id="email" name="email" type="email" maxlength="254" required autocomplete="email" placeholder="you@example.com">
      <label class="consent"><input type="checkbox" name="consent" value="yes" required><span>I voluntarily agree to submit this information to this independent demonstration website.</span></label>
      <button type="submit">JOIN THE COMMUNITY</button>
    </form>
  </section>
</div>
</main>
<footer>Independent demonstration website. Not affiliated with the U.S. government or any official organization.<br>Do not enter sensitive or confidential information.</footer>
<script>
async function updateCounter(){
  try{
    const response=await fetch("/api/count",{cache:"no-store"});
    if(!response.ok)return;
    const data=await response.json();
    document.getElementById("count").textContent=Number(data.count).toLocaleString("en-US");
  }catch(e){console.log("Counter refresh failed.");}
}
updateCounter();
setInterval(updateCounter,5000);
</script>
</body>
</html>
"""

@app.get("/")
def home():
    if "csrf_token" not in session:
        session["csrf_token"] = secrets.token_urlsafe(32)
    return render_template_string(PAGE, csrf_token=session["csrf_token"])

@app.get("/api/count")
@limiter.limit("60 per minute")
def count():
    ensure_db()
    with db_connect() as db:
        row = db.execute("SELECT COUNT(*) AS total FROM signups").fetchone()
    response = jsonify(count=row["total"])
    response.headers["Cache-Control"] = "no-store"
    return response

@app.post("/join")
@limiter.limit("10 per hour")
def join():
    token = request.form.get("csrf_token", "")
    session_token = session.get("csrf_token", "")
    if not session_token or not secrets.compare_digest(token, session_token):
        flash("Session expired. Please try again.", "error")
        return redirect(url_for("home"))

    if request.form.get("consent") != "yes":
        flash("Please confirm your consent.", "error")
        return redirect(url_for("home"))

    name = request.form.get("name", "").strip()
    email = request.form.get("email", "").strip().lower()

    if not name or len(name) > 80:
        flash("Please enter a valid name.", "error")
        return redirect(url_for("home"))
    if len(email) > 254 or not re.fullmatch(r"[^@\s]+@[^@\s]+\.[^@\s]+", email):
        flash("Please enter a valid email address.", "error")
        return redirect(url_for("home"))

    try:
        ensure_db()
        with db_connect() as db:
            db.execute(
                "INSERT INTO signups (name, email, created_at) VALUES (%s, %s, %s)",
                (name, email, datetime.now(timezone.utc))
            )
        flash("Registration successful. Welcome!", "success")
    except psycopg.errors.UniqueViolation:
        flash("This email is already registered.", "error")
    except psycopg.Error:
        app.logger.exception("Database operation failed")
        flash("Registration is temporarily unavailable. Please try again later.", "error")

    return redirect(url_for("home"))

@app.get("/health")
def health():
    return {"status": "ok"}

if __name__ == "__main__":
    # Local development only; production uses Gunicorn.
    app.run(host="127.0.0.1", port=5000, debug=False)
