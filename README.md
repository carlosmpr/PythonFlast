# Python Flask Tutorial

# Flask configurations and first step

### Setting Virtual envirotment  
Virtual environments are independent groups of Python libraries, one for each project. Packages installed for one project will not affect other projects or the operating system’s packages.

## Create an environment
1. First create a project folder and a venv folder withim

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

# Run the Aplication on MAC/Linux
Before running you need to tell your terminal the application to work with by exporting the FLASK_APP environment variable:

    export FLASK_APP=hello
Then run:

    flask run

# Flask Advance Conifg


## Aplication Factory

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

6. Change the env varibles 

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

4. Inside auth.py add the sign up/register function:

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



    
    
