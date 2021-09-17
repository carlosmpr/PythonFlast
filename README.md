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
