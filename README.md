# Continuos Integration and Delivery over AWS

## Problem

Normal process:

- Developer takes a task in SCRUM (Jira or other PMS)
- Submit code to control version Github / Code Commit
- CI Server fetch code 
- Automatic deployment in development
- Automatic deployment in target production

## Pieces

There are two CloudFormation stacks being used, the Jenkins stack and the Target environment.

The Jenkins Stack

* Creates the jenkins.example.com Route53 Hosted Zone
* Creates an EC2 instance with Tomcat and Jenkins installed and configured on it.
* Runs the CD Pipeline

The Target Stack

* Creates the target.example.com Route53 Hosted Zone
* Creates an EC2 instance with environment (Rails, Java or other + databases).
* Runs the Custom Application.

## Team

2 Senior and 2 Junior Devops + Manager.

SCRUM management: Half of team dedicated to Devops projects + half of the team SRE.

## Timing

1 month: Normal deployment with normal staging.

## Solution

A single delivery pipeline that gives our customer (developers, testers, etc.) unencumbered access to resources and a single click automated deployment to production. To enable this, the pipeline needed to include:

* The ability for any authorized team member to create a new target environment using a single click
* Automated deployment to the target environment
* End-to-end testing
* The ability to terminate unnecessary environments
* Automated deployment into production with a single click

## Technologies

### AWS

AWS EC2: Cloud-based virtual hardware instances
We use EC2 for all of our virtual hardware needs. All instances, from development to production are run on EC2

AWS S3: Cloud-based storage
We use S3 as both a binary repository and a place to store successful build artifacts.

AWS IAM: User-based access to AWS resources
We create users dynamically and use their AWS access and secret access keys so we don’t have to store credentials as properties

AWS CloudWatch: System monitoring
Monitors all instances in production. If an instance takes an abnormal amount of strain or shuts down unexpectedly, SNS sends an email to designated parties

AWS SNS: Email notifications
When an environment is created or a deployment is run, SNS is used to send notifications to affected parties.

Cucumber: Acceptance testing
Cucumber is used for testing at almost every step of the way. We use Cucumber to test infrastructure, deployments and application code to ensure correct functionality. Cucumber’s unique english-ess  verbiage allows both technical personnel and customers to communicate using an executable test.

Liquibase: Automated database change management
Liquibase is used for all database changesets. When a change is necessary within the database, it is made to a liquibase changelog.xml

AWS CloudFormation: Templating language for orchestrating AWS resources
CloudFormation is used for creating a Jenkins environment and Target environment. For instance for the Jenkins environment it creates the EC2 instance with CloudWatch monitoring alarms, associated IAM user, SNS notification topic, everything required for Jenkins to build. This along with Jenkins are the major pieces of the infrastructure.

AWS SimpleDB: Cloud-based NoSQL database
SimpleDB is used for storing dynamic property configuration and passing properties through the CD Pipeline. As part of the environment creation process, we store multiple values such as IP addresses that we need when deploying the application to the created environment.

Jenkins: We will use Jenkins to implement a Continous Delivery pipeline using the Build Pipeline plugin.
Jenkins runs the CD pipeline which does the building, testing, environment creation and deploying. Since the CD pipeline is also code (i.e. configuration code), we version our Jenkins configuration.

Capistrano: Deployment automation
Capistrano orchestrates and automates deployments. Capistrano is a Ruby-based deployment DSL that can be used to deploy to multiple platforms including Java, Ruby and PHP. It is called as part of the CD pipeline and deploys to the target environment.

Ansible: Infrastructure automation
Ansible takes care of the environment provisioning. CloudFormation requests the environment and then calls Puppet to do the dynamic configuration. We configured Puppet to install, configure, and manage the packages, files and services.

Github or Code Commit: Version control system
Github or Code Commit is the version control repository where every piece of the infrastructure is stored. This includes the environment scripts such as the Ansible modules, the CloudFormation templates, Capistrano deployment scripts, etc.

## Pipeline

The CD pipeline consists of Jenkins jobs. These jobs are configured to run one after the other. If any one of the jobs fail, the pipeline fails and that release candidate cannot be released to production. The five Jenkins jobs are listed below.

1- A job that set the variables used in the pipeline (SetupVariables)
2- Main build job (Build)
3- Production database update job (StoreLatestProductionData)
4- Target environment creation job (CreateTargetEnvironment)
5- A deployment job (DeployApplication) which enables a one-click deployment into production.

We use Jenkins plugins to add more features to our Jenkins install (assumming is a Java app for example, you should customize):

* Stack plugins: There are several plugins for Grails or other stacks.
* Github: http://updates.jenkins-ci.org/download/plugins/subversion/1.40/subversion.hpi
* Paramterized Trigger: http://updates.jenkins-ci.org/download/plugins/parameterized-trigger/2.15/parameterized-trigger.hpi
* Copy Artifact: http://updates.jenkins-ci.org/download/plugins/copyartifact/1.21/copyartifact.hpi
* Build Pipeline: http://updates.jenkins-ci.org/download/plugins/build-pipeline-plugin/1.2.3/build-pipeline-plugin.hpi
* S3: http://updates.jenkins-ci.org/download/plugins/s3/0.2.0/s3.hpi

### Examples setup variables

Parameter: STACK_NAME
Type: String
Where: Used in both CreateTargetEnvironment and DeployManateeApplication jobs
Purpose: Defines the CloudFormation Stack name and SimpleDB property domain associated with the CloudFormation stack.

Parameter: HOST
Type: String
Where: Used in both CreateTargetEnvironment and DeployManateeApplication jobs
Purpose: Defines the CNAME of the domain created in the CreateTargetEnvironment job. The DeployManateeApplication job uses it when it dynamically creates configuration files. For instance, in test.oneclickdeployment.com, test would be the HOST

Parameter: PRODUCTION_IP
Type: String
Where: Used in the StoreProductionData job
Purpose: Sets the production IP for the job so that it can SSH into the existing production environment and run a database script that exports the data and uploads it to S3.

We add more parameters to configure the environment.

### Steps

Basically we will have these steps:

 - Build: Compiles the source code and creates a WAR file (depending of the app)
 - Store Production: Save the production data with a dump of the Postgres or MySQL on S3
 - Create Target: Create a new environment using a Cloud Formation template to create the AWS resources and provision with Ansible.
 - Deploy using Capistrano the new war and update configuration changes and restarts all services. Assuming the tests pass, an AWS SNS email gets dispatched to the developer with information on how to access their new development application. We version the Jenkins configuration to rollbacks.	A script is executed each hour that checks for new jobs or updated configuration and commits them up to version control.

 The CD pipeline enables any change in the app, infrastructure, database or configuration to make a pass to production using automation.

## Budget

We can use the AWS calculator to estimate the budget for the Jenkins and Platform stack. This environment will use auto scale to grow automatically if the number of deployments in development and production grows.
At least we will need the resources for the initial stack. We will need to calculate depending the resources, we can start using the t2 to t4. 

## Cloudformation

We will create a CloudFormation template which references various AWS resources that you want to use. When the template runs, it will create the AWS resources using the AWS Console, CloudFormation CLI tools or the CloudFormation API.

We use CloudFormation in order to have a fully scripted, versioned infrastructure.

The template does:

1. IAM User with AWS Access keys 
2. SNS Topic 
3. CloudWatch Alarm and SNS topic 
4. Security Group
5. Wait Condition 
6. Jenkins EC2 Instance is created with the Security Group and uses AWSInstanceType2Arch and AWSRegionArch2AMI to decide what AMI and OS type to use.
7. Jenkins EC2 Instance runs UserData script and executes cfn_init.
8. Wait Condition waits for Jenkins EC2 instance to finish UserData script
9. Elastic IP is allocated and associated with Jenkins EC2 instance
10. Route53 domain name with Jenkins Elastic IP
11. If everything creates successfully, the stack signals complete and outputs are displayed

## Dynamic config

Throughout the CD pipeline, we often need to manage state across multiple Jenkins jobs. To do this, we use SimpleDB.

As Jenkins jobs or CloudFormation templates are run, we often end up with properties that need to be used elsewhere. Instead of hard coding all of the values to be used in a property file, we create, store and retrieve them as the pipeline executes.

We could use alternatively S3 for this part instead having properties with hard coded key/value pairs that gets passed into the pipeline.

## Deployment Automation

We use Capistrano to script our deployments to target environments.

We use Capistrano in order to have a fully scripted, versioned deployment. Every step in our application deployment is scripted and fully automated – which reduces errors when deploying. This gives us complete control over our deployment and the ability to deploy whenever we are ready.

With Capistrano’s automation in conjunction with Cucumber’s acceptance testing, we are given a high level of confidence in our deployments and that when they are run, the application will be deployed successfully.

## Ansible

We can use other configuration management tools instead Ansible like Puppet or Chef. 

We use Ansible for provisioning our target environments. We create a new target environment as part of the CD pipeline.

1. CloudFormation dynamically creates a params in Ansible with AWS variables
2. CloudFormation runs playbooks as part of UserData
3. Ansible runs the playbooks defined in hosts/default.pp.
4. Cucumber acceptance tests are run to verify the infrastructure was provisioned correctly.

This includes java version, services running (nginx, apache, mysql, etc), libraries, depedencies, compilers, etc.
