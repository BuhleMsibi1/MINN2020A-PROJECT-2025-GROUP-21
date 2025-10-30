# MINN2020A-PROJECT-2025-GROUP-21
#IMPORTS 
from flask import Flask, render_template_string, request, redirect, url_for, session
from flask_sqlalchemy import SQLAlchemy
from werkzeug.security import generate_password_hash, check_password_hash
import folium
import plotly.express as px
import pandas as pd
from markupsafe import Markup
import os

#FLASK APP CONFIG
app = Flask(_name_)
app.secret_key = "mysecretkey2025"
app.config["SQLALCHEMY_DATABASE_URI"] = "sqlite:///app.db"
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False

db = SQLAlchemy(app)

#DATABASE MODELS 
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(50), unique=True, nullable=False)
    password = db.Column(db.String(200), nullable=False)
    role = db.Column(db.String(30), nullable=False)

class Mineral(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    country = db.Column(db.String(100), nullable=False)
    mineral = db.Column(db.String(50), nullable=False)
    quantity = db.Column(db.Integer, nullable=False)
    lat = db.Column(db.Float, nullable=False)
    lon = db.Column(db.Float, nullable=False)

class CountryProfile(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    country = db.Column(db.String(100), unique=True, nullable=False)
    gdp = db.Column(db.Float, nullable=True)
    key_projects = db.Column(db.Text, nullable=True)
    notes = db.Column(db.Text, nullable=True)

#INITIALIZE DATABASE
with app.app_context():
    db.create_all()
    if not User.query.first():
        default_users = [
            User(username="Tshepho", password=generate_password_hash("Admin123"), role="Administrator"),
            User(username="Sethabile", password=generate_password_hash("Invest123"), role="Investor"),
            User(username="Emihle", password=generate_password_hash("Research123"), role="Researcher"),
            User(username="Buhle", password=generate_password_hash("Developer123"), role="Developer"),
        ]
        db.session.add_all(default_users)
        db.session.commit()

    if not Mineral.query.first():
        sample_data = [
            {"country": "Nigeria", "mineral": "Lithium", "quantity": 5000, "lat": 9.0820, "lon": 8.6753},
            {"country": "DR Congo", "mineral": "Cobalt", "quantity": 12000, "lat": -4.0383, "lon": 21.7587},
            {"country": "South Africa", "mineral": "Platinum", "quantity": 8000, "lat": -30.5595, "lon": 22.9375},
        ]
        for rec in sample_data:
            db.session.add(Mineral(**rec))
        db.session.commit()

    if not CountryProfile.query.first():
        country_info = [
            CountryProfile(country="Nigeria", gdp=440.8, key_projects="Kaduna Lithium Project", notes="Expanding lithium extraction capacity."),
            CountryProfile(country="DR Congo", gdp=64.0, key_projects="Tenke Fungurume Mine", notes="Largest cobalt supplier globally."),
            CountryProfile(country="South Africa", gdp=420.0, key_projects="Mogalakwena PGM Mine", notes="Major platinum producer."),
        ]
        # ROUTES

@app.route("/view_as/<role>", methods=["POST"])
def view_as(role):
    if session.get("role") in ["Administrator", "Developer"]:
        session["original_role"] = session["role"]
        session["role"] = role
    return redirect(url_for("dashboard"))

@app.route("/return_admin", methods=["POST"])
def return_admin():
    if "original_role" in session:
        session["role"] = session.pop("original_role")
    return redirect(url_for("dashboard"))

@app.route("/", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        username = request.form["username"]
        password = request.form["password"]
        role = request.form["role"]
        user = User.query.filter_by(username=username).first()
        if user and check_password_hash(user.password, password) and user.role == role:
            session["username"] = user.username
            session["role"] = user.role
            return redirect(url_for("dashboard"))
        return render_template_string(login_page, error="Invalid login")
    return render_template_string(login_page)

@app.route("/logout")
def logout():
    session.clear()
    return redirect(url_for("login"))

@app.route("/dashboard")
def dashboard():
    if "username" not in session:
        return redirect(url_for("login"))

    countries = [c[0] for c in db.session.query(Mineral.country).distinct()]
    mineral_types = [m[0] for m in db.session.query(Mineral.mineral).distinct()]
    query = Mineral.query
    if request.args.get('country'):
        query = query.filter_by(country=request.args.get('country'))
    if request.args.get('mineral'):
        query = query.filter_by(mineral=request.args.get('mineral'))

    minerals = query.all()
    users = User.query.all()
    country_profiles = CountryProfile.query.all()

    return render_template_string(
        dashboard_page,
        session=session,
        minerals=minerals,
        users=users,
        countries=countries,
        mineral_types=mineral_types,
        country_profiles=country_profiles,
        request=request
    )

        db.session.add_all(country_info)
        db.session.commit()
        # HTML TEMPLATES

login_page = """
<!DOCTYPE html>
<html>
<head>
  <title>Login</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
  <style>
    body {
      background-color: #2f3e46;
      color: #ffffff;
      font-family: Arial, sans-serif;
    }
    .login-container {
      width: 100%;
      max-width: 400px;
      margin: 7% auto;
      background-color: #354f52;
      padding: 30px;
      border-radius: 12px;
      box-shadow: 0 4px 12px rgba(0,0,0,0.4);
    }
    input, select {
      background-color: #52796f;
      color: #ffffff;
      border: none;
    }
    input::placeholder {
      color: #dcdcdc;
    }
    .btn-success {
      background-color: #84a98c;
      border: none;
      color: #2f3e46;
      font-weight: bold;
    }
    .btn-success:hover {
      background-color: #95bfa2;
       }
  </style>
  </head>
<body>
  <div class="login-container">
    <h3 class="text-center mb-4 text-light">African Critical Minerals App</h3>
    <form method="POST">
      <input class="form-control mb-3" name="username" placeholder="Username" required>
      <input class="form-control mb-3" type="password" name="password" placeholder="Password" required>
      <select class="form-select mb-3" name="role" required>
        <option value="">-- Choose your role --</option>
        <option value="Administrator">Administrator</option>
        <option value="Investor">Investor</option>
        <option value="Researcher">Researcher</option>
        <option value="Developer">Developer</option>
      </select>
      <button class="btn btn-success w-100" type="submit">Login</button>
      {% if error %}
        <div class="alert alert-danger mt-3">{{ error }}</div>
      {% endif %}
    </form>
  </div>
</body>
</html>
"""
  
#DASHBOARD PAGE 

dashboard_page = """ 
<!DOCTYPE html>
<html>
<head>
  <title>Dashboard</title>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
  <style>
    /* GLOBAL OVER ALL THEME */
    body {
      background-color: #2f3e46; /* dark blue-gray */
      color: #ffffff;
      font-family: 'Segoe UI', Arial, sans-serif;
    }

    /*  NAVVIGATION BAR */
    .navbar {
      background-color: #1e2b2f !important;
      border-bottom: 3px solid #d4af37;
      box-shadow: 0 2px 5px rgba(0,0,0,0.2);
    }
    .navbar-brand {
      color: #d4af37 !important;
      font-weight: 700;
      letter-spacing: 1px;
    }
    .navbar .nav-link, .navbar .btn {
      color: #ffffff !important;
    }
    .navbar .dropdown-menu {
      background-color: #52734d;
      border: none;
    }
    .navbar .dropdown-item {
      color: #ffffff;
    }
    .navbar .dropdown-item:hover {
      background-color: #d4af37;
      color: #2f3e46;
    }

    /* CONTAINER */
    .container {
      background-color: #354f52;
      border-radius: 12px;
      padding: 30px;
      margin-top: 30px;
      box-shadow: 0 4px 12px rgba(0,0,0,0.3);
    }

    h3, h4 {
      color: #ffffff;
      font-weight: bold;
    }

    h4 {
      margin-top: 40px;
      border-left: 5px solid #d4af37;
      padding-left: 10px;
    }

    hr {
      border-top: 1px solid #d4af37;
    }

    /* TABLES APPEARANCE */
    table {
      background-color: #ffffff;
      color: #2f3e46;
      border-radius: 8px;
      overflow: hidden;
    }
    thead {
      background-color: #52734d;
      color: #ffffff;
    }
    tbody tr:nth-child(even) {
      background-color: #f2f2f2;
    }

    /* BUTTONS APPEARANCE */
    .btn-success {
      background-color: #52734d;
      border: none;
    }
    .btn-success:hover {
      background-color: #435e40;
    }

    .btn-primary {
      background-color: #ffffff;
      border: none;
      color: #2f3e46;
      font-weight: 600;
    }
    .btn-primary:hover {
      background-color: #b8952e;
    }

    .btn-danger {
      background-color: #b93e3e;
      border: none;
    }
    .btn-danger:hover {
      background-color: #922e2e;
    }

    .btn-warning {
      background-color: #d4af37;
      border: none;
      color: #2f3e46;
      font-weight: 600;
    }
    .btn-warning:hover {
      background-color: #b8952e;
    }

    /* ALERTS & IFRAMES */
    .alert-info {
      background-color: #e9f5ec;
      border-color: #52734d;
      color: #2f3e46;
    }

    iframe {
      border: 2px solid #52734d;
      border-radius: 8px;
    }

    /* RESPONSE TO SCREEN SIZE */
    @media (max-width: 768px) {
      .container { padding: 15px; }
      h3 { font-size: 1.3rem; }
      h4 { font-size: 1.1rem; }
    }
  </style>
</head>
<body>
  <nav class="navbar navbar-expand-lg navbar-dark">
    <div class="container-fluid">
      <a class="navbar-brand" href="#">Critical Minerals</a>
      <div class="collapse navbar-collapse" id="navbarNav">
        <ul class="navbar-nav ms-auto">
          {% if session['role'] in ["Administrator", "Developer"] %}
          <li class="nav-item dropdown">
            <a class="nav-link dropdown-toggle" href="#" data-bs-toggle="dropdown">View As</a>
            <ul class="dropdown-menu dropdown-menu-end">
              <li><form action="{{ url_for('view_as', role='Researcher') }}" method="POST"><button class="dropdown-item" type="submit">Researcher</button></form></li>
              <li><form action="{{ url_for('view_as', role='Investor') }}" method="POST"><button class="dropdown-item" type="submit">Investor</button></form></li>
            </ul>
          </li>
          {% endif %}
          <li class="nav-item"><form action="{{ url_for('logout') }}" method="GET"><button class="btn btn-danger btn-sm mt-2">Logout</button></form></li>
        </ul>
      </div>
    </div>
  </nav>

  <div class="container mt-4">
    {% if session.get('original_role') %}
    <div class="alert alert-info text-center">
      You are viewing as {{ session['role'] }}.
      <form action="{{ url_for('return_admin') }}" method="POST" style="display:inline;">
        <button class="btn btn-warning btn-sm">⬅ Return to {{ session['original_role'] }} View</button>
      </form>
    </div>
    {% endif %}

    <h3>Welcome, {{ session['username'] }} ({{ session['role'] }})</h3>
    <hr>

    {% if session['role'] in ["Administrator", "Developer"] %}
    <h4>Manage Users</h4>
    <table class="table table-striped">
      <thead><tr><th>ID</th><th>Username</th><th>Role</th><th>Action</th></tr></thead>
      <tbody>
      {% for u in users %}
      <tr><td>{{ u.id }}</td><td>{{ u.username }}</td><td>{{ u.role }}</td>
        <td>{% if u.username != session['username'] %}<form method="POST" action="{{ url_for('delete_user', id=u.id) }}"><button class="btn btn-danger btn-sm">Delete</button></form>{% endif %}</td>
      </tr>{% endfor %}
      </tbody>
    </table>

    <form method="POST" action="{{ url_for('add_user') }}" class="row g-3 mb-4">
      <div class="col-md-3"><input name="username" class="form-control" placeholder="Username" required></div>
      <div class="col-md-3"><input type="password" name="password" class="form-control" placeholder="Password" required></div>
      <div class="col-md-3"><select name="role" class="form-select" required><option value="">Choose Role</option><option>Administrator</option><option>Investor</option><option>Researcher</option><option>Developer</option></select></div>
      <div class="col-md-3"><button class="btn btn-success w-100">Add User</button></div>
    </form>

    <h4>Manage Minerals</h4>
    <table class="table table-bordered">
      <thead><tr><th>ID</th><th>Country</th><th>Mineral</th><th>Quantity</th><th>Action</th></tr></thead>
      <tbody>
      {% for m in minerals %}
      <tr><td>{{ m.id }}</td><td>{{ m.country }}</td><td>{{ m.mineral }}</td><td>{{ m.quantity }}</td>
        <td><form method="POST" action="{{ url_for('delete_mineral', id=m.id) }}"><button class="btn btn-danger btn-sm">Delete</button></form></td>
      </tr>{% endfor %}
      </tbody>
    </table>

    <form method="POST" action="{{ url_for('add_mineral') }}" class="row g-3 mb-4">
      <div class="col-md-2"><input name="country" class="form-control" placeholder="Country" required></div>
      <div class="col-md-2"><input name="mineral" class="form-control" placeholder="Mineral" required></div>
      <div class="col-md-2"><input name="quantity" type="number" class="form-control" placeholder="Quantity" required></div>
      <div class="col-md-2"><input name="lat" step="0.0001" type="number" class="form-control" placeholder="Latitude" required></div>
      <div class="col-md-2"><input name="lon" step="0.0001" type="number" class="form-control" placeholder="Longitude" required></div>
      <div class="col-md-2"><button class="btn btn-success w-100">Add</button></div>
    </form>

    <h4>Country Profiles</h4>
    <table class="table table-bordered">
      <thead><tr><th>Country</th><th>GDP (Billion USD)</th><th>Key Projects</th><th>Notes</th><th>Action</th></tr></thead>
      <tbody>
      {% for c in country_profiles %}
      <tr>
        <td>{{ c.country }}</td>
        <td>{{ "%.1f"|format(c.gdp) if c.gdp else "N/A" }}</td>
        <td>{{ c.key_projects }}</td>
        <td>{{ c.notes }}</td>
        <td><form method="POST" action="{{ url_for('delete_country', id=c.id) }}"><button class="btn btn-danger btn-sm">Delete</button></form></td>
      </tr>{% endfor %}
      </tbody>
    </table>

    <form method="POST" action="{{ url_for('add_country') }}" class="row g-3 mb-4">
      <div class="col-md-3"><input name="country" class="form-control" placeholder="Country" required></div>
      <div class="col-md-2"><input name="gdp" type="number" step="0.1" class="form-control" placeholder="GDP (Billion USD)"></div>
      <div class="col-md-3"><input name="key_projects" class="form-control" placeholder="Key Projects"></div>
      <div class="col-md-3"><input name="notes" class="form-control" placeholder="Notes"></div>
      <div class="col-md-1"><button class="btn btn-success w-100">Add</button></div>
    </form>
    {% endif %}

    {% if session['role'] in ["Researcher", "Investor"] %}
    <h4>Country Profiles</h4>
    <table class="table table-bordered">
      <thead><tr><th>Country</th><th>GDP (Billion USD)</th><th>Key Projects</th><th>Notes</th></tr></thead>
      <tbody>
      {% for c in country_profiles %}
      <tr><td>{{ c.country }}</td><td>{{ "%.1f"|format(c.gdp) if c.gdp else "N/A" }}</td><td>{{ c.key_projects }}</td><td>{{ c.notes }}</td></tr>{% endfor %}
      </tbody>
    </table>

    <form method="GET" action="{{ url_for('dashboard') }}" class="row g-3 mb-4">
      <div class="col-md-6"><select name="country" class="form-select"><option value="">All Countries</option>{% for c in countries %}<option value="{{ c }}" {% if request.args.get('country') == c %}selected{% endif %}>{{ c }}</option>{% endfor %}</select></div>
      <div class="col-md-6"><select name="mineral" class="form-select"><option value="">All Minerals</option>{% for m in mineral_types %}<option value="{{ m }}" {% if request.args.get('mineral') == m %}selected{% endif %}>{{ m }}</option>{% endfor %}</select></div>
      <div class="col-md-12 mt-2"><button class="btn btn-primary w-100">Apply Filters</button></div>
    </form>

    <iframe src="{{ url_for('map_page', country=request.args.get('country'), mineral=request.args.get('mineral')) }}" width="100%" height="500"></iframe>
    <h4 class="mt-5">Interactive Mineral Chart</h4>
    <iframe src="{{ url_for('charts_page', country=request.args.get('country'), mineral=request.args.get('mineral')) }}" width="100%" height="550"></iframe>
    {% endif %}
  </div>
  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
"""


