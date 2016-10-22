# How To Setup CI/CD in Container Development Kit version 2.1.0

This demo works only on Container Developemen Kit verson 2.1.0. When you download CDK from http://developers.redhat.com/products/cdk/download/, choose older downloads.

We assume that you have already made CDK v2.1.0 run on your local and that you are excited to see how the CI/CD works and want to set it up on your local.

## Installing the CI/CD project

0. Inside the $CDK_INSTALL_DIR/components/rhel/misc/shared_folder/rhel-ose do

`vagrant ssh`

This should ssh you to the cdk virtual machine.

1. Clone the git project:

`git clone https://github.com/corpbob/openshift-cd-demo.git`

This should create the directory:

`openshift-cd-demo`

3. We need to get the branch openshift-3.2 so  cd to openshift-cd-demo directory and do:

`git checkout openshift-3.2`

You should see the following files:

`cicd-github-template.yaml  images          jenkins-slave  README.md`
`cicd-gogs-template.yaml    jenkins-master  oc`

4. Create the projects. Execute the following on the commandline:

`oc new-project dev --display-name="Tasks - Dev"`  
`oc new-project stage --display-name="Tasks - Stage"`  
`oc new-project cicd --display-name="CI/CD"`  
`oc policy add-role-to-user edit system:serviceaccount:cicd:default -n cicd`  
`oc policy add-role-to-user edit system:serviceaccount:cicd:default -n dev`  
`oc policy add-role-to-user edit system:serviceaccount:cicd:default -n stage`  

5. Import the template and create the application by executing:

`oc process -f cicd-gogs-template.yaml | oc create -f -`

6. Point your browser to [https://10.1.2.2:8443/console](https://10.1.2.2:8443/console) and login using the following credentials:

username: _admin_
password: _admin_

7. Click on CI/CD. You should see the applications being created from the template. Be patient at this point since OpenShift will download the docker images remotely.

8. When the dial ("circle") for [gogs-cicd.rhel-cdk.10.1.2.2.xip.io](gogs-cicd.rhel-cdk.10.1.2.2.xip.io) turns blue, you can click on the link "[http://gogs-cicd.rhel-cdk.10.1.2.2.xip.io/](http://gogs-cicd.rhel-cdk.10.1.2.2.xip.io/)" to open the Gogs application.

We are now ready to import some code into Gogs. But first, we need to create a Gogs account and create a repository.

## Create New Gogs Account
1. Click on Register
2. Enter username: _demo_
3. Enter password: _demo_
4. Click Create New Account

## Sign In and Create a Repository
1. Click Sign-In in Gogs and enter username/password pair:

_demo/demo_

2. On the right-side navigation, click on "_+_" sign of My Repositories.
3. In the New Repository Form, Enter Repository Name:

_openshift-tasks_

4. Click on Create Repository

Now that we have a bare-bones repository, we now put code into it. Type the following commands:

`cd`
`git clone https://github.com/OpenShiftDemos/openshift-tasks.git`

We need to change the remote url to point to our gogs instance. To do this:

`cd openshift-tasks/`  
`git remote remove origin`  

`git remote add origin http://gogs-cicd.rhel-cdk.10.1.2.2.xip.io/demo/openshift-tasks.git`  
`git config user.name demo`  
`git config user.email demo@example.com`  
`git commit -m “initial import”`  
`git push origin master`  

Git will ask you for the username and password, enter _demo_ for username and _demo_ for password.


## Verify that the files are pushed
1. Go to gogs [http://gogs-cicd.rhel-cdk.10.1.2.2.xip.io](http://gogs-cicd.rhel-cdk.10.1.2.2.xip.io) and click on openshift-tasks. You should see your files there.
 
We now have configured Gogs. Next is to configure Jenkins:

1. In OpenShift, Click on Projects->CI/CD
2. Click on Service [jenkins-cicd.rhel-cdk.10.1.2.2.xip.io](jenkins-cicd.rhel-cdk.10.1.2.2.xip.io)
3. Click on tasks-cd-pipeline.
4. Click on Configure.
5. Change the git url in the pipeline definition:

From: [https://github.com/OpenShiftDemos/openshift-tasks.git](https://github.com/OpenShiftDemos/openshift-tasks.git)

To: [http://gogs-cicd.rhel-cdk.10.1.2.2.xip.io/demo/openshift-tasks.git](http://gogs-cicd.rhel-cdk.10.1.2.2.xip.io/demo/openshift-tasks.git)

6. Click Apply then click Save.

Let's have a sanity check at this point. Start a build in Jenkins.

## Start a Build
0. Go to [http://jenkins-cicd.rhel-cdk.10.1.2.2.xip.io/](http://jenkins-cicd.rhel-cdk.10.1.2.2.xip.io/)
1. Click on the tasks-cd-pipeline and click __Build Now__.
2. This should trigger a build. Initially the build will be slow due to downloading of docker images from remote. Subsequent builds will be faster since these images will already by cached by the local docker registry.

## Here's the fun part. Add a webhook.

This is the part where jenkins will automatically build your project and deploy them to OpenShift after pushing to the repository.
1. Load [http://gogs-cicd.rhel-cdk.10.1.2.2.xip.io/](http://gogs-cicd.rhel-cdk.10.1.2.2.xip.io/)
2. Click on openshift-tasks in My Repositories
3. Click on Settings (at the right-hand side)
4. Click on Webhooks
5. Click on Add Webhook. Choose Gogs.
6. Set the value of Payload URL to  [http://jenkins:8080/job/tasks-cd-pipeline/build?delay=0sec](http://jenkins:8080/job/tasks-cd-pipeline/build?delay=0sec)
7. Click Add Webhook to finish.


## Check that your webhook works. 
1. cd to `/home/vagrant/openshift-tasks`
2. Edit the file `./src/test/java/org/jboss/as/quickstarts/tasksrs/service/UserResourceTest.java`
3. Find the function `getUsersSortedByTask` and comment-out the `@Ignore` line. Save your work.
4. push your changes to gogs:

`git add ./src/test/java/org/jboss/as/quickstarts/tasksrs/service/UserResourceTest.java`  
`git commit -m "removed Ignore"`  
`git push origin master`  

5. Notice that jenkins will immediately build your project. On this occassion, it the unit test will fail.

