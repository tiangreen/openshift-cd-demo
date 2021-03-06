*For other versions of OpenShift, follow the instructions in the corresponding branch e.g. ocp-3.7, origin-3.7, etc*

# CI/CD Demo - OpenShift Container Platform 3.7

This repository includes the infrastructure and pipeline definition for continuous delivery using Jenkins, Nexus, SonarQube and Eclipse Che on OpenShift. 

* [Introduction](#introduction)
* [Prerequisites](#prerequisites)
* [Deploy on RHPDS](#deploy-on-rhpds)
* [Deploy on OpenShift (script)](#deploy-on-openshift-script)
* [Deploy on OpenShift (manual)](#deploy-on-openshift-manual)
* [Troubleshooting](#troubleshooting)
* [Demo Guide](#demo-guide)
* [Using Eclipse Che for Editing Code](#using-eclipse-che-for-editing-code)


## Introduction

On every pipeline execution, the code goes through the following steps:

1. Code is cloned from Gogs, built, tested and analyzed for bugs and bad patterns
2. The WAR artifact is pushed to Nexus Repository manager
3. A container image (_tasks:latest_) is built based on the _Tasks_ application WAR artifact deployed on JBoss EAP 6
4. The _Tasks_ container image is deployed in a fresh new container in DEV project
5. If tests successful, the DEV image is tagged with the application version (_tasks:7.x_) in the STAGE project
6. The staged image is deployed in a fresh new container in the STAGE project

The following diagram shows the steps included in the deployment pipeline:

![](images/pipeline.png?raw=true)

The application used in this pipeline is a JAX-RS application which is available on GitHub and is imported into Gogs during the setup process:
[https://github.com/OpenShiftDemos/openshift-tasks](https://github.com/OpenShiftDemos/openshift-tasks/tree/eap-7)

## Prerequisites
* 8+ GB memory for OpenShift (10+ GB memory if using SonarQube)
* JBoss EAP 7 imagestreams imported to OpenShift (see Troubleshooting section for details)

## Deploy on RHPDS

If you have access to RHPDS, provisioning of this demo is automated via the service catalog under **OpenShift Demos &rarr; OpenShift CI/CD for Monolith**. If you don't know what RHPDS is, read the instructions in the next section.

## Deploy on OpenShift (Script)
You can se the `scripts/provision.sh` script provided to deploy the entire demo:

  ```
  ./provision.sh --help
  ./provision.sh deploy --deploy-sonar --deploy-che --ephemeral
  ./provision.sh delete 
  ```
  
## Deploy on OpenShift (Manual)
Follow these [instructions](docs/local-cluster.md) in order to create a local OpenShift cluster. Otherwise using your current OpenShift cluster, create the following projects for CI/CD components, Dev and Stage environments:

  ```shell
  # Create Projects
  oc new-project dev --display-name="Tasks - Dev"
  oc new-project stage --display-name="Tasks - Stage"
  oc new-project cicd --display-name="CI/CD"

  # Grant Jenkins Access to Projects
  oc policy add-role-to-user edit system:serviceaccount:cicd:jenkins -n dev
  oc policy add-role-to-user edit system:serviceaccount:cicd:jenkins -n stage
  ```
OpenShift 3.7 by default uses an older version of Jenkins. Import a newer Jenkins image in order to use with this demo:
  ```
  oc login -u system:admin
  oc import-image jenkins:v3.7 --from="registry.access.redhat.com/openshift3/jenkins-2-rhel7" --confirm -n openshift
  oc tag jenkins:v3.7 jenkins:latest -n openshift
  ```
  
You can choose to use either SonarQube for static code and security analysis or instead use Maven plugins 
and generated reports within the Jenkins:

  ```
  # Deploy Pipeline with SonarQube
  oc new-app -n cicd -f cicd-template.yaml

  # Deploy Pipeline without SonarQube
  oc new-app -n cicd -f cicd-template.yaml --param=WITH_SONAR=false

  # Deploy Pipeline woth Eclipse Che
  oc new-app -n cicd -f cicd-template.yaml --param=WITH_CHE=true
  ```

To use custom project names, change `cicd`, `dev` and `stage` in the above commands to
your own names and use the following to create the demo:

  ```shell
  oc new-app -n cicd -f cicd-template.yaml --param DEV_PROJECT=dev-project-name --param STAGE_PROJECT=stage-project-name
  ```


## Troubleshooting

* If pipeline execution fails with ```error: no match for "jboss-eap70-openshift"```, import the jboss imagestreams in OpenShift.
  ```
  oc create -f https://raw.githubusercontent.com/jboss-openshift/application-templates/master/jboss-image-streams.json -n openshift
  ```
* If Maven fails with `/opt/rh/rh-maven33/root/usr/bin/mvn: line 9:   298 Killed` (e.g. during static analysis), you are running out of memory and need more memory for OpenShift.

## Demo Guide

1. A Jenkins pipeline is pre-configured which clones Tasks application source code from Gogs (running on OpenShift), builds, deploys and promotes the result through the deployment pipeline. In the CI/CD project, click on _Builds_ and then _Pipelines_ to see the list of defined pipelines.

    Click on _tasks-pipeline_ and _Configuration_ and explore the pipeline definition.

    You can also explore the pipeline job in Jenkins by clicking on the Jenkins route url, logging in with the OpenShift credentials and clicking on _tasks-pipeline_ and _Configure_.

2. Run an instance of the pipeline by starting the _tasks-pipeline_ in OpenShift or Jenkins.

3. During pipeline execution, verify a new Jenkins slave pod is created within _CI/CD_ project to execute the pipeline.

4. Pipelines pauses at _Deploy STAGE_ for approval in order to promote the build to the STAGE environment. Click on this step on the pipeline and then _Promote_.

5. After pipeline completion, demonstrate the following:
  * Explore the _snapshots_ repository in Nexus and verify _openshift-tasks_ is pushed to the repository
  * Explore SonarQube or pipeline in Jenkins and show the metrics, stats, code coverage, etc
  * Explore _Tasks - Dev_ project in OpenShift console and verify the application is deployed in the DEV environment
  * Explore _Tasks - Stage_ project in OpenShift console and verify the application is deployed in the STAGE environment  

![](images/jenkins-analysis.png?raw=true)
![](images/sonarqube-analysis.png?raw=true)

6. Clone and checkout the _eap-7_ branch of the _openshift-tasks_ git repository and using an IDE (e.g. JBoss Developer Studio), remove the ```@Ignore``` annotation from ```src/test/java/org/jboss/as/quickstarts/tasksrs/service/UserResourceTest.java``` test methods to enable the unit tests. Commit and push to the git repo.

7. Check out Jenkins, a pipeline instance is created and is being executed. The pipeline will fail during unit tests due to the enabled unit test.

8. Check out the failed unit and test ```src/test/java/org/jboss/as/quickstarts/tasksrs/service/UserResourceTest.java``` and run it in the IDE.

9. Fix the test by modifying ```src/main/java/org/jboss/as/quickstarts/tasksrs/service/UserResource.java``` and uncommenting the sort function in _getUsers_ method.

10. Run the unit test in the IDE. The unit test runs green. 

11. Commit and push the fix to the git repository and verify a pipeline instance is created in Jenkins and executes successfully.

![](images/openshift-pipeline.png?raw=true)


## Using Eclipse Che for Editing Code

If you deploy the demo template using `WITH_CHE=true` paramter, or the deploy script and use `--deploy-che` flag, then an [Eclipse Che](https://www.eclipse.org/che/) instances will be deployed within the CI/CD project which allows you to use the Eclipse Che web-based IDE for editing code in this demo.


Here is a step-by-step guide for editing and pushing the code to the Gogs repository (step 6) using Eclipse Che.

Click on Eclipse Che route url in the CI/CD project which takes you to the workspace administration page. Select the *Java* stack and click on the *Create* button to create a workspace for yourself.

![](images/che-create-workspace.png?raw=true)

Once the workspace is created, click on *Open* button to open your workspace in the Eclipse Che in the browser.

![](images/che-open-workspace.png?raw=true)

It might take a little while before your workspace is set up and ready to be used in your browser. Once it's ready, click on **Import Project...** in order to import the `openshift-tasks` Gogs repository into your workspace.

![](images/che-import-project.png?raw=true)

Enter the Gogs repository HTTPS url for `openshift-tasks` as the Git repository url. You can find the repository url in Gogs web console. Make sure the check the **Branch** field and enter `eap-7` in order to clone the `eap-7` branch which is used in this demo. Click on **Import**

![](images/che-import-git.png?raw=true)

Change the project configuration to  **Maven** and then click **Save**

![](images/che-import-maven.png?raw=true)

Configure you name and email to be stamped on your Git commity by going to **Profile > Preferences > Git > Committer**.

![](images/che-configure-git-name.png?raw=true)

Follow the steps 6-10 in the above guide to edit the code in your workspace. 

![](images/che-edit-file.png?raw=true)

In order to run the unit tests within Eclipse Che, wait till all dependencies resolve first. To make sure they are resolved, run a Maven build using the commands palette icon or by clicking on **Run > Commands Palette > build**. 

Make sure you run the build again, after fixing the bug in the service class.

Run the unit tests in the IDE after you have corrected the issue by right clicking on the unit test class and then **Run Test > Run JUnit Test**

![](images/che-run-tests.png?raw=true)

![](images/che-junit-success.png?raw=true)


Click on **Git > Commit** to commit the changes to the `openshift-tasks` git repository. Make sure **Push commited changes to ...** is UNCHECKED. The Gogs git server deployed on your OpenShift cluster, does not support neither SSH or OAuth. Therefore, you should use the **Terminal** window in Eclipse Che in order to use push the changes via http to the Gogs repository.

![](images/che-commit.png?raw=true)

Click on the **Terminal** window and push the changes to the upstream repository:
```
cd openshift-tasks
git push origin eap-7
```

Enter `gogs/gogs` as username and password.

![](images/che-git-push.png?raw=true)
