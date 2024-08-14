from flask import Flask, render_template, request, redirect, url_for, session
from flask_socketio import SocketIO, emit
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your_secret_key'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///db.sqlite3'
db = SQLAlchemy(app)
socketio = SocketIO(app)

# Database Models
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(100), unique=True, nullable=False)
    messages = db.relationship('Message', backref='user', lazy=True)

class Message(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    content = db.Column(db.String(500), nullable=False)

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username']
        if User.query.filter_by(username=username).first():
            return 'Username already taken'
        new_user = User(username=username)
        db.session.add(new_user)
        db.session.commit()
        session['username'] = username
        return redirect(url_for('chat'))
    return render_template('register.html')

@app.route('/chat')
def chat():
    if 'username' not in session:
        return redirect(url_for('register'))
    return render_template('chat.html', username=session['username'])

@socketio.on('send_message')
def handle_message(data):
    username = session['username']
    user = User.query.filter_by(username=username).first()
    new_message = Message(user_id=user.id, content=data['message'])
    db.session.add(new_message)
    db.session.commit()
    emit('receive_message', {'username': username, 'message': data['message']}, broadcast=True)

if __name__ == '__main__':
    db.create_all()
    socketio.run(app, debug=True)
