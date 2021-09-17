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