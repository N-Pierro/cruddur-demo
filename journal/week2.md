# Week 2 — Distributed Tracing

## Container Service 

### Run python command 

```sh  
cd backend-flask
export FRONTEND_URL="*"
export BACKEND_URL="*"
python -m flask run --host=0.0.0.0 --port=4567
cd ..
```

- make sure to unlock the lock in the port tree
- copy the url to browser
- append to the url /api/activities/home
- you should now see the backend json file 


### ADD DOCKERFILE
```Dockerfile
FROM python:3.10-slim-buster

WORKDIR /backend-flask

COPY requirements.txt requirements.txt

RUN pip3 install -r requirements.txt

COPY . .

ENV FLASK_ENV=development 
EXPOSE ${PORT}
CMD ["python3", "-m", "flask", "run", "--host=0.0.0.0", "--port=4567"]
```
### BUILD CONTAINER 

```
docker build -t backend-flask ./backend-flask
```

### RUN CONRAINER 
run

```sh
# -e flag is used to pass enviromental variable 
docker run --rm -p 4567:4567 -it -e BACKEND_URL='*' -e FRONTEND_URL='*'  backend-flask
```
run in background mode 

```sh
docker run --rm -p 4567:4567 -d -e BACKEND_URL='*' -e FRONTEND_URL='*'  backend-flask
```
### Get the container images or running container ids
```sh
docker ps
docker images
```
### Send Curl to Test Server 
```
curl -X GET http://localhost:4567/api/activities/home -H "Accept: application/json" -H "Content-Type: application/json"
```
## Containerized Frontend

### Run NPM install

we have to run npm install before building the container since it needs to copy contents of the node_module 
```sh
cd frontend-react-js
npm i
```

### Create the Dockerfile

```Dockerfile
FROM node:16.18

ENV PORT=3000

COPY . /frontend-react-js
WORKDIR /frontend-react-js
RUN npm install
EXPOSE ${PORT}
CMD ["npm", "start"]
```

## Build Container 

```sh
docker build -t frontend-react-js ./frontend-react-js
```
## Run Container

```sh
docker run -p 3000:3000 -d frontend-react-js
```

## Multiple Containers
### Create a docker-compose file 
Wih docker-composer we have more than one service running and working together locally 
-  In this case we have two services; Frontend-react-js and backend-flask service
  
create a docker-compose.yml file at the root of your project directory
```Dockerfile
version: "3.8"
services:
  backend-flask:
    environment:
      FRONTEND_URL: "https://3000-${GITPOD_WORKSTATION_ID}.${GITPOD_WORKSTATION_CLUSTER_HOST}"
      BACKEND_URL: "https://4567-${GITPOD_WORKSTATION_ID}.${GITPOD_WORKSTATION_CLUSTER_HOST}"
    build: ./backend-flask
    ports:
    -  "4567:4567"
    volumes:
    -  ./backend-flask:/backend-flask
  frontend-react-js:
    environment:
      REACT_APP_BACKEND_URL: "https://4567-${GITPOD_WORKSTATION_ID}.${GITPOD_WORKSTATION_CLUSTER_HOST}"
    build: ./frontend-react-js
    ports:
    -  "3000:3000"
    volumes:
    -  ./frontend-react-js:/frontend-react-js
# The name flag is a hack to change the default prepend folder
# name when outputting the image name
networks:
  internal-network:
    driver: bridge
    name: cruddur
```
either run the command
```sh
docker-compose up
```
or if using vscode, locate the docker-compose file and right-click on it, then select the option "compose up"

## Futher Reading
## Images and Containers:
-  ### Image:
        A lightweight, standalone, and executable package that includes everything needed to run a piece of software.
-  ### Container:
         An instance of a Docker image. It runs a software application with all dependencies isolated from the host system.
   ## Docker Basic Commands:
   Pull an Image:
   ```sh
   docker pull IMAGE_NAME[:TAG]
   ```
   List Downloaded Images:
   ```sh
   docker images
   ```
   Run a Container:
   ```sh
   docker run IMAGE_NAME
   ```
  List Running Containers:
  ```sh
  docker ps
  ```
  List all running containers including stopped conatiners:
  ```sh
  docker ps -a
  ```
  Stop a running container and remove a container:
  ```sh
  docker stop CONTAINER_ID
  docker rm CONTAINER_ID
 ```
  Remove and image:
  ```sh
  docker rmi IMAGE_ID
  ```
## Docker Compose
A tool for defining and running multi-Docker applications 
## Docker Compose Commands
Create and start containers from docker-compose.yml:
```sh
docker-compose up
```
Start comtainer in the Background:
```sh
docker-compose up -d
```
## Must know commands
Build and image from a Dockerfile:
```sh
docker build -t IMAGE_NAME[:TAG] PATH_TO_DOCKERFILE
```
Run a command in a running container:
```sh
docker exec -it CONTAINER_ID COMMAND
```
View container Logs:
```sh
docker logs CONTAINER_ID
```
Inspect container detials:
```sh
docker inspect CONTAINER_ID
```
Prune unused resources:
```sh
docker system prune
```
## Create Notification feature 
from the project directory, first cd into the front-end-react-js

## Install openapi(swagger) to work with api end-points
- install openapi in vscode
- navigate to the openapi-3.yml file
- select the openapi icon on vscode to view the paths
- right-click on the /api/activities end-point to add a new operation
- name the new operation /api/activities/notification
- safe and commit the changes

## edited the app.py file 
duplicated the /api/activities/home 
changed the duplicate to ../notifications
```python
@app.route("/api/activities/home", methods=['GET'])
def data_home():
  data = HomeActivities.run()
  return data, 200

@app.route("/api/activities/notifications", methods=['GET'])
def data_home():
  data = NotificationsActivities.run()
  return data, 200
```
added the notifications_activities file to the service folder 
added "from services.notifications_activity import *" to the app.py file, represents the particular service for the notifications service created; each service represents and endpoint that can be called and hence very useful when running the applcation as a microservice.

  
```sh
cd front-end-react-js
```
run npm i command 
```sh
npm i
```
Note the default confirmation code for the email verification is 1234, this code is hard-coded

## implementing front-end notification page 
Open the frontend-react-js
- edit the app.js file to add the notification for the front-end application
- add the this line 

```javascript
import './App.css';

import HomeFeedPage from './pages/HomeFeedPage';
import NotificationsFeedPage from './pages/NotificationsFeedPage'; // new line

// add the route to this new end-point bellow
{
    path: "/notifications",
    element: <NotificationsFeedPage />
  },
```
Create 2 new files in the pages folder for the notifications
- NotificationsFeedPage.js
- NotificationsFeedPage.css
  
Copy contents of HomeFeedPages.js into the NotificationsFeedPages.js and make the following edits

```javascript
import './NotificationsFeedPage.css'; // change Home to NotificationsFeedPages
import React from "react";

import DesktopNavigation  from '../components/DesktopNavigation';
import DesktopSidebar     from '../components/DesktopSidebar';
import ActivityFeed from '../components/ActivityFeed';
import ActivityForm from '../components/ActivityForm';
import ReplyForm from '../components/ReplyForm';

// [TODO] Authenication
import Cookies from 'js-cookie'

export default function HomeFeedPage() {
  const [activities, setActivities] = React.useState([]);
  const [popped, setPopped] = React.useState(false);
  const [poppedReply, setPoppedReply] = React.useState(false);
  const [replyActivity, setReplyActivity] = React.useState({});
  const [user, setUser] = React.useState(null);
  const dataFetchedRef = React.useRef(false);

  const loadData = async () => {
    try {
      const backend_url = `${process.env.REACT_APP_BACKEND_URL}/api/activities/notifications` // change /home to /notifications
      const res = await fetch(backend_url, {
        method: "GET"
      });
      let resJson = await res.json();
      if (res.status === 200) {
        setActivities(resJson)
      } else {
        console.log(res)
      }
    } catch (err) {
      console.log(err);
    }
  };

  const checkAuth = async () => {
    console.log('checkAuth')
    // [TODO] Authenication
    if (Cookies.get('user.logged_in')) {
      setUser({
        display_name: Cookies.get('user.name'),
        handle: Cookies.get('user.username')
      })
    }
  };

  React.useEffect(()=>{
    //prevents double call
    if (dataFetchedRef.current) return;
    dataFetchedRef.current = true;

    loadData();
    checkAuth();
  }, [])

  return (
    <article>
      <DesktopNavigation user={user} active={'notifications'} setPopped={setPopped} />
      <div className='content'>
        <ActivityForm  
          popped={popped}
          setPopped={setPopped} 
          setActivities={setActivities} 
        />
        <ReplyForm 
          activity={replyActivity} 
          popped={poppedReply} 
          setPopped={setPoppedReply} 
          setActivities={setActivities} 
          activities={activities} 
        />
        <ActivityFeed 
          title="Notifications" 
          setReplyActivity={setReplyActivity} 
          setPopped={setPoppedReply} 
          activities={activities} 
        />
      </div>
      <DesktopSidebar user={user} />
    </article>
  );
}
```
