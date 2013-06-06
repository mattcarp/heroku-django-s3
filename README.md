Django Heroku Project
================

####*A clean project of a Django web-framework running on Heroku with static files served from Amazon S3, attempting to follow the [12-factor](http://www.12factor.net/) design pattern.*

**Django** is a web framework that simplifies the work required to make a web app in python.

**Heroku** is touted as one of the best 'platform as a service' providers. You can host apps >without having to get too involved with installing your own serving software etc.

**Amazon S3** is a quick, cheap way to host static files (images, js, css etc. that doesn't >change) because heroku doesn't do that.

This collection is touted as one fo the best ways to serve up django apps, that's free initially and can scale up easily. This repo gets it up and running quickly and securely.

### Pre-requisites
This setup using the excellent virtualenvwrapper to isolate the installed dependencies and environmental variables.

- heroku account and [toolbelt](https://toolbelt.heroku.com/) installed on your computer
- git installed
- [virtualenv](https://pypi.python.org/pypi/virtualenv) and [virtualenvwrapper](https://bitbucket.org/dhellmann/virtualenvwrapper)
- an Amazon Web Services account

## Installation

1. Make a new virtualenv and clone this repo
        
    ```sh
    mkvirtualenv [name-of-your-project]
    git clone git@github.com:mattcarp/heroku-django-s3.git [name-of-your-project]
    cd [name-of-your-project]
    setvirtualenvproject
    ```
2. Install the development environment dependencies with pip. These are specified in requirements/dev.txt. Becuase requirements may differ slightly between environments (for instance you may want test coverage and a debug toolbar in your developmnet env but not in produciton), separate requirements files are placed in the requirements directory.  dev.txt and prod.txt inherit a common set of requirements from common.txt.  Heroku will see the requirements.txt file at the root level of the project, which inherits from requirements/prod.txt and will therefore install only production requirements.
If you leave versio numbers out of this file it will install the latest versions available.

    ```sh
    pip install -r requirements.txt
    ```
3. Create your database.  Make a note of its location for the next step.

4. All the private or environment-dependant settings in `settings.py` are kept as environment variables.
    We need to set these variables everytime we enter this virtual environment. virtualenvwrapper does this with a `postactivate` script. Edit this file:

    ```sh
    vim $VIRTUAL_ENV/bin/postactivate 
    ```

    Add the following variables:

    ```sh
    #!/bin/zsh
    # This hook is run after this virtualenv is activated.
    # Django database - use the location from the previous step
    export DATABASE_URL=postgres://localhost:5432/[your-db]
    # Django static file storage
    export AWS_STORAGE_BUCKET_NAME=[YOUR AWS S3 BUCKET NAME]
    export AWS_ACCESS_KEY_ID=XXXXXXXXXXXXXXXXXXXX
    export AWS_SECRET_ACCESS_KEY=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
    # Django debug setting
    export DJ_DEBUG=True
    export DJ_SECRET_KEY=[Any random sequence of 40ish characters  - django uses it for added security]
    export DJANGO_SETTINGS_MODULE=[name-of-your-project].settings.dev
    ```

5. Reopen the virtualenv to run this script

    ```sh
    workon [name-of-your-project]
    ```

6. Run the following:

    ```sh
    python manage.py syncdb
    python manage.py runserver
    ```
Everything should now work for **local development**.  Check that we can see the admin pages at `http://127.0.0.1:8000/admin/`


7. Freeze your dependencies, then create the app on heroku and push everything there. Heroku will detect it's a python app and install everything in requirements.txt:

    ```sh
    pip freeze > requirements.txt
    heroku create [name-of-your-project]
    git add .
    git commit -m 'initial commit'
    git push heroku master
    ...
    ```

8. The heroku django needs the environmental variables too (`DATABASE_URL` is already set on heroku) so we'll send over the values set locally:

    ```sh
    heroku config:add AWS_STORAGE_BUCKET_NAME=$AWS_STORAGE_BUCKET_NAME
    heroku config:add AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
    heroku config:add AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
    heroku config:add DJ_SECRET_KEY=$DJ_SECRET_KEY
    heroku config:add DJ_DEBUG=False
    ```
    Set the Django setting variable to pick up the prod.py settings file:

    ```sh
    heroku config:add DJANGO_SETTINGS_MODULE=settings.prod
    ```

    You can turn debug on/off by changing the DJ_DEBUG setting (only do this if something has gone wrong:

    ```sh
    heroku config:add DJ_DEBUG=True
    ``` 

8. Should now be ready. Go and build that web app!

    ```sh
    heroku run python manage.py syncdb
    heroku run python manage.py collectstatic
    ...
    heroku open
    ```
    **Go to http://[your-project-name].herokuapp.com/admin and you should see a CSS-styled admin login if it's all worked correctly.**

        
    Lastly, go into settings.py and change the trusted hosts to be specifically your domains (for added security)

    ```python
    ALLOWED_HOSTS = ['[your-project-name].herokuapp.com']
    ```



## Differences from a fresh django install
    
####Programs installed
 - django
 - psycopg2 (to be able to talk to postgreSQL databases)
 - gunicorn (python HTTP server to use from heroku)
 - [dj-database-url](https://github.com/kennethreitz/dj-database-url) (to use a URL environmental variable to reference the location of the database)
 - [django-storages](http://django-storages.readthedocs.org/en/latest/backends/amazon-S3.html) (custom storage backends for django, best S3 support makes use of 'boto'...)
 - Boto (Python interface to Amazon Web Services, simplifies the AWS connection to just the access keys)
 - south (for migrations)

####Changes Made

Requirements are split into common.txt, dev.txt, test.txt, and prod.txt.  The `common` file is imported from the other three.  These are all in the top-level `requirements` directory.  This allows the use of modules in dev and test that you wouldn't want to have installed in prod (such as test runners and code coverage tools).  The top level requirements.txt file just inherits from `settings/prod.txt`, so that Heroku can grab the right file.

####Base template
Added a base layout that can be extended by app templates.  The base imports css and js from twitter bootstrap v3.0.0-wip.  Note that this is a beta version of bootstrap: you'll want to update it.

####`settings.py`

Split settings into common.py, dev.py, test.py, and prod.py.  As with requirements, the `common` file is imported into the other files.  This is done so that you can have installed apps in, say, your test environment, which you don't want in production.You can set the `DJANGO_SETTINGS_MODULE` env variable to pick up the appropriate file for your environment (see above).

Removed `# 'django.contrib.sites'` add-on that can enable multiple sites to use the same back-end but confuses matters here.

As part of the plan to make the settings transferable and secure did the following:

```python
# Helper function to grab env vars, report a useful message if not found:

def get_env_setting(setting):
    """ Get the environment setting or return exception """
    try:
        return environ[setting]
    except KeyError:
        error_msg = "Set the %s env variable" % setting
        raise ImproperlyConfigured(error_msg)

Various env vars are used to allow portability between environments and projects.

Database settings are set by env var:
    
```python   
# Parse database configuration from $DATABASE_URL
import dj_database_url
DATABASES['default'] =  dj_database_url.config(default=get_env_setting('DATABASE_URL'))

# Honor the 'X-Forwarded-Proto' header for request.is_secure()
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
```

`'admin'` and `'admindocs'` enabled.  `'storages', 'south'` added to `INSTALLED_APPS`

Added settings to store statics on S3
    
```python
#Storage on S3 settings are stored as os.environs to keep settings.py clean 
if not DEBUG:
    AWS_STORAGE_BUCKET_NAME = os.environ['AWS_STORAGE_BUCKET_NAME']
    AWS_ACCESS_KEY_ID = os.environ['AWS_ACCESS_KEY_ID']
    AWS_SECRET_ACCESS_KEY = os.environ['AWS_SECRET_ACCESS_KEY']
    STATICFILES_STORAGE = 'storages.backends.s3boto.S3BotoStorage'
    S3_URL = 'http://%s.s3.amazonaws.com/' % AWS_STORAGE_BUCKET_NAME
    STATIC_URL = S3_URL
```

####`urls.py`
Uncommented lines 4, 5, 13 and 16 from urls to enable admin urls

####`Procfile`
Runs gunicorn process for heroku

####`.gitignore`
Ignores common ignorables for python and django development
