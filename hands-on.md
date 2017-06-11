Target
---------------------------------

- A simple nodejs application
- Add mongodb to the application
- CI/CD for the application
- Logging, Monitoring, Debugging


A simple nodejs application
---------------------------------

### Introduction

### Deploy the application

- Prepare

  ```
  oc login https://10.17.3.32:8443 -u developer  # Login openshift server
  ```

  Passworld: developer

  ```
  oc new-project <your name here>    # Create new project
  oc status                          # Verify environment
  ```

  You should see:

  ```
  In project <you name here> on server https://10.17.3.32:8443

  You have no services, deployment configs, or build configs.
  Run 'oc new-app' to create an application.
  ```

- Deploy the application

```
oc new-app https://github.com/adu-21/nodejs-ex.git -l name=myapp   # Create a new application in your project
```

​	Login the Openshift console : <https://10.17.3.32:8443/console> 

​	Click your project and verify your application.

### Redeploy/Rollback the application when code changes

- Redeploy

  ```
  Hook ? Update buildConfig ?
  ```

- Rollback

  ```
  oc rollback nodejs-ex       # Perform a rollback to the last successfully completed deployment for a deploymentconfig 
  oc rollback nodejs-ex-1     # Perform a rollback to a specific deployment
  ```

### Scale up/down the deployment



### Get yourself familiar with the commands and concepts

Add mongodb to the application
---------------------------------

### Deploy a database with persistent volume

### Connect nodejs application to database

### Consider HA of the database

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







