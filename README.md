# Automate-Model-Deployment-with-GitHub-Actions-and-AWS
Introduction
In a typical software development process, the deployment comes at the end of the software development life cycle.

I described how we could build a model, wrap it with a Rest API, containerize it, and finally deploy it on cloud services. But the entire process of containerization and deployment was manual. In this article, we will create a continuous deployment pipeline with which we can automate model deployment.

By the end of the article, you will

Learn the importance of automated Machine Learning deployment workflow (MLops)

Learn about the basics of CI/CD.

Learn the working and benefits of GitHub Actions

Learn how to create an automated pipeline for continuous model deployment

What is MLOps?
MlOps is an acronym for Machine Learning Operations and is the DevOps equivalent of Machine Learning. MlOps streamline the process of taking the models to production and monitoring them. A typical machine learning development cycle involves data ingestion, data preparation, model building, deploying, and monitoring. It requires tight collaboration between data engineers, Data scientists, and ML engineers to keep the entire operation synchronized.

DevOps streamlines software development for the rapid shipping of applications. Similarly, MlOps intends to reduce the friction between different data teams to increase the pace of model development and monitoring by implementing state-of-the-art CI/CD/CT practices.

CI/CD in Machine Learning
CI/CD is the method of automating various stages of the software development life cycle. CI and CD are acronyms for Continuous Integration and Continuous Delivery (Continuous Deployment).

So, what is CI/CD(and CD)? The Continuous Integration or CI part is concerned with continuous building, testing, and merging of the pull requests to the main branch. It is responsible for speeding the code development process by eliminating redundant manual unit testing tasks.

The CD in CI/CD could either mean Continuous Delivery or Deployment. Continuous delivery usually means a developer’s changes to an application are automatically bug-tested and uploaded to a repository or a container registry. The other CD or Continuous deployment is the process of automated deployment when a new code is added to a shared repository(main branch).

The same concepts are also applied to machine learning model development as well. The CI/CD for machine learning involves training the model, storing it in a suitable registry, and deploying it on a cloud server. In the machine learning context, we can add another process called CT or continuous training. Model performances start to degrade after a while for various reasons. So, the models need to be trained again on new data.

In this article, we will build an automated pipeline in GitHub Actions. The workflow will be triggered when a new commit is made to the main branch.

GitHub Actions
The GitHub actions is a CI/CD tool from GitHub. The best thing about GitHub actions is that we can automate development workflows from the repository itself. It uses YAML(Yet Another Mark-up Language) to define sequential jobs. GitHub marketplace already has thousands of actions to automate various workflows. So, more often than not, there will already be an action related to what you are trying to do. For more information, visit their official documentation.

A typical GitHub workflow will look like this.


COPY
...
name: Workflow Name 
on: push # Define which events can cause the workflow to run
jobs: # Define a list of jobs
  first_job: # ID of the job
    name: First Job # Name of the job
    runs-on: ubuntu-latest # Name of machine to run the job on
    steps:
      ...
  second_job:
    name: Second Job
    runs-on: ubuntu-latest
    steps: 
      ...
On is the event that triggers the workflow. In the above code, the trigger is PUSH. We can further add pushing to which branch triggers the workflow. For example,
on:

push:

branches: [ “main” ]

The jobs define the different tasks the workflow will be executing. These are independent of each other.

The steps are a series of sub-tasks executed in order.

So now that we have a basic know-how of GitHub workflows, let’s start with our main event. And we will figure it out as we proceed.

GitHub Actions for Model Deployment
Like I said earlier, In a previous article, we built a spam classification model and deployed it on AWS ECS. We will now automate the deployment process using GitHub actions.

First, go to your repository and create a folder path .github/workflows/name-of-your-workflow.yml. This is the YAML file that the GitHub action will use to trigger workflows. Make sure you get the name of the folders right.

The next step is to create an IAM role and an ECS task definition for our project.

Configure AWS
An IAM (Identity and Access Management) is created to delegate permissions to a user. The root user can manage the permissions level granted to a user.

Follow their official documentation and create an IAM role with full access. Save the credential somewhere safe. Now login with the IAM user and head over to ECS. Follow this article to create a cluster with Fargate and a task definition.

Click on the task definition you just created, go to the JSON tab, and copy the JSON file. Or run the following command in the AWS CLI.


COPY
aws ecs describe-task-definition --task-definition task-definition-name > filename.json
Now, create a .aws/task-definition.json file path in your repository.

Specify Event
The On parameter will specify the trigger event. And branch parameter defines the commitment to which branch will trigger the workflow.


COPY
name: Deploy to Amazon ECS

COPY
on:
  push:
    branches: [ "main" ]
Now we will define the environment variables.


COPY
env:
  AWS_REGION: ap-south-1                  # set this to your preferred AWS region, e.g. us-west-1
  ECS_SERVICE: custom-service                 # set this to your Amazon ECS service name
  ECS_CLUSTER: default                       # set this to your Amazon ECS cluster name
  ECS_TASK_DEFINITION: .aws/task-definition.json # set this to the path to your Amazon ECS task definition
                                               # file, e.g. .aws/task-definition.json
  CONTAINER_NAME: custom                    # set this to the name of the container in the
                                               # containerDefinitions section of your task definition
Specify Steps
Next, create a job called deploy , which consists of several steps executed in order. The first few steps will set up the environment before running the code.


COPY
jobs:

deploy:

name: Deploy

runs-on: ubuntu-latest

environment: production




steps:

- name: Checkout

uses: actions/checkout@v3
Next, configure AWS and Docker Hub credentials.


COPY
- name: Configure AWS credentials

uses: aws-actions/configure-aws-credentials@v1

with:

aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}

aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

aws-region: ${{ env.AWS_REGION }}




- name: Log in to Docker Hub

uses: docker/login-action@f054a8b539a109f9f41c372932f1ae047eff08c9

with:

username: ${{ secrets.DOCKER_USERNAME }}

password: ${{ secrets.DOCKER_PASSWORD }}
Instead of directly providing passwords, we will use GitHub secrets to encrypt our data.

Secrets are encrypted environment variables that you create in a repository. To create a secret, go to your repository and click Settings → Secrets → Actions → New repository secret.

model deployment

Next, extract metadata for docker and then build and push images to docker hub.


COPY
- name: Extract metadata (tags, labels) for Docker
  id: meta
  uses: docker/metadata-action@98669ae865ea3cffbcbaa878cf57c20bbf1c6c38
  with:
      images: sunilkumardash9/spam_classify #docker hub repository name
- name: Build and push Docker image
  uses: docker/build-push-action@ad44023a93711e3deb337508980b4b5e9bcdc5dc
  with:
    context: .
    push: true
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
If you are wondering,

uses: selects an action for performing a complex task. GitHub marketplace has thousands of actions that make it easier to work with GitHub Actions.

With: defines input parameters required by action

Next, fill in the new image id in the ECS task definition.


COPY
- name: Fill in the new image ID in the Amazon ECS task definition
  id: task-def
  uses: aws-actions/amazon-ecs-render-task-definition@v1
  with:
    task-definition: ${{ env.ECS_TASK_DEFINITION }}
    container-name: ${{ env.CONTAINER_NAME }}
    image: ${{ steps.meta.outputs.tags }}
Now, the final step is to deploy the AWS ECS task definition.


COPY
- name: Deploy Amazon ECS task definition
  uses: aws-actions/amazon-ecs-deploy-task-definition@v1
  with:
    task-definition: ${{ steps.task-def.outputs.task-definition }}
    service: ${{ env.ECS_SERVICE }}
    cluster: ${{ env.ECS_CLUSTER }}
    wait-for-service-stability: true
Review full YAML code


COPY
name: Deploy to Amazon ECS

on:

push:

branches: [ "main" ]
env:


COPY
AWS_REGION: ap-south-1 # set this to your preferred AWS region, e.g. us-west-1

# ECR_REPOSITORY: MY_ECR_REPOSITORY # set this to your Amazon ECR repository name

ECS_SERVICE: custom-service # set this to your Amazon ECS service name

ECS_CLUSTER: default # set this to your Amazon ECS cluster name

ECS_TASK_DEFINITION: .aws/task-definition.json # set this to the path to your Amazon ECS task definition

# file, e.g. .aws/task-definition.json

CONTAINER_NAME: custom # set this to the name of the container in the

# containerDefinitions section of your task definition
permissions:


COPY
contents: read

COPY
jobs:

deploy:

name: Deploy

runs-on: ubuntu-latest

environment: production

COPY
steps:

- name: Checkout

uses: actions/checkout@v3




- name: Configure AWS credentials

uses: aws-actions/configure-aws-credentials@v1

with:

aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}

aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

aws-region: ${{ env.AWS_REGION }}




- name: Log in to Docker Hub

uses: docker/login-action@f054a8b539a109f9f41c372932f1ae047eff08c9

with:

username: ${{ secrets.DOCKER_USERNAME }}

password: ${{ secrets.DOCKER_PASSWORD }}




- name: Extract metadata (tags, labels) for Docker

id: meta

uses: docker/metadata-action@98669ae865ea3cffbcbaa878cf57c20bbf1c6c38

with:

images: sunilkumardash9/spam_classify







- name: Build and push Docker image

uses: docker/build-push-action@ad44023a93711e3deb337508980b4b5e9bcdc5dc

with:

context: .

push: true

tags: ${{ steps.meta.outputs.tags }}

labels: ${{ steps.meta.outputs.labels }}

COPY
- name: Fill in the new image ID in the Amazon ECS task definition

id: task-def

uses: aws-actions/amazon-ecs-render-task-definition@v1

with:

task-definition: ${{ env.ECS_TASK_DEFINITION }}

container-name: ${{ env.CONTAINER_NAME }}

image: ${{ steps.meta.outputs.tags }}




- name: Deploy Amazon ECS task definition

uses: aws-actions/amazon-ecs-deploy-task-definition@v1

with:

task-definition: ${{ steps.task-def.outputs.task-definition }}

service: ${{ env.ECS_SERVICE }}

cluster: ${{ env.ECS_CLUSTER }}

wait-for-service-stability: true
Now, commit this to the main branch to trigger the workflow. Head over to the Actions tab on your GitHub repository. You can see all the steps getting executed in real time.

model deployment

Wait until every step is completed. Now, head over to the cluster you created. Click on the tasks, and wait for a moment if it is in pending status. When it changes to running, you know your app is up and running on the server.

model deployment

To get the IP address of your app, click on the task link, and you will see the public IP under the network section. Copy and paste it into the browser and plug your port at the end. For example http://35.154.18.246:8000/

model deployment

It should show our API message.

API message.

Plug /docs at the end of the link to go to Swagger Ui and see if our app is running.

Swagger Ui

You can also access your API endpoints from a mobile browser as well.

API endpoints

In case you encounter any issues, you can check out the logs section ECS.

model deployment

The deployment process will happen automatically when you make changes to your code.

Conclusion
Throughout the article, we discussed MlOps, CI/CD, and GitHub actions in brief and finally created a continuous model deployment pipeline. So, here are some of the key takeaways from the article.

MlOps practices reduce friction between different data teams and streamline model deployment, building, and monitoring processes.

CI/CD/CT are methods of automating various processes of model building, deploying, and re-training.

GitHub Actions is a CI/CD tool that uses YAML to define workflows.

A workflow has various jobs, and each has different steps that execute specific actions.

So, this was all about continuous deployments with GitHub actions.
