---
comments: true
---



# Azure

Deploying on azure is extremely difficult. Here are some notes to self for deploying on Azure.

## Updating Env Vars

Restarting the server is considerably longer than Heroku for some reason and it'll just be unresponsive without status updates for about 5-10 mins, even if it says **"successfully restarted web app"**.

So just be cognizant of that when you restart the web app that it seems like nothing is happening, but it is. Further attempts to re-restart the server only delays your testing.

## Python (Non-Dockerized code) Deployment

### Serving Static Files

It requires whitenoise, so make sure you have whitenoise in your middleware:

```python
MIDDLEWARE = [
        'django.middleware.security.SecurityMiddleware',
        'whitenoise.middleware.WhiteNoiseMiddleware',
        ...
```

`STATICFILES_STORAGE = 'whitenoise.storage.CompressedStaticFilesStorage'`

and `whitenoise` in requirements.txt of course.

### Server Won't Even Start

Despite doing a python deployment, Azure will implicitly add a very slow starting Docker. There's a 4-min limit request limit before the server crashes, and Docker starting up takes 3 and a half minutes. This is bad.

To get around this, make env var:

`WEBSITES_CONTAINER_START_TIME_LIMIT = 600`

When your server starts, you'll run into your server blowing up with 500 errors because Azure will implicitly probe your site with random IP addresses.

### ALLOWED_HOSTS

For some reason, they use health check (even when it is turned off) which are internal addresses that vary when you deploy your server. Thus, you will run into issues with your `ALLOWED_HOSTS`. It changes each time despite setting up a static IP + NAT. To get around this, do this:

```python
import socket
local_ip = str(socket.gethostbyname(socket.gethostname()))

ALLOWED_HOSTS = [
    "localhost",
  	...
    "website.azurewebsites.net",
    local_ip,
]
```

After that, you'll notice your database is busted.

### Postgres

#### Migrations

For some reason, Azure won't run migrations for you when you deploy. So you'll need to shove these things in your startup script. But Azure's documentation for startup scripts is outdated and instead you should:

1. Deploy the app once. You cannot do the below steps before deploying because the script in `/opt/startup/startup.sh` won't know which directory to cd into yet. Once it's deployed:
2. visit the ssh shell at `https://<your site>.scm.azurewebsites.net/webssh/host``
3. ``cp /opt/startup/startup.sh /home/startup.sh`
4. Modify the `startup.sh` to include things you need it to do.
5. Set the App Service > Configuration > General Settings to `/home/startup.sh`

Example, you'd substitute their vanilla gunicorn call with

```bash
echo "migrations"
python manage.py makemigrations
python manage.py migrate
echo "starting server..."
gunicorn --workers 1 --threads 8 --timeout 0 appname.wsgi:application > /dev/stdout 2>&1 &
celery -A appname worker -l INFO --concurrency 1
```

If you do this, you'll note that this doesn't work at all since postgres in Azure requires SSL. Note the `&` in the gunicorn and the piping to stdout so Azure's log service can pick it up. Note we are not allowed to have an `&` for celery as the `startup.sh` will exit, causing an error (the deployment service expects startup.sh to be persistent).

#### SSL

You need to download an SSL cert, put it in your project's root directory and update your Django settings like this:

```
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql_psycopg2",
        "NAME": os.environ.get("DJANGO_DATABASE_NAME", "db_name"),
        "USER": os.environ.get("DJANGO_DATABASE_USER", "postgres"),
        "PASSWORD": os.environ.get("DJANGO_DATABASE_PASSWORD", "***"),
        "HOST": os.environ.get("DJANGO_DATABASE_HOST", "localhost"),
        "PORT": "5432",
        "OPTIONS": {
            "sslmode": "require",
            "sslrootcert": "DigiCertGlobalRootCA.crt.pem",
        },
    }
}
```

where your .pem is your cert you downloaded. See [here](https://learn.microsoft.com/en-us/azure/postgresql/single-server/concepts-ssl-connection-security) for where to download cert for old postgres (will be deprecated), and [here](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-connect-tls-ssl) for new postgres (flexible).

**Remember** localhost won't need ssl so do something like this:

```
if DJANGO_ENV != 'dev':
    DATABASES = {
        "default": {
            "ENGINE": "django.db.backends.postgresql_psycopg2",
            "NAME": os.environ.get("DJANGO_DATABASE_NAME", "dbname"),
            "USER": os.environ.get("DJANGO_DATABASE_USER", "postgres"),
            "PASSWORD": os.environ.get("DJANGO_DATABASE_PASSWORD", "***"),
            "HOST": os.environ.get("DJANGO_DATABASE_HOST", "localhost"),
            "PORT": "5432",
            "OPTIONS": {
                "sslmode": "require",
                "sslrootcert": "DigiCertGlobalRootCA.crt.pem",
            },
        }
    }
else:
    DATABASES = {
        "default": {
            "ENGINE": "django.db.backends.postgresql_psycopg2",
            "NAME": os.environ.get("DJANGO_DATABASE_NAME", "dbname"),
            "USER": os.environ.get("DJANGO_DATABASE_USER", "postgres"),
            "PASSWORD": os.environ.get("DJANGO_DATABASE_PASSWORD", "***"),
            "HOST": os.environ.get("DJANGO_DATABASE_HOST", "localhost"),
            "PORT": "5432",
        }
    }
```

Once you do this, if your auto-generated password doesn't have a special character, then it'll work. But if you're unlucky you need to modify your password.

#### Failing to authenticate with correct password

Auto-generated passwords is problematic and you should avoid dollar signs in the password.

Example, a password in env variable with `x$mCyu&` gets truncated to just `x` .

So make sure the password is without `$`. To debug, ssh into the `azurewebsites.net` and print out your password explicitly:

`python -c 'import os; print(os.environ.get("DJANGO_DATABASE_PASSWORD", "postgres"))'`

#### Moving Heroku db to Azure

**CAUTION:** this deletes your azure deployments' data. This assumes your azure deployment is just some test data and your heroku's your main postgres data that you'd like to preserve.

##### 1. Download your heroku db

```bash
heroku pg:backups capture --app <your app name> # backup your db and capture snapshot
heroku pg:backups:url --app <your app name> # get url of snapshot
curl -o latest.dump "<the url it gave in previous command>" # download latest.dump to your local computer
```

##### 2. Upload it to your azure db

Assuming you did the SSL correctly above and downloaded the cert, just do this:

```bash
# set env vars since pg_restore itself does not have a cert/ssl required option.
export PGSSLMODE=require
export PGSSLROOTCERT=/path/to/DigiCertGlobalRootCA.crt.pem
# destroys your Azure's db upon logging in with --clean and restores it with your latest.dump which you got from Heroku
pg_restore --verbose --clean --no-acl --no-owner -U <your azure db username> -d <your app name> -h <your azure db url> -p 5432 latest.dump
```

### Azure DevOps

As of this writing, with or without an Azure subscription, you need to request [pipeline parallelism](https://aka.ms/azpipelines-parallelism-request). All accounts come with 0 parallelism, meaning all accounts cannot run pipelines until you get it approved. Took me 1 business day but they say it can take 2-5 biz days. Something like this in `azure-pipelines.yml`

```yaml
# Python to Linux Web App on Azure
# Build your Python project and deploy it to Azure as a Linux Web App.
# Change python version to one thats appropriate for your application.
# https://docs.microsoft.com/azure/devops/pipelines/languages/python

trigger:
- main

variables:
  # Azure Resource Manager connection created during pipeline creation
  azureServiceConnectionId: 'some-number'

  # Web app name
  webAppName: 'appname'

  # Agent VM image name
  vmImageName: 'ubuntu-latest'

  # Environment name
  environmentName: 'envname'

  # Project root folder. Point to the folder containing manage.py file.
  projectRoot: $(System.DefaultWorkingDirectory)

  # Python version: 3.9
  pythonVersion: '3.9'

stages:
- stage: Build
  displayName: Build stage
  jobs:
  - job: BuildJob
    pool:
      vmImage: $(vmImageName)
    steps:
    - task: UsePythonVersion@0
      inputs:
        versionSpec: '$(pythonVersion)'
      displayName: 'Use Python $(pythonVersion)'

    - script: |
        python -m venv antenv
        source antenv/bin/activate
        python -m pip install --upgrade pip
        pip install setup
        pip install -r requirements.txt
        
      workingDirectory: $(projectRoot)
      displayName: "Install requirements"

     # Adding Node.js installation here
    - task: NodeTool@0
      inputs:
        versionSpec: '16.x'
      displayName: 'Install Node.js'
      
    - script: |
        source antenv/bin/activate
        npm install
        npm run build
      workingDirectory: $(projectRoot)
      displayName: "Install and build npm packages"
      
    - script: |
        source antenv/bin/activate
        python manage.py collectstatic --noinput
      
      workingDirectory: $(projectRoot)
      displayName: "Collect static files"

    - task: ArchiveFiles@2
      displayName: 'Archive files'
      inputs:
        rootFolderOrFile: '$(projectRoot)'
        includeRootFolder: false
        archiveType: zip
        archiveFile: $(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip
        replaceExistingArchive: true

    - upload: $(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip
      displayName: 'Upload package'
      artifact: drop

- stage: Deploy
  displayName: 'Deploy Web App'
  dependsOn: Build
  condition: succeeded()
  jobs:
  - deployment: DeploymentJob
    pool:
      vmImage: $(vmImageName)
    environment: $(environmentName)
    strategy:
      runOnce:
        deploy:
          steps:

          - task: UsePythonVersion@0
            inputs:
              versionSpec: '$(pythonVersion)'
            displayName: 'Use Python version'

          - task: AzureWebApp@1
            displayName: 'Deploy Azure Web App : appname'
            inputs:
              azureSubscription: $(azureServiceConnectionId)
              appName: $(webAppName)
              package: $(Pipeline.Workspace)/drop/$(Build.BuildId).zip
```

### Unable to sync with stripe with dj-stripe

If you use `dj-stripe` as opposed to manually do webhooks with Stripe, you need to do this. If not, you can ignore this.

Azure for some reason, adds a port to Stripe's IP when they post to your webhook. This is incompatible with how `dj-stripe` updates your database, and will cause it to fail. To fix this, you need to just create a `middleware.py` in the same directory as where your `settings.py` is like thus:

```python
class RemovePortMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # Get the remote address from the request's META info
        remote_addr = request.META.get('REMOTE_ADDR')
        xheader = request.META.get("HTTP_X_FORWARDED_FOR")
        
        # Check if the remote address contains a port
        if remote_addr and ':' in remote_addr:
             # Remove the port number
             request.META['REMOTE_ADDR'] = remote_addr.split(':')[0]
        
        if xheader and ':' in xheader:
            # Remove the port number
            request.META['HTTP_X_FORWARDED_FOR'] = xheader.split(':')[0]
        
        # Continue processing the request as usual
        response = self.get_response(request)
        return response
```

Then add this line to your `MIDDLEWARE` in `settings.py`:

`'<your app name>.middleware.RemovePortMiddleware',`

DJ-Stripe's github says stripping the port from `HTTP_X_FORWARDED_FOR` is enough, but GPT says `REMOTE_ADDR` is more robust, so I just did both.

## Conclusion

Azure is much more diffiuclt to use than Heroku.

Heroku's much better in these departments:

* Easy to do almost everything
* Quick deployment. About 3-5X faster than Azure on the same codebase
* 1-click everything

Heroku's bad in this department:

* Current Azure setup is about $26/mo. An equivalent setup in Heroku would run $80+/mo.
* Costs are even more skewed as you scale up.

So, Heroku's very easy to use and very fast to deploy.

Azure's extremely painful, confusing and you'll run into caveats in every single step of the way, even on a simple deployment.

But Heroku's way more expensive.

MVPs should use Heroku since saving a few bucks a month doesn't matter if you don't ever leave your MVP stage.

For businesses that you seriously want to scale, use Azure since server costs of say 100s of thousands a month can save you the same order of magnitude of money. And for that kind of money, it's worth the 1-time pain of setting it up.

Alternatively, if you already have a good `.yml` and a good flow and you are just using a Django template, you can just copy/paste all the above steps pretty easily and save money. Most of the pain I experienced is having to hunt down every single corner case and solve them 1-by-1 over the course of a week. But just implementing all of the above, with the solutions already in hand, might take an hour of overhead which is a really good dollar per hour if you save $40/mo (assuming you run your server for a year).
