This guide was written with:

| Software | Version  | 
|----------|----------|
| Noop     | v2.0.110 |
| Python   | v3.12.0  |
| Django   | v4.2.7   |

### Project Bootstrap

Download and install [Python](https://wiki.python.org/moin/BeginnersGuide/Download).

Create and activate a Python virtual environment

```bash
python -m venv venv
source venv/bin/activate
```

Install Django and other requirements from this repo's `requirements.txt`

```bash
pip install -r requirements.txt
```

Bootstrap a Django project

```bash
django-admin startproject noop_example
```

[Install Noop Desktop](https://noop.dev/docs/installation/)

[Create a local project](https://noop.dev/docs/local-development/), selecting the Django project directory

Create a Noop config file at `.noop/app.yml`

```bash
mkdir .noop
touch .noop/app.yml
```

Edit `.noop/app.yml` to include this basic Django configuration

```yaml
components:
  - name: Django
    type: service
    image: python:3.12.0-slim
    port: 8000
    build:
      steps:
        - directory: /app
        - copy: requirements.txt
          destination: ./
        - copy: noop_example
          destination: noop_example
        - run: pip3 install -r requirements.txt --no-cache-dir
    runtime:
      command: python3 noop_example/manage.py runserver 0.0.0.0:8000
      resources:
        - DjangoDB
      variables:
        DB_NAME:
          $resources: DjangoDB.database
        DB_USER:
          $resources: DjangoDB.username
        DB_PASS:
          $resources: DjangoDB.password
        DB_HOST:
          $resources: DjangoDB.host
        DB_PORT:
          $resources: DjangoDB.port
  - name: DjangoMigrate
    type: task
    image: python:3.12.0-slim
    build:
      steps:
        - directory: /app
        - copy: requirements.txt
          destination: ./
        - copy: noop_example
          destination: noop_example
        - run: pip3 install -r requirements.txt --no-cache-dir
    runtime:
      command: python3 noop_example/manage.py migrate
      resources:
        - DjangoDB
      variables:
        DB_NAME:
          $resources: DjangoDB.database
        DB_USER:
          $resources: DjangoDB.username
        DB_PASS:
          $resources: DjangoDB.password
        DB_HOST:
          $resources: DjangoDB.host
        DB_PORT:
          $resources: DjangoDB.port
lifecycles:
  - event: BeforeServices
    components:
    - DjangoMigrate
resources:
  - name: DjangoDB
    type: postgresql
routes:
  - target:
      component: Django

```

Update `noop_example/noop_example/settings.py` to read the database bindings from the environment variables defined in `.noop/app.yml` and allow noop hostnames.

```python
import environ

env = environ.Env()
environ.Env.read_env()

# ...

ALLOWED_HOSTS = ['.noop.app']

# ...

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASS'),
        'HOST': env('DB_HOST'),
        'PORT': env('DB_PORT')
    }
}

# ...
```

In order to automate most of the Application setup, including Environment creation, database provisioning, building and deploying, etc., we're going to create a Runbook. Create a file called `quickstart.yml` in the `.noop/runbooks` directory.

Here is the Runbook content:

```yaml
name: Quickstart Setup

description: Demo full stack setup of this Django app with internet endpoint

workflow:
  inputs:
    - name: EnvironmentName
      description: What should we name your new environment?
      type: string
      required: true
      default: Quickstart Demo

    - name: Cluster
      type: Cluster
      description: Which cluster should the environment launch on?
      required: true

  timeout: 300

  steps:
    - name: Environment
      action: EnvironmentCreate
      params:
        name:
          $inputs: EnvironmentName
        production: false
        appId:
          $runbook: Application.id
        clusterId:
          $inputs: Cluster.id

    - name: Build
      action: BuildExecute
      params:
        sourceCodeId:
          $runbook: SourceCode.id
        appId:
          $runbook: Application.id

    - name: Resources
      action: ResourceLaunch
      params:
        envId:
          $steps: Environment.id
        sourceCodeId:
          $runbook: SourceCode.id

    - name: Deploy
      action: DeploymentExecute
      params:
        envId:
          $steps: Environment.id
        buildId:
          $steps: Build.id

    - name: Endpoint
      action: InternetEndpointRandom
      params:
        orgId:
          $runbook: Organization.id
        routes:
          - name: 'Demo Environment'
            target:
              environments:
                - $steps: Environment.id

```
