# Week 4 — Postgres and RDS
We are going to us postgres and DynamoDB local, in future labs we can bring them in as containers and refrence them externally
Let's intergrate them into our docker compose file:

## Postgres
```yml
services:
  db:
    image: postgres:13-alpine
    restart: always
    enviroment:
      -  POSTGRES_USER=postgres
      -  POSTGRES_PASSWORD=password
    ports:
      -  '5432:5432'
    volumes:
      -  db:/var/lib/postgresql/data
  volumes:
    db:
      driver: local
```

## DynamoDB Local
```yml
services:
  dynamodb-local:
    # we need to add user:root to get this working
    user:root
    command: " -jar DynamoDBLocal.jar -sharedDB -dbPath ./data"
    image: "amazon/dynamodb-local:latest"
    container_name: dynamodb-local
    ports:
      - "8000:8000"
    volumes:
      -  "./docker/dynamodb:/home/dynamodblocal/data"
    working_dir: /home/dynamodblocal
```

## Volumes 
directory volume mapping 
```yml
volumes:
  -  "./docker/dynamodb:/home/dynamodylocal/data"
```
named volume mapping 




# Distributed Tracing 

## Honeycomb

When creating a new dateset in honeycomb it will provide all this  installation instructions 
Add the following lines to the requirements.txt file [requirements.txt file](../backend-flask/requirements.txt)

```txt
flask
flask-cors
opentelemetry-api
opentelemetry-sdk
opentelemetry-exporter-otlp-proto-http
opentelemetry-instrumentation-flask
opentelemetry-instrumentation-requests
```
Install this dependencies. 

navigate to the backend-flask directory 

run the command: 
```sh
pip install -r requirements.txt
```
add to the app.py

```py
# app.py updates 
from opentelemetry import trace 
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TraceProvider 
from opentelemetry.sdk.trace.export import BatchSpanProcessor


# Initialize tracing and an exporter that can send data to Honeycomb
provider = TraceProvider()
processor = BatchSpanProcessor(OTLPSpanExporter())
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)
tracer = trace.get_tracer(__name__)

# Initialize automatic instrumention with Flask
app = Flask(__name__)
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()
```


A third-pary tool that serve this purpose is honeycomb. 
- Install create a honeycomb account
- Log into honeycomb
- create a new enviroment [screenshot, creating a new env in honeycomb](./screenshot.md)
- open the new enviroment and copy the api key
- in gitpod run the following code to export and the api key to gitpod env

### Note:
for security purposes it's best practice for a multiple services running on the backend, to hard-code this in the docker-compose file for the different services.
  
```sh
export HONEYCOMB_API_KEY="CArsB0ZoEIeets9QJcR8dM"
gp env HONEYCOMB_API_KEY="CArsB0ZoEIeets9QJcR8dM"
```
## Add the following env to the docker compose file 
for the backend service 
```Dockerfile
ervices:
  backend-flask:
    environment:
      FRONTEND_URL: "https://3000-${GITPOD_WORKSPACE_ID}.${GITPOD_WORKSPACE_CLUSTER_HOST}"
      BACKEND_URL: "https://4567-${GITPOD_WORKSPACE_ID}.${GITPOD_WORKSPACE_CLUSTER_HOST}"
      OTEL_SERVICE_NAME: "backend-flask"                                              #added line
      OTEL_EXPORTER_OTLP_ENDPOINT: "https://api.honeycomb.io"                         #added line
      OTEL_EXPORTER_OTLP_HEADERS: "x-honeycomb-team=$(HONEYCOMB_API_KEY)"             #added line 
    build: ./backend-flask
    ports:
    -  "4567:4567"
    volumes:
    -  ./backend-flask:/backend-flask
```


# Instrument XRay

AWS-X-Ray provides API's for managining debug traces and retrieving sevices maps and 

other data created by processing those traces.

Export and configure the env for gp

```sh
export AWS_REGION="us-east-1"
gp env AWS_REGION="us-east-1"
```
Add to the requirements.txt file 

```sh
aws-xray-sdk
```
Install the dependencies

```sh
pip install -r requirements.txt
```
add to app.py

```python
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.ext.flask.middleware import XRayMiddleware

xray_url=os.getenv("AWS_XRAY_URL")
xray_recorder.configure(service='backend-flask', dynamic_naming=xray_url)
XRayMiddleware(app, xray_recorder)
```
## Setup AWS Xray Resources

add `aws/json/aws.json`

```json
{
  "SamplingRule": {
      "RuleName":  "Cruddur",
      "ResourceARN":  "*",
      "Priority":  9000,
      "FixedRate":  0.1,
      "ReservoirSize":  5,
      "SercieName":  "backend-flask",
      "SercieType":  "*",
      "Host":  "*",
      "HTTPMethod":  "*",
      "URLPath":  "*",
      "Version":  1
  }
}
```

run the command to below to create x-ray group in order to group the traces. Note this is not important

cd > cd backend-flask

```sh
FLASK_ADDRESS="https://4567-${GITPOD_WORKSPACE_ID}.${GITPOD_WORKSPACE_CLUSTER_HOST}"
aws xray create-group \
    --group-name "Cruddur" \
    --filter-expression "service(\"FLASK_ADDRESS\"){ault OR error}"
```
create a sampling rule 

run the command below to create a sampling rule for the above sampling rule json file 

```sh
aws xray create-sampling-rule --cli-input-json file://aws/json/xray.json
```
# Install X-ray Daemon

Github aws-xray-daemon X-Ray Docker Compose example 

To install the daemon onto our enviroment the run the below command 

```sh
wget https://s3.us.east-2.amazonaws.com/aws-xray-assets.us-east-2/xray-daemon/aws-xray-daemon-3.x.deb
sudo dpkg -i **.deb
```

It will be preferable to run daemon as a container in docker compose; use the docker compose file 

## Add Deamon Service to Docker Compose 

```Dockerfile
xray-daaemon:
  image: "amazon/aws-xray-daemon"
  environment:
    AWS_ACCESS_KEY_ID: "$(AWS_ACCESS_KEY_ID)"
    AWS_SECRET_ACCESS_KEY: "$(AWS_SECRET_ACCESS_KEY)"
    AWS_REGION: "us-east-2"
  command:
    - "xray -o -b xray-daemon:2000"
  ports:
    - 2000:2000/udp

```
## add these two env cars to our backend-flask in our docker-compose.yml file

```Dockerfile
AWS_XRAY_URL: "*4567-${GITPOD_WORKSPACE_ID}.${GITPOD_WORKSPACE_CLUSTER_HOST}*"
AWS_XRAY_DAEMON_ADDRESS: "xray-daemon:2000"
```


## Cloud Watch Logs

add to requirements.txt 

`watchtower`

```sh
pip install -r requirements.txt
```
in `appl.py`

```sh
import watchtower
import logging
from time import strftime
```
```sh
# configure logger to use cloudwatch log

# # Configure logger to use CloudWatch Logs
LOGGER = logging.getLogger(__name__)
LOGGER.setLevel(logging.DEBUG)

# Create handlers
console_handler = logging.StreamHandler()  # Assuming you want to log to the console as well
cw_handler = watchtower.CloudWatchLogHandler(log_group='cruddur')

# Add handlers to the logger
LOGGER.addHandler(console_handler)
LOGGER.addHandler(cw_handler)
```

```sh
@app.after_request
def after_request(response):
  timestamp = strftime('[%Y-%b-%d %H:%M]')
  LOGGER.error('%s %s %s %s %s %s', timestamp, request.remote_addr, request.method, request.scheme, request.full_path, response.status)
  return response
```

we'll log something in API endpoint

`LOGGER.info('Hello api cloudwatch! from /api/activities/home')`

set the env var in backend-flask for `docker-compose.yml`

```dockerfile
AWS_DEFAULT_REGION: "${AWS_DEFAULT_REGION}"
AWS_ACCESS_KEY_ID: "${AWS_ACCESS_KEY_ID}"
AWS_SECRET_ACCESS_KEY: "${AWS_SECRET_ACCESS_KEY}"
```
`cd front-end-react-js`

run `nmp i` 

`cd backend-flask`

Run the docker-compose up 



## Rollbar 

https://rollbar.com

Create a new project in rollbar called cruddur

pyrollbar requires both blinker and rollbar to be installed

Add to `requirements.txt`

```
blinker
rollbar
```
install deps

```sh
cd backend-flask
```
run the command below

`pip install -r requirements.txt`

Next set the access token 

```
export ROLLBAR_ACCESS_TOKEN=""
gp env ROLLBAR_ACCESS_TOKEN=""
```
The above set export the rollbar access token in gitpod 

Next add to backend-flask for `docker-compose.yml`

`ROLLBAR_ACCESS_TOKEN: "${ROLLBAR_ACCESS_TOKEN}"`

import to Rollbar

```Dockerfile
import os
import rollbar
import rollbar.contrib.flask
from flask import got_request_exception

```

```Dockerfile 
rollbar_access_token = os.getenv('ROLLBAR_ACCESS_TOKEN')
@app.before_first_request
def init_rollbar():
    """init rollbar module"""
    rollbar.init(
        # access token
        rollbar_access_token,
        # environment name
        'production',
        # server root directory, makes tracebacks prettier
        root=os.path.dirname(os.path.realpath(__file__)),
        # flask already sets up logging
        allow_logging_basic_config=False)

    # send exceptions from `app` to rollbar, using flask's signal system.
    got_request_exception.connect(rollbar.contrib.flask.report_exception, app)

```

Adding and endpoint just for testing app.py

```python
@app.route('/rollbar/test')
def rollbar_test():
  rollbar.report_message('Hello world!', 'warning')
  return "Hello world!"

```












