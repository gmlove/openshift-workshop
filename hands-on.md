Target
---------------------------------

- A simple nodejs application
- Add mongodb to the application
- CI/CD for the application
- Logging, Monitoring, Debugging


A simple nodejs application
---------------------------------

### Introduction

In this section, We are going to create a nodejs project with mongodb in OpenShift. We assume that you have done all the preparation work listed in the invitation email of this workshop. And there're some additional steps to  get yourself ready.

### Preparation

- fork [the demo project](https://github.com/openshift/nodejs-ex) to your own github account and get the repository url:
 
 > https://github.com/<your-github-name>/nodejs-ex.git
 
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

  `oc create -f mongodb-secrets.yaml`
  
- checkout secret

  `oc describe secret nodejs-ex`
  
- Create persistent volume claim

  `oc create -f mongodb-pvc.yaml`

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

**Several ways to integrate pipeline:**

- Keep the original pipeline and artifact repository. Manage your pipeline outside of openshift world. When you need to deploy to an environment, you can trigger a build in openshift to grab your artifact and build an image and deploy it to openshift.
- Use openshift integrated Jenkins as your pipeline. (heavily customized, with openshift plugin and k8s plugin installed.)

### Integrate Jenkins with the application

- Create a pipeline build config

  Create a file named `nodejs-ex-pipeline.yaml` with content:

  ```
  kind: "BuildConfig"
  apiVersion: "v1"
  metadata:
    name: "nodejs-ex-pipeline"
  spec:
    strategy:
      jenkinsPipelineStrategy:
        jenkinsfile: "node('nodejs') {\n  stage 'build'\n  openshiftBuild(buildConfig: 'nodejs-ex', showBuildLogs: 'true')\n  stage 'deploy'\n  openshiftDeploy(deploymentConfig: 'nodejs-ex')\n}\n"
  ```

  Run command `oc create -f nodejs-ex-pipeline.yaml`. Openshift will add one route `routes/jenkins` and two services `svc/jenkins` `svc/jenkins` to the project.

- Run `oc status` to get your dns of the jenkins application. Then you can open it in your browser and do an oauth login with openshift credentials.

- Trigger a build manually and you can check logs from jenkins. You can also find what is openshift doing by `watch oc get all`

  + Openshift creates a slave based on the node being used, here it's `nodejs`

  + Run the build in that slave

  + The build will simply trigger a build, works just like `oc start-build bc/nodejs-ex`, and then trigger a deployment


### Multi environment management

In openshift you define several resources (API Objects) and you can start an environment. So multi environment means multi similar resources sets. To manage multi environment, you have many options, such as:

- Via labels and unique naming within a single project
- Via distinct projects within a cluster
- Via distinct clusters

Consider your situation in your organization and choose one properly. We'll try the second one as an example. We'll develop a way to share the image created in the project (can be called as dev project).

- Spend some time thinking about which resources you will need for another environment (ImageStream DeploymentConfig Service Route PersistentVolumeClaim)

- Tag all the resources to export by

  ```
  oc label dc/mongodb promotion=nodejs-ex
  oc label dc/nodejs-ex promotion=nodejs-ex
  oc label svc/nodejs-ex promotion=nodejs-ex
  oc label svc/mongodb promotion=nodejs-ex
  oc label routes/nodejs-ex promotion=nodejs-ex
  oc label pvc/mongodb promotion=nodejs-ex
  oc label is/nodejs-ex promotion=nodejs-ex
  oc label secret/nodejs-ex promotion=nodejs-ex
  ```

- Export those resources

  `oc export dc,svc,routes,pvc,is,secret -l promotion=nodejs-ex -o yaml > exported-for-promotion.yaml`

- Open the exported file do the below:

  + Remove image hash tag
  + Replace all string `test1` to `test1-sys` and we're creating a project named `test1-sys`
  + Remove `annotations` `volumeName` `status` `creationTimestamp` from `PersistentVolumeClaim`

- Create a new project by `oc new-project test1-sys`

- Create resources by `oc create -f exported-for-promotion.yaml`

- Tag an image from test1
  
  `oc tag test1/nodejs-ex:latest nodejs-ex:latest`

- After that you can trigger a deployment, and after it succeeded, you will be able to open the application in `sys` environment

- Try create a template for these resources and then you can create any new environment with one command


Logging, Monitoring, Debugging
----------------------------------

### Logging aggregation

- EFK solution, check a video [here](https://www.youtube.com/watch?v=RMDX3YC0CSQ)

### Monitoring solutions

- Prometheus/zabbix/hawk: https://github.com/openshift/openshift-tools/tree/stg/openshift_tools/monitoring
- dynatrace/coscale/sysdig/appdynamics

### Debugging

- To monitor the change of resources: `watch oc get all`
- Check logs of pods: `oc logs po/nodejs-ex-1-0s5sl`
- Check events: `oc get event`
- Login to container: `oc rsh po/nodejs-ex-1-0s5sl`




