A simple nodejs application
---------------------------------

### Introduction

- Setup a mini cluster: `oc cluster up`
- Shutdown a mini cluster: `oc cluster down`
- Install openshift client: `brew install socat openshift-cli`
- Still have problems? Use the [created mini cluster](https://10.17.3.32:8443)
- What is the application?

### Deploy the application through webconsole

- Login (create user automatically)
- Create a simple nodejs application from webconsole (under project `test`)
- Explore and explain every page: Overview/Application/Deployments/Pods...

### Deploy the application through commandline

- Login by `oc login -u test` using the same password you just login from webconsole
- List projects by `oc projects`, you'll see the project you created through webconsole
- Fork [the project](https://github.com/openshift/nodejs-ex) to your own github account
- Create a new project: `oc new-project test1`
- Create a new application: `oc new-app https://github.com/<your-github-name>/nodejs-ex`
- Check your project status by `oc status` and all resources by `oc get all`
- Switch to previous project and compare the resources: `oc project test && oc get all && oc project test1 && oc get all`
- Create a route to make the new application available to outside: `oc expose service nodejs-ex`
- Check your application by opening url `http://nodejs-ex-test1.192.168.1.5.xip.io/` in your browser
- What it does?
    + Autodetect what kind of components you have provided
    + Set up a build: source code -> image
    + Deploy via a deployment configuration
    + Create a service connected to the first public port of the app
    + Trigger a new build automatically

### Build/Deploy/Rollback the application when code changes

- Clone the project and made some changes, then push to github
- Run build manually (since github cannot notify your local server) by `oc start-build bc/nodejs-ex`, then check running build by `watch oc status` or from webconsole
- Deployment will be triggered automatically. Open the application `http://nodejs-ex-test1.<your-ip>.xip.io/` in your browser and you should see `version: V2` contained in the title
- Rollback to version 1 by `oc rollback dc/nodejs-ex`
    + Update the deployment configuration `Latest Version` from `3` to `2`
    + And disable images triggers `nodejs-ex:latest`
- Open the application `http://nodejs-ex-test1.<your-ip>.xip.io/` in your browser and you should not see `version: V2` any more.
- Resume the automated deployment by `oc set triggers dc/nodejs-ex --auto`
- Trigger a latest build by `oc rollout latest`

### Scale up/down your deployment

- Scale up your deployment by `oc scale dc/nodejs-ex --replicas=2`
- Scale down your deployment by `oc scale dc/nodejs-ex --replicas=1`

### Export all the current resources

- Export the resources by `oc export bc,is,dc,svc,routes -o yaml > oc-exported.yaml`
- Explore/explain all the resources we manage
    + Most oc commands are ways to help simplify changing the resources
    + Actually we can achieve all these by simple `curl` command


Add mongodb to the application
---------------------------------

### Deploy a database with persistent volume

Don't need application to be a 12-factor application. Any application can be migrated into openshift in theory.

**Through webconsole:**

- Open webconsole and switch to `test` project
- Click `add to project` to add mongodb-persistent to this project (remember to input a value for username/password/admin_password)

**Through command line:**

- Create secrets by `oc create -f mongodb-secrets.json`
- Check created secrets by `oc get secrets` and `oc describe secret nodejs-ex`
- Create persistent volumn claim by `oc create -f mongodb-pvc.json` and check by `oc get pvc`
- Create mongodb deployment config by `oc create -f mongodb-deploy-config.yaml`
- Create a mongodb service by `oc create -f mongodb-service.yaml`
- Trigger a deployment for mongodb by `oc rollout latest dc/mongodb`

### Connect nodejs application to database

**Through webconsole:**

- Open node-ex deployment config and add four environment `MONGODB_USER` `MONGODB_PASSWORD` `MONGODB_DATABASE` `MONGODB_ADMIN_PASSWORD` with the values you just input, and add environment `DATABASE_SERVICE_NAME=mongodb`. It will trigger a deployment automatically.
- Open the application `http://nodejs-ex-test1.<your-ip>.xip.io/` in your browser and you should see `Page view count` is counting

**Through command line:**

- Change deployment config and add environment parameters by `oc edit dc/nodejs-ex`
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
- Trigger a deployment for nodejs-ex by `oc rollout latest dc/nodejs-ex`
- Open the application `http://nodejs-ex-test1.<your-ip>.xip.io/` in your browser and you should see `Page view count` is counting

### Using a template to define all the things

- Explore the template [here](https://github.com/gmlove/nodejs-ex/blob/master/openshift/templates/nodejs-mongodb-persistent.json)
- Create the whole app by `oc new-app -f nodejs-ex/openshift/templates/nodejs-mongodb-persistent.json -p SOURCE_REPOSITORY_URL=https://github.com/<your-github-name>/nodejs-ex.git`

### Consider HA of the database

- One master with multi slave / multi master


CI/CD for the application
----------------------------------

**Several ways to integrate pipeline:**

- Keep the original pipeline and artifact repository. Manage your pipeline outside of openshift world. When you need to deploy to an environment, you can trigger a build in openshift to grab your artifact and build an image and deploy it to openshift.
- Use openshift integrated Jenkins as your pipeline. (heavily customized, with openshift plugin and k8s plugin installed.)

### Integrate Jenkins with the application

- Create a pipeline build config by `oc create -f nodejs-ex-pipeline.yaml`. Openshift will add one route `routes/jenkins` and two services `svc/jenkins` `svc/jenkins` to the project.
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
- Export those resources by `oc export dc,svc,routes,pvc,is,secret -l promotion=nodejs-ex -o yaml > exported-for-promotion.yaml`
- Open the exported file do the below:
    + Remove image hash tag
    + Replace all string `test1` to `test1-sys` and we're creating a project named `test1-sys`
    + Remove `annotations` `volumeName` `status` `creationTimestamp` from `PersistentVolumeClaim`
- Create a new project by `oc new-project test1-sys`
- Create resources by `oc create -f exported-for-promotion.yaml`
- Tag an image from test1 by `oc tag test1/nodejs-ex:latest nodejs-ex:latest`
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


