# SLS Deploy Microservices

Useful for deploying microservices AWSLambda functions in a monorepo with Serverless Framework.

## Variables

- **\*current_app:** The AWSLambda function name to deploy. **Should be the same as the folder name**
- **\*environment:** The environment to deploy to.
- **\*aws_access_key_id:** The AWS access key ID.
- **\*aws_secret_access_key:** The AWS secret access key.
- **aws_region:** The AWS region. **default: us-east-1**
- **serverless_version:** The Serverless Framework version. **default: latest**
- **use_ci:** Use npm ci. **default: "false"** (If true, you must have a package-lock.json file in the root of the project.)


**You can also add your own environment variables to be used in the deployment by Serverless Framework.**
```yml
environment:
  MY_ENV_VAR: "my value"
```

## Usage

```yml
- name: Deploy
  uses: Pubnic/sls-deploy-microservices@v1
  with:
    current_app: users
    environment: production
    aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws_region: ${{ secrets.AWS_DEFAULT_REGION }}
    serverless_version: 3.19.0
    use_ci: "true"
  env:
    LISTENER_ARN: ${{ secrets.LISTENER_ARN }}
```

### Lambda Function Final Name
- current_app: users
- environment: production


**Final Name:** `users-pubnic-production`

## serverless.yml example with alb
```yml
service: pubnic
provider:
  name: aws
  stackName: ${env:CURRENT_APP}-pubnic-${env:ENVIRONMENT}
  runtime: python3.9
  region: sa-east-1
  memorySize: 128
  timeout: 30

functions:
  microservices:
    name: ${env:CURRENT_APP}-pubnic-${env:ENVIRONMENT}
    handler: app.handler
    layers:
      - Ref: PythonRequirementsLambdaLayer
    environment:
      PYTHONPATH: /src
      ENVIRONMENT: ${env:ENVIRONMENT}
      CURRENT_APP: ${env:CURRENT_APP}
    events:
      - alb:
          listenerArn: ${env:LISTENER_ARN}
          priority: ${self:custom.albPriority.${env:CURRENT_APP}}
          multiValueHeaders: true
          conditions:
            host: api.pubnic.app
            path: /${env:CURRENT_APP}/*

package:
  patterns:
    - '!**'
    - 'src/common/**'
    - 'src/${env:CURRENT_APP}/**'

custom:
  albPriority:
    users: 1
  pythonRequirements:
    slim: true
    strip: false
    dockerizePip: non-linux
    dockerImage: public.ecr.aws/sam/build-python3.9:latest
    layer:
      name: ${sls:stage}-pubnic
      description: Pubnic Dependencies
      compatibleRuntimes:
        - python3.9
    noDeploy:
      - appnope
      - asgiref
      - backcall
      - click
      - decorator
      - flake8
      - h11
      - httptools
      - iniconfig
      - ipython
      - ipython-genutils
      - jedi
      - matplotlib-inline
      - mccabe
      - packaging
      - parso
      - pexpect
      - pickleshare
      - pluggy
      - prompt-toolkit
      - ptyprocess
      - py
      - pycodestyle
      - pyflakes
      - Pygments
      - pyparsing
      - pytest
      - pytest-mock
      - PyYAML
      - toml
      - traitlets
      - uvicorn
      - uvloop
      - watchgod
      - wcwidth
      - websockets

plugins:
  - serverless-python-requirements
```

## FastAPI example

### APP
```python
from fastapi import FastAPI
from mangum import Mangum
from routers import router


app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=['*'],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(router)

handler = Mangum(app, lifespan='on')
```

### Users Function

You can have one app with multiples AWSLambda functions.

- path: `src/users/views.py`

```python
from fastapi import APIRouter


users_router = APIRouter()


@users_router.get('/status/')
async def status():
    return {'status': 'OK'}
```

- path: `src/routers.py`

```python
import os
from fastapi import APIRouter
from enum import Enum

ENVIRONMENT = os.getenv('ENVIRONMENT', 'development')
CURRENT_APP = os.getenv('CURRENT_APP')


router = APIRouter()


class AppNameSet(str, Enum):
    USERS = 'users'  # Should be the same as the folder name


def can_load_app(app_name: AppNameSet):
    return (
        ENVIRONMENT == 'development' or
        app_name == CURRENT_APP
    )

# Here you can add your own routes
if can_load_app(AppNameSet.USERS):
    from users.views import users_router
    router.include_router(users_router, prefix='/users')
```
