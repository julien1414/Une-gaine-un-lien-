# Une-gaine-un-lien-
Jardin partagé 
C'est quoi # ─────────────────────────────────────────────────────────────
 
# Projet : Jardin Partagé « Une graine, un lien »
 
# Stack   : Python 3.10+, Flask, SQLite, Tailwind (CDN), Leaflet (OSM)
 
# ─────────────────────────────────────────────────────────────
 
# Arborescence suggérée
 
# 
 
# jardin-partage/
 
# ├─ app.py
 
# ├─ config.py
 
# ├─ requirements.txt
 
# ├─ .env.example
 
# ├─ instance/
 
# │   └─ jardin.db           # créé automatiquement
 
# ├─ static/
 
# │   ├─ css/style.css
 
# │   ├─ js/map.js
 
# │   └─ img/logo.png        # placez ici votre logo
 
# └─ templates/
 
# ├─ base.html
 
# ├─ index.html
 
# ├─ association.html
 
# ├─ parcelles.html
 
# ├─ evenements.html
 
# ├─ benevolat.html
 
# ├─ contact.html
 
# └─ 404.html
 
# 
 
# Copiez chaque bloc dans les fichiers indiqués ci‑dessous.
 
# ─────────────────────────────────────────────────────────────
 
# ===========================
 
# requirements.txt
 
# ===========================
 
# Flask==3.0.3
 
# Flask-SQLAlchemy==3.1.1
 
# python-dotenv==1.0.1
 
# itsdangerous==2.2.0
 
# ===========================
 
# .env.example
 
# ===========================
 
# FLASK_DEBUG=1
 
# SECRET_KEY=changez-moi-en-une-chaine-secrete-longue
 
# DATABASE_URL=sqlite:///jardin.db
 
# ADMIN_USER=admin
 
# ADMIN_PASSWORD=motdepasse-tres-secret
 
# SITE_NOM=Jardin partagé – Une graine, un lien
 
# SITE_COMMUNE=Votre Commune
 
# SITE_ADRESSE=Place du Jardin, 00000 Votre Commune, France
 
# LAT=48.8566
 
# LNG=2.3522
 
# ===========================
 
# config.py
 
# ===========================
 
from **future** import annotations import os from pathlib import Path
 
class Config: SECRET_KEY = os.getenv("SECRET_KEY", "dev-secret") SQLALCHEMY_DATABASE_URI = os.getenv("DATABASE_URL", "sqlite:///jardin.db") SQLALCHEMY_TRACK_MODIFICATIONS = False SITE_NOM = os.getenv("SITE_NOM", "Jardin partagé – Une graine, un lien") SITE_COMMUNE = os.getenv("SITE_COMMUNE", "Votre Commune") SITE_ADRESSE = os.getenv("SITE_ADRESSE", "Adresse municipale") LAT = float(os.getenv("LAT", "48.8566")) LNG = float(os.getenv("LNG", "2.3522")) ADMIN_USER = os.getenv("ADMIN_USER", "admin") ADMIN_PASSWORD = os.getenv("ADMIN_PASSWORD", "admin")
 
# ===========================
 
# app.py
 
# ===========================
 
from **future** import annotations import os from datetime import datetime from functools import wraps
 
from flask import Flask, render_template, request, redirect, url_for, flash, abort from flask_sqlalchemy import SQLAlchemy from werkzeug.security import safe_str_cmp
 
app = Flask(**name**, instance_relative_config=True) app.config.from_object("config.Config")
 
# Assure le dossier instance/
 
Path(app.instance_path).mkdir(parents=True, exist_ok=True)
 
db = SQLAlchemy(app)
 
# ── Modèles ─────────────────────────────────────────────────
 
class Subscriber(db.Model): id = db.Column(db.Integer, primary_key=True) name = db.Column(db.String(120), nullable=False) email = db.Column(db.String(200), nullable=False, unique=True) created_at = db.Column(db.DateTime, default=datetime.utcnow)
 
class Message(db.Model): id = db.Column(db.Integer, primary_key=True) name = db.Column(db.String(120), nullable=False) email = db.Column(db.String(200), nullable=False) subject = db.Column(db.String(200), nullable=False) body = db.Column(db.Text, nullable=False) created_at = db.Column(db.DateTime, default=datetime.utcnow)
 
class Event(db.Model): id = db.Column(db.Integer, primary_key=True) title = db.Column(db.String(200), nullable=False) date = db.Column(db.Date, nullable=False) time = db.Column(db.String(20), nullable=True) location = db.Column(db.String(200), nullable=True) description = db.Column(db.Text, nullable=True) created_at = db.Column(db.DateTime, default=datetime.utcnow)
 
# Création DB au premier run
 
with app.app_context(): db.create_all()
 
# ── Helpers ─────────────────────────────────────────────────
 
def admin_required(view_func): @wraps(view_func) def wrapper(*args, **kwargs): user = request.authorization.username if request.authorization else None pwd = request.authorization.password if request.authorization else None if not user or not pwd: return _auth_required() if not (safe_str_cmp(user, app.config["ADMIN_USER"]) and safe_str_cmp(pwd, app.config["ADMIN_PASSWORD"])): return _auth_required() return view_func(*args, **kwargs) return wrapper
 
def _auth_required(): return ("Authentification requise", 401, {"WWW-Authenticate": "Basic realm='Admin'"})
 
# ── Routes publiques ────────────────────────────────────────
 
@app.route("/") def index(): upcoming = Event.query.filter(Event.date >= datetime.utcnow().date()).order_by(Event.date.asc()).all() return render_template("index.html", upcoming=upcoming)
 
@app.route("/association") def association(): return render_template("association.html")
 
@app.route("/parcelles") def parcelles(): return render_template("parcelles.html")
 
@app.route("/evenements") def evenements(): events = Event.query.order_by(Event.date.asc()).all() return render_template("evenements.html", events=events)
 
@app.route("/benevolat") def benevolat(): return render_template("benevolat.html")
 
@app.route("/inscription", methods=["POST"]) def inscription(): name = request.form.get("name", "").strip() email = request.form.get("email", "").strip().lower() if not name or not email: flash("Merci d'indiquer votre nom et votre email.", "error") return redirect(request.referrer or url_for("index")) if Subscriber.query.filter_by(email=email).first(): flash("Cet email est déjà inscrit.", "warning") return redirect(request.referrer or url_for("index")) db.session.add(Subscriber(name=name, email=email)) db.session.commit() flash("Bienvenue ! Vous êtes inscrit·e à la newsletter du jardin.", "success") return redirect(url_for("index"))
 
@app.route("/contact") def contact(): return render_template("contact.html")
 
@app.route("/contact", methods=["POST"]) def contact_post(): name = request.form.get("name", "").strip() email = request.form.get("email", "").strip() subject = request.form.get("subject", "").strip() body = request.form.get("body", "").strip() if not all([name, email, subject, body]): flash("Tous les champs sont requis.", "error") return redirect(url_for("contact")) db.session.add(Message(name=name, email=email, subject=subject, body=body)) db.session.commit() flash("Merci pour votre message ! Nous revenons vers vous rapidement.", "success") return redirect(url_for("contact"))
 
# ── Routes admin (ajout d'évènements) ───────────────────────
 
@app.route("/admin/evenements/nouveau", methods=["GET", "POST"]) @admin_required def admin_evenement_nouveau(): if request.method == "POST": try: title = request.form.get("title", "").strip() date_str = request.form.get("date", "").strip() time = request.form.get("time", "").strip() location = request.form.get("location", "").strip() description = request.form.get("description", "").strip() date_obj = datetime.strptime(date_str, "%Y-%m-%d").date() if not title: raise ValueError("Titre requis") ev = Event(title=title, date=date_obj, time=time, location=location, description=description) db.session.add(ev) db.session.commit() flash("Évènement ajouté !", "success") return redirect(url_for("evenements")) except Exception as e: flash(f"Erreur: {e}", "error") return render_template("admin_nouveau_evenement.html")
 
# ── Gestion erreurs ─────────────────────────────────────────
 
@app.errorhandler(404) def not_found(e): return render_template("404.html"), 404
 
# ── Lancement ───────────────────────────────────────────────
 
if **name** == "**main**": app.run(debug=bool(int(os.getenv("FLASK_DEBUG", "1"))))
 
# ===========================
 
# templates/base.html
 
# ===========================
 
# 
 
# 
 
# 
 
# 
 
# 
 
# {{ config.SITE_NOM }}
 
# 
 
# 
 
# 
 
# 
 
# 
 
# 
 
# 
 
# 
 
# 
 
# 
 
# 
 
# 
 
# 
 
# logo
 
# {{ config.SITE_NOM }}
 
# 
 
# Association
 
# Parcelles
 
# Évènements
 
# Adhérer / Bénévolat
 
# Contact
 
# 
 
# 
 
# 
 
# 
 
# {% block content %}{% endblock %}
 
# 
 
# 
 
# 
 
# 
 
# 
### {{ config.SITE_NOM }}
 
# 
{{ config.SITE_ADRESSE }}
 
# 
Terrain communal – {{ config.SITE_COMMUNE }}
 
# 
 
# 
 
# 
### Liens
 
# 

 
# 
- Statuts & charte
 
# 
- Agenda
 
# 
- Parcelles
 
# 
 
# 
 
# 
 
# 
### Newsletter
 
# 
 
# 
 
# 
 
# S'inscrire
 
# 
 
# 
 
# 
 
# 
© {{ now().year }} {{ config.SITE_NOM }} – Fait avec ❤, solidarité & compost.
 
# 
 
# 
 
# {% with messages = get_flashed_messages(with_categories=True) %}
 
# {% if messages %}
 
# 
 
# {% for cat, msg in messages %}
 
# 
{{ msg }}
 
# {% endfor %}
 
# 
 
# {% endif %}
 
# {% endwith %}
 
# 
 
# 
 
# ===========================
 
# templates/index.html
 
# ===========================
 
# {% extends 'base.html' %}
 
# {% block content %}
 
# 
 
# 
 
# 
# Jardin partagé« Une graine, un lien »
 
# 
Un lieu d'apprentissage, d'écologie et de lien social, ouvert à toutes et tous. Rejoignez l'association pour cultiver des parcelles, participer aux ateliers et faire pousser la solidarité.
 
# 
 
# Adhérer
 
# Voir l'agenda
 
# 
 
# 
 
# 
 
# 
 
# 
Le jardin se situe au terrain communal – {{ config.SITE_COMMUNE }}.
 
# 
 
# 
 
# 
 
# 
 
# 
 
# 
### Écologie
 
# 
Compost partagé, récupération d'eau, semences paysannes, zéro pesticide.
 
# 
 
# 
 
# 
### Lien social
 
# 
Ateliers intergénérationnels, entraide et évènements conviviaux.
 
# 
 
# 
 
# 
### Partage
 
# 
Parcelles individuelles et collectives, outils mutualisés, troc de plants.
 
# 
 
# 
 
# 
 
# 
 
# 
## Prochains évènements
 
# {% if upcoming %}
 
# 

 
# {% for e in upcoming %}
 
# 
- 
 
# 
 
# 
{{ e.title }}
 
# 
{{ e.date.strftime('%d/%m/%Y') }}{% if e.time %} • {{ e.time }}{% endif %}{% if e.location %} • {{ e.location }}{% endif %}
 
# 
 
# Détails
 
# 
 
# {% endfor %}
 
# 
 
# {% else %}
 
# 
Pas d'évènements à venir. Revenez bientôt ou proposez-en un !
 
# {% endif %}
 
# 
 
# {% endblock %}
 
# ===========================
 
# templates/association.html
 
# ===========================
 
# {% extends 'base.html' %}
 
# {% block content %}
 
# 
 
# 
# L'association
 
# 
Notre association, née de l'esprit de partage, promeut une pratique du jardinage respectueuse du vivant et créatrice de liens. Le jardin est implanté sur un **terrain communal** mis à disposition par la mairie.
 
# 
## Principes
 
# 

 
# 
- Respect du sol, de l'eau et de la biodiversité.
 
# 
- Gouvernance participative et bienveillante.
 
# 
- Transmission des savoirs et mixité des publics.
 
# 
 
# 
## Charte
 
# 
• Pas de pesticides ni d'engrais de synthèse.• Compostage des déchets verts.• Partage équitable des ressources (eau, outils, graines).• Horaires calmes et respect du voisinage.
 
# 
 
# {% endblock %}
 
# ===========================
 
# templates/parcelles.html
 
# ===========================
 
# {% extends 'base.html' %}
 
# {% block content %}
 
# 
 
# 
# Parcelles & fonctionnement
 
# 
Le jardin propose des parcelles individuelles et une grande zone collective. Les inscriptions se font en début d'année, avec une liste d'attente si besoin.
 
# 
## Ressources partagées
 
# 

 
# 
- Cabane à outils, récupérateur d'eau, composteurs.
 
# 
- Bibliothèque de semences (graines reproductibles).
 
# 
- Table de rempotage et coin convivial.
 
# 
 
# 
 
# {% endblock %}
 
# ===========================
 
# templates/evenements.html
 
# ===========================
 
# {% extends 'base.html' %}
 
# {% block content %}
 
# 
 
# 
# Agenda des évènements
 
# {% if events %}
 
# 

 
# {% for e in events %}
 
# 
- 
 
# 
 
# 
{{ e.title }}
 
# 
{{ e.date.strftime('%d/%m/%Y') }}{% if e.time %} • {{ e.time }}{% endif %}{% if e.location %} • {{ e.location }}{% endif %}
 
# {% if e.description %}
{{ e.description }}
{% endif %}
 
# 
 
# 
 
# {% endfor %}
 
# 
 
# {% else %}
 
# 
Aucun évènement pour le moment.
 
# {% endif %}
 
# 
Admin : `/admin/evenements/nouveau` (authentification basique)
 
# 
 
# {% endblock %}
 
# ===========================
 
# templates/benevolat.html
 
# ===========================
 
# {% extends 'base.html' %}
 
# {% block content %}
 
# 
 
# 
# Adhérer & Bénévolat
 
# 
Envie de nous rejoindre ? L'adhésion (prix libre, conseillé : 10€) permet d'assurer l'assurance et l'achat de semences. Chacun·e peut contribuer selon ses envies : arrosage, bricolage, ateliers, communication…
 
# 
 
# 
## S'inscrire à la newsletter
 
# 
 
# 
 
# 
 
# Valider
 
# 
 
# 
 
# 
 
# {% endblock %}
 
# ===========================
 
# templates/contact.html
 
# ===========================
 
# {% extends 'base.html' %}
 
# {% block content %}
 
# 
 
# 
# Contact
 
# 
 
# 
 
# 
 
# 
 
# 
 
# Envoyer
 
# 
 
# 
 
# {% endblock %}
 
# ===========================
 
# templates/admin_nouveau_evenement.html
 
# ===========================
 
# {% extends 'base.html' %}
 
# {% block content %}
 
# 
 
# 
# Nouvel évènement
 
# 
 
# 
 
# 
 
# 
 
# 
 
# 
 
# Publier
 
# 
 
# 
 
# {% endblock %}
 
# ===========================
 
# templates/404.html
 
# ===========================
 
# {% extends 'base.html' %}
 
# {% block content %}
 
# 
 
# 
# 404
 
# 
Oups, cette page a été compostée…
 
# Retour à l'accueil
 
# 
 
# {% endblock %}
 
# ===========================
 
# static/css/style.css
 
# ===========================
 
# :root { --ring: 0 0% 100%; }
 
# .btn { @apply inline-flex items-center justify-center rounded-2xl px-4 py-2 font-semibold bg-emerald-700 text-white shadow hover:bg-emerald-800 transition; }
 
# .btn-outline { @apply inline-flex items-center justify-center rounded-2xl px-4 py-2 font-semibold border border-emerald-700 text-emerald-700 hover:bg-emerald-700 hover:text-white transition; }
 
# .btn-sm { @apply inline-flex items-center justify-center rounded-xl px-3 py-1.5 text-sm font-semibold bg-emerald-700 text-white hover:bg-emerald-800; }
 
# .input { @apply w-full rounded-xl border border-emerald-200 px-3 py-2 focus:outline-none focus:ring-2 focus:ring-emerald-400; }
 
# .textarea { @apply w-full rounded-xl border border-emerald-200 px-3 py-2 focus:outline-none focus:ring-2 focus:ring-emerald-400; }
 
# .nav { @apply px-2 py-1 rounded-xl hover:bg-emerald-50; }
 
# .section-title { @apply text-3xl font-extrabold mb-4; }
 
# .h2 { @apply text-xl font-bold mt-6 mb-2; }
 
# .card { @apply bg-white rounded-2xl shadow p-6; }
 
# .card-title { @apply text-xl font-semibold mb-1; }
 
# .item { @apply bg-white rounded-xl p-4 shadow flex items-center justify-between; }
 
# .link { @apply underline underline-offset-4 hover:no-underline; }
 
# .toast { @apply px-4 py-2 rounded-xl text-white shadow; }
 
# .toast.success { @apply bg-emerald-600; }
 
# .toast.error { @apply bg-red-600; }
 
# .toast.warning { @apply bg-amber-600; }
 
# ===========================
 
# static/js/map.js
 
# ===========================
 
# document.addEventListener('DOMContentLoaded', () => {
 
# const el = document.getElementById('map');
 
# if (!el) return;
 
# const lat = parseFloat('{{ config.LAT }}');
 
# const lng = parseFloat('{{ config.LNG }}');
 
# const map = L.map('map').setView([lat, lng], 16);
 
# L.tileLayer('[https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png](https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png)', {
 
# maxZoom: 19,
 
# attribution: '© OpenStreetMap'
 
# }).addTo(map);
 
# L.marker([lat, lng]).addTo(map).bindPopup("{{ config.SITE_NOM }}{{ config.SITE_ADRESSE }}").openPopup();
 
# });
 
# ===========================
 
# Guide de démarrage (README rapide)
 
# ===========================
 
# 1) Créez le projet :
 
# mkdir jardin-partage && cd jardin-partage
 
# python3 -m venv .venv && source .venv/bin/activate
 
# pip install -r requirements.txt
 
# 2) Configurez vos variables :
 
# cp .env.example .env   # puis éditez .env
 
# 3) Placez votre logo dans static/img/logo.png (PNG carré recommandé).
 
# 4) Lancez :
 
# flask --app app run --debug
 
# 5) Ouvrez [http://127.0.0.1:5000](http://127.0.0.1:5000)
 
# 6) Ajoutez des évènements via /admin/evenements/nouveau (auth basique).
 
# 
 
# Personnalisation :
 
# • Modifiez LAT/LNG et SITE_ADRESSE dans .env pour positionner le jardin.
 
# • Ajoutez des pages en créant de nouveaux templates + routes.
 
# • Pour déployer (Railway/Render/Fly.io), définissez les variables d'env et utilise
