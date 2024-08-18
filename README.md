- 👋 Hi, I’m @fegfhwgduwhu
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...

<!---
fegfhwgduwhu/fegfhwgduwhu is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
from flask import Flask, render_template, request, redirect, url_for, flash, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager, UserMixin, login_user, logout_user, login_required, current_user
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, TextAreaField, BooleanField
from wtforms.validators import InputRequired, Length, Email, EqualTo
from werkzeug.security import generate_password_hash, check_password_hash
from itsdangerous import URLSafeSerializer, BadSignature
import os
import json
import requests

app = Flask(__name__)
app.config["SECRET_KEY"] = "secret_key_here"
app.config["SQLALCHEMY_DATABASE_URI"] = "sqlite:///database.db"
app.config["MAIL_SERVER"] = "smtp.gmail.com"
app.config["MAIL_PORT"] = 587
app.config["MAIL_USE_TLS"] = True
app.config["MAIL_USERNAME"] = "your_email_here"
app.config["MAIL_PASSWORD"] = "your_password_here"
app.config["STRIPE_SECRET_KEY"] = "stripe_secret_key_here"
app.config["STRIPE_PUBLIC_KEY"] = "stripe_public_key_here"
app.config["FACEBOOK_APP_ID"] = "facebook_app_id_here"
app.config["FACEBOOK_APP_SECRET"] = "facebook_app_secret_here"
app.config["TWITTER_API_KEY"] = "twitter_api_key_here"
app.config["TWITTER_API_SECRET"] = "twitter_api_secret_here"

db = SQLAlchemy(app)
login_manager = LoginManager(app)
serializer = URLSafeSerializer(app.config["SECRET_KEY"])

class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(128), nullable=False)
    confirmed = db.Column(db.Boolean, default=False)
    profile_picture = db.Column(db.String(128), nullable=True)
    bio = db.Column(db.Text, nullable=True)

    def set_password(self, password):
        self.password_hash = generate_password_hash(password)

    def check_password(self, password):
        return check_password_hash(self.password_hash, password)

    def generate_confirmation_token(self):
        return serializer.dumps(self.email)

    def confirm(self, token):
        try:
            email = serializer.loads(token)
        except BadSignature:
            return False
        if email == self.email:
            self.confirmed = True
            db.session.commit()
            return True
        return False

class Post(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(100), nullable=False)
    content = db.Column(db.Text, nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey("user.id"))
    user = db.relationship("User", backref=db.backref("posts", lazy=True))

class Comment(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    content = db.Column(db.Text, nullable=False)
    post_id = db.Column(db.Integer, db.ForeignKey("post.id"))
    post = db.relationship("Post", backref=db.backref("comments", lazy=True))
    user_id = db.Column(db.Integer, db.ForeignKey("user.id"))
    user = db.relationship("User", backref=db.backref("comments", lazy=True))

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))

class LoginForm(FlaskForm):
    username = StringField("Username", validators=[InputRequired(), Length(min=4, max=64)])
    password = PasswordField("Password", validators=[InputRequired(), Length(min=8)])

class RegisterForm(FlaskForm):
    username = StringField("Username",
