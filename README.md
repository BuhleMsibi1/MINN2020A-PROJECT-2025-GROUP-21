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
