Target
---------------------------------

- A simple nodejs application
- Add mongodb to the application
- CI/CD for the application
- Logging, Monitoring, Debugging


A simple nodejs application
---------------------------------

### Introduction

### Preparation

- fork [the demo project](https://github.com/openshift/nodejs-ex) to your own github account and get the repository url:
 
 > https://github.com/\<your-github-name\>/nodejs-ex.git
 
- clone code  

  `git clone https://github.com/\<your-github-name\>/nodejs-ex.git`
  
- clone configuration files

  `git clone https://github.com/gmlove/openshift-workshop.git`
  
  Files in this repo will be used later.
  
- log on to OpenShift web console

  In browser, navigate to `https://192.168.99.100:8443` and login with any username and password you'd like to use. OpenShift will create that account for you.

### Login
- `oc login -u <your-username>`  and input password `<your-password>`, you'll see:

 > Authentication required for https://192.168.99.100:8443 (openshift) 
 >
 > Username: <your-username> 
 > 
 > Password: <your-password>

- check status

  `oc status`
  > You don't have any projects. You can try to create a new project, by running
  >
  > oc new-project <projectname>
  
- check project
  
  `oc project`
  > No project has been set. Pass a project name to make that the default.
  
### Create project

- create a new project

  `oc new-project testproject`
  
- check the current project on which you're working

  `oc project`
  
  > Using project "testproject" on server "https://192.168.99.100:8443".
  
- check all projects

  `oc  get projects`
  
### Create applicatioin

- create a new application with your own git repository url
  
  `oc new-app https://github.com/<your-github-name>/nodejs-ex.git`
  
  > you can specify the application name with a parameter `--name <application-name>`
  
- check status

  get project status : `oc status` 
  
  get all resources: `oc get all`
 
  
### Create route

- expose service
  
  `oc expose service/nodejs-ex`

- get route url

  `oc get route` 
  
  > your route url should look like this: 
  
  > nodejs-ex-testproject.192.168.99.100.nip.io
  
- check out the url in web browser you'll see the welcome page
  
  
### Build & deploy
- build

  `oc start-build nodejs-ex`
  
    > A build will be triggered automatically once the application is created successfully.
  
- deploy  

  `oc deploy --latest dc/nodejs-ex`
  
  > A deployment will be triggered automatically when a build is finished successfully.
  
### Rollback

- modify code and commit your changes

  modify \<code-dir\>/nodejs-ex/views/index.html line 219
  
  `Welcome to your Node.js` to `This is my awesome Node.js`
  
- rebuild & redeploy

  start a new build with `oc start-build nodejs-ex`
  
  > no need to start a deployment manually here as mentioned in previous section
  
- check your changes in web browser

  you'll see the tilte of the page changed from `Welcome to your Node.js` to `This is my awesome Node.js`
  
- rollback to the last successful deployment

  `oc rollback dc/nodejs-ex` or `oc rollout undo dc/nodejs-ex`
  
  you'll get the following messages:
  
  >  Warning: the following images triggers were disabled: nodejs-ex:latest
  >
  >  You can re-enable them with: oc set triggers dc/nodejs-ex --auto
  
  The message means after a build finished, OpenShift will no longer trigger an aotumatic deployment. You can re-enable it use the command given in the message if you would like to.
  
- rollback to specified deploy

  `oc rollout undo dc/nodejs-ex --to-revision=1` or `oc rollback nodejs-ex --to-version=1`

### Scale up/down
- scale manually

  scale up to 2 pods
  
   `oc scale dc/nodejs-ex --replicas=2`
  
  now you can scale down to 1 pod
  
  `oc scale dc/nodejs-ex --replicas=1`
  
- config autoscaling

  `oc autoscale dc/nodejs-ex --min 1 --max 10 --cpu-percent=80`
  
  > You need to configure Metrics first. That is not easy, so we're not going    to do that today.


Add mongodb to the application
---------------------------------

### Deploy a database with persistent volume

- crete secret

  `oc create -f mongodb-secrets.json`
  
- checkout secret

  `oc describe secret nodejs-ex`
  
- Create persistent volume claim

  `oc create -f mongodb-pvc.json`

- Check persistent volume claim

  `oc get pvc`

- Create mongodb deployment config

  `oc create -f mongodb-deploy-config.yaml`

- Create a mongodb service

  `oc create -f mongodb-service.yaml`

- Trigger a deployment for mongodb

  `oc rollout latest dc/mongodb`
  
### Connect nodejs application to database

- Change deployment config and add environment parameters

  `oc edit dc/nodejs-ex`  and add the following lines:
  
  ```
  env:
          - name: DATABASE_SERVICE_NAME
            value: mongodb
          - name: MONGODB_USER
            valueFrom:
              secretKeyRef:
                key: database-user
                name: nodejs-mongo-persistent
          - name: MONGODB_PASSWORD
            valueFrom:
              secretKeyRef:
                key: database-password
                name: nodejs-mongo-persistent
          - name: MONGODB_DATABASE
            value: sampledb
          - name: MONGODB_ADMIN_PASSWORD
            valueFrom:
              secretKeyRef:
                key: database-admin-password
                name: nodejs-mongo-persistent
  ```
  
- Trigger a deployment for nodejs-ex

  `oc rollout latest dc/nodejs-ex`

- Check

  In you browser, open the application with url `nodejs-ex-testproject.192.168.99.100.nip.io`, and check the `Page view count` in the web page, it should be counting.



CI/CD for the application
----------------------------------

### Integrate Jenkins with openshift

### A simple pipeline

### Multi environment management

Logging, Monitoring, Debugging
----------------------------------

### Logging aggregation

### Monitoring solutions

### Debugging







