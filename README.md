# Python Flask Tutorial

# Flask configurations and first step

### Setting Virtual environment  
Virtual environments are independent groups of Python libraries, one for each project. Packages installed for one project will not affect other projects or the operating system’s packages.

## Create an environment
1. First create a project folder and a venv folder within

        mkdir myproject
        cd myproject
        python3 -m venv venv

2. Active the environment

        . venv/bin/activate

3. Install Flask

        pip install Flask
        
        
# Creating the first Flask app

Create a new document and save in project directory

    mkdir hello.py
First we imported the Flask class. An instance of this class will be our WSGI application

    from flask import Flask
>A Python package containing your application code and files use in flask.

Next we create an instance of this class. The first argument is the name of the application’s module or package. __name__ is a convenient shortcut for this that is appropriate for most cases. This is needed so that Flask knows where to look for resources such as templates and static files

    app = Flask(__name__)

We then use the route() decorator to tell Flask what URL should trigger our function.

    @app.route("/")
    def hello_world():
    return "<p>Hello, World!</p>"

# Run the Application on MAC/Linux
Before running you need to tell your terminal the application to work with by exporting the FLASK_APP environment variable:

    export FLASK_APP=hello
Then run:

    flask run

# Flask Advance Config


## Application Factory

The application factory holds any configuration, registration, and other setup the application needs.

1. make new directory

        mkdir flaskr

2. create a new file

        __init__.py
    >it will contain the application factory, and it tells Python that the flaskr directory should be treated as a package.

3. Create and configure the app

        def create_app(test_config=None):
            app = Flask(__name__, instance_relative_config=True)
            app.config.from_mapping(
            SECRET_KEY='dev',
            DATABASE=os.path.join(app.instance_path, 'flaskr.sqlite'),
        )
4. Ensure the instance folder exists

        try:
            os.makedirs(app.instance_path)
        except OSError:
            pass

5. Define your routes

        @app.route('/hello')
        def hello():
            return 'Hello, World!'
        return app

6. Change the env variables 

        export FLASK_APP=flaskr
        export FLASK_ENV=development

# Connect to the Database

1. Create a new file flaskr/db.py
2. Add the configuration:

        import sqlite3
        import click

        from flask import current_app, g
        from flask.cli import with_appcontext


        def get_db():
            if 'db' not in g:
                g.db = sqlite3.connect(
                current_app.config['DATABASE'],
                detect_types=sqlite3.PARSE_DECLTYPES
            )
            g.db.row_factory = sqlite3.Row
            return g.db


        def close_db(e=None):
            db = g.pop('db', None)
            if db is not None:
            db.close()
>g It is used to store data that might be accessed by multiple functions during the request.

>current_app get_db will be called when the application has been created and is handling a request, so current_app can be used.

# Create the Tables

1. Create a new file in flaskr call schema.sql

2. Add SQL to create the table info
    
        DROP TABLE IF EXISTS user;
        DROP TABLE IF EXISTS post;

        CREATE TABLE user (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT UNIQUE NOT NULL,
            password TEXT NOT NULL
        );

        CREATE TABLE post (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            author_id INTEGER NOT NULL,
            created TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
            title TEXT NOT NULL,
            body TEXT NOT NULL,
            FOREIGN KEY (author_id) REFERENCES user (id)
        );
3. Go back to db.py an add the functions that will run the SQL Commands add the top.

        def init_db():
            db = get_db()
            with current_app.open_resource('schema.sql') as f:
                db.executescript(f.read().decode('utf8'))

        @click.command('init-db')
        @with_appcontext
        def init_db_command():
        """Clear the existing data and create new tables."""
            init_db()
            click.echo('Initialized the database.')

4. After the close_db write a function that takes an application and does the registration.

        def init_app(app):
            app.teardown_appcontext(close_db)
            app.cli.add_command(init_db_command)

5. Go back to the 
        
        __init__.py 

    - import the db info:

            from . import db

    - add the function to the end to register the database

            db.init_app(app)

6. Initialize the Database File

        flask init-db
>There will now be a flaskr.sqlite file in the instance folder in your project.

# Create a Blueprints 
A Blueprint is a way to organize a group of related views and other code. Rather than registering views and other code directly with an application, they are registered with a blueprint. Then the blueprint is registered with the application when it is available in the factory function.


## Example blueprint for auth

1. Make a new file:

        flaskr/auth.py

2. Add the configuration for auth

        import functools

        from flask import (
                Blueprint, flash, g, redirect, render_template, request, session, url_for
        )
        from werkzeug.security import check_password_hash, generate_password_hash

        from flaskr.db import get_db

        bp = Blueprint('auth', __name__, url_prefix='/auth')

3. Register the bluePrint named 'auth' to the application

    - Go to:

            flaskr/__init__.py
    - Import auth:

            from . import auth
    - Register the blueprint:

            app.register_blueprint(auth.bp)

4. Inside auth.py add the sign-up/register function:

        @bp.route('/register', methods=('GET', 'POST'))
        def register():
            if request.method == 'POST':
                username = request.form['username']
                password = request.form['password']
                db = get_db()
            error = None

            if not username:
                error = 'Username is required.'
            elif not password:
                error = 'Password is required.'

            if error is None:
                try:
                    db.execute(
                    "INSERT INTO user (username, password) VALUES (?, ?)",
                    (username, generate_password_hash(password)),
                )
                db.commit()
                except db.IntegrityError:
                    error = f"User {username} is already registered."
            else:
                return redirect(url_for("auth.login"))

            flash(error)

        return render_template('auth/register.html')

5. Add the Login to auth.py:

        @bp.route('/login', methods=('GET', 'POST'))
        def login():
            if request.method == 'POST':
                username = request.form['username']
                password = request.form['password']
            db = get_db()
            error = None
            user = db.execute(
                'SELECT * FROM user WHERE username = ?', (username,)
            ).fetchone()

            if user is None:
                error = 'Invalid Credentials.'
            elif not check_password_hash(user['password'], password):
                error = 'Invalid Credentials.'

            if error is None:
                session.clear()
                session['user_id'] = user['id']
                return redirect(url_for('index'))

        flash(error)

        return render_template('auth/login.html')
6. Add the Logout func to auth.py:

        @bp.route('/logout')
        def logout():
            session.clear()
            return redirect(url_for('index'))
7. Require Authentication in Other Views inside auth.py

        def login_required(view):
        @functools.wraps(view)
        def wrapped_view(**kwargs):
            if g.user is None:
                return redirect(url_for('auth.login'))

            return view(**kwargs)

         return wrapped_view

# Create The Template Views

1. Create a template folder inside your project directory.
2. Create a base.html inside the templates folder
      
        <!doctype html>
            <title>{% block title %}{% endblock %} - Flaskr</title>
            <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">

        <nav>
            <h1>Flaskr</h1>
            <ul>
                {% if g.user %}
                    <li><span>{{ g.user['username'] }}</span>
                    <li><a href="{{ url_for('auth.logout') }}">Log Out</a>
                {% else %}
                    <li><a href="{{ url_for('auth.register') }}">Register</a>
                    <li><a href="{{ url_for('auth.login') }}">Log In</a>
                {% endif %}
            </ul>
        </nav>

        <section class="content">
            <header>
                {% block header %}{% endblock %}
            </header>
                {% for message in get_flashed_messages() %}
            <div class="flash">{{ message }}</div>
            {% endfor %}
            {% block content %}{% endblock %}
        </section>

3. Inside the template folder create new auth folder an add your base html to login and register with the same name of your routes

    - Register
        
                {% extends 'base.html' %}

                {% block header %}
                    <h1>{% block title %}Register{% endblock %}</h1>
                {% endblock %}

                {% block content %}
                    <form method="post">
                        <label for="username">Username</label>
                        <input name="username" id="username" required>
                        <label for="password">Password</label>
                        <input type="password" name="password" id="password" required>
                        <input type="submit" value="Register">
                    </form>
                {% endblock %}
    - Log In

                {% extends 'base.html' %}
                {% block header %}
                    <h1>{% block title %}Log In{% endblock %}</h1>
                {% endblock %}

                {% block content %}
                <form method="post">
                    <label for="username">Username</label>
                    <input name="username" id="username" required>
                    <label for="password">Password</label>
                    <input type="password" name="password" id="password" required>
                    <input type="submit" value="Log In">
                </form>
                {% endblock %}
4. Register your user, go to http://127.0.0.1:5000/auth/register. with the server running

# Create the Blog BluePrint

1. Inside flaskr create a new file name blog.py
2. Add the following code:
      
        from flask import (
            Blueprint, flash, g, redirect, render_template, request, url_for
        )
        from werkzeug.exceptions import abort

        from flaskr.auth import login_required
        from flaskr.db import get_db

        bp = Blueprint('blog', __name__)


        @bp.route('/')
        def index():
            db = get_db()
            posts = db.execute(
                'SELECT p.id, title, body, created, author_id, username'
                ' FROM post p JOIN user u ON p.author_id = u.id'
                ' ORDER BY created DESC'
            ).fetchall()
            return render_template('blog/index.html', posts=posts)

        @bp.route('/create', methods=('GET', 'POST'))
        @login_required
        def create():
            if request.method == 'POST':
                title = request.form['title']
                body = request.form['body']
                error = None

             if not title:
                error = 'Title is required.'

            if error is not None:
                flash(error)
            else:
                db = get_db()
                db.execute(
                    'INSERT INTO post (title, body, author_id)'
                    ' VALUES (?, ?, ?)',
                    (title, body, g.user['id'])
                )
                db.commit()
                return redirect(url_for('blog.index'))

        return render_template('blog/create.html')


        def get_post(id, check_author=True):
            post = get_db().execute(
            'SELECT p.id, title, body, created, author_id, username'
            ' FROM post p JOIN user u ON p.author_id = u.id'
            ' WHERE p.id = ?',
            (id,)
            ).fetchone()

            if post is None:
                abort(404, f"Post id {id} doesn't exist.")

            if check_author and post['author_id'] != g.user['id']:
                abort(403)

        return post


        @bp.route('/<int:id>/update', methods=('GET', 'POST'))
        @login_required
        def update(id):
            post = get_post(id)

            if request.method == 'POST':
                title = request.form['title']
                body = request.form['body']
                error = None

            if not title:
                error = 'Title is required.'

            if error is not None:
                flash(error)
            else:
                db = get_db()
                db.execute(
                    'UPDATE post SET title = ?, body = ?'
                    ' WHERE id = ?',
                    (title, body, id)
                )
            db.commit()
            return redirect(url_for('blog.index'))

        return render_template('blog/update.html', post=post)

        @bp.route('/<int:id>/delete', methods=('POST',))
        @login_required
        def delete(id):
            get_post(id)
            db = get_db()
            db.execute('DELETE FROM post WHERE id = ?', (id,))
            db.commit()
        return redirect(url_for('blog.index'))

3. Create a new folder for name blog in the templates folder.
4. Create a index.html inside the blog folder to show all the post:

        {% extends 'base.html' %}

        {% block header %}
        <h1>{% block title %}Posts{% endblock %}</h1>
        {% if g.user %}
            <a class="action" href="{{ url_for('blog.create') }}">New</a>
        {% endif %}
        {% endblock %}

        {% block content %}
        {% for post in posts %}
        <article class="post">
            <header>
                <div>
                    <h1>{{ post['title'] }}</h1>
                    <div class="about">by {{ post['username'] }} on {{ post['created'].strftime('%Y-%m-%d') }}</div>
                </div>
                {% if g.user['id'] == post['author_id'] %}
                    <a class="action" href="{{ url_for('blog.update', id=post['id']) }}">Edit</a>
                {% endif %}
            </header>
            <p class="body">{{ post['body'] }}</p>
            </article>
            {% if not loop.last %}
            <hr>
            {% endif %}
            {% endfor %}
            {% endblock %}

5. Create a create.html inside the blog folder add the following:
    
            {% extends 'base.html' %}

            {% block header %}
                h1>{% block title %}New Post{% endblock %}</h1>
            {% endblock %}

            {% block content %}
                <form method="post">
                    <label for="title">Title</label>
                    <input name="title" id="title" value="{{ request.form['title'] }}" required>
                    <label for="body">Body</label>
                    <textarea name="body" id="body">{{ request.form['body'] }}</textarea>
                    <input type="submit" value="Save">
                </form>
            {% endblock %}
6. Create a update.html inside the blog folder

            {% extends 'base.html' %}

            {% block header %}
                <h1>{% block title %}Edit "{{ post['title'] }}"{% endblock %}</h1>
            {% endblock %}

            {% block content %}
            <form method="post">
                <label for="title">Title</label>
                <input name="title" id="title"
                value="{{ request.form['title'] or post['title'] }}" required>
                <label for="body">Body</label>
                <textarea name="body" id="body">{{ request.form['body'] or post['body'] }}</textarea>
                <input type="submit" value="Save">
            </form>
            <hr>
            <form action="{{ url_for('blog.delete', id=post['id']) }}" method="post">
            <input class="danger" type="submit" value="Delete" onclick="return confirm('Are you sure?');">
            </form>
            {% endblock %}
