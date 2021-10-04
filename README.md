# Continuous Integration and Deployment with AWS
The general steps we want to follow are:
1. Install docker
2. Build our image
3. Run the test suite
4. Deploy to AWS
## Initial set up with Travis CI
In order to set up a CI workflow we need to create a configuration file for Travis called `.travis.yml`. This should look something like the following:
```
language: generic
sudo: required
services:
  - docker

before_install:
  - docker build -t matthewjhcarr/docker-react -f Dockerfile.dev .

script:
  - docker run -e CI=true matthewjhcarr/docker-react yarn test
```
The important bits here are the following:
```
sudo: required
```
This tells Travis we need superuser privileges (since we'll be installing things)  
```
services:
  - docker
```
This tells Travis that we need Docker installed first and foremost
```
before_install:
  - ...
```
Any commands here will be executed before anything else gets started. Hence why we're building our image here.
```
script:
  - ...
```
A series of commands that may be executed once the installation phase has begun. This is where we will run our tests.
```
docker run -e CI=true matthewjhcarr/docker-react yarn test
```
This runs our image and replaces the initial command with "yarn test". It also forces Jest (the test runner) to run only once so that the process doesn't hang and wait for a response.
## Elastic Beanstalk and AWS
Elastic Beanstalk (EB) is by far the easiest way to get started with production docker instances. It is most appropriate when you are running one single container at a time.

### Updating Travis CI for Elastic Beanstalk
In order to get Travis CI to deploy our application to AWS, we must add some configuration to the YML file we created. This should be as follows:
```
deploy:
  provider: elasticbeanstalk
  region: "us-east-2"
  app: "docker-react"
  env: "Dockerreact-env"
  bucket_name: 
  bucket_path: "docker-react"
  on:
    branch: master
```
Lets break that down:

`provider: elasticbeanstalk`: This tells Travis we're deploying to AWS _(specifically EB)_

`region: "us-east-2"`  
This refers to the region our EB environment was created in, found in the URL on the environment dashboard.

`app: "docker-react"`  
This is whatever name you gave your application. Also found on the environment dashboard.

`env: "Dockerreact-env`  
The name of your environment.

#### Definition: bucket
When Travis deploys your application, it takes all the files in your repo, zips them, and copies them to an S3 bucket _(basically a hard drive on AWS)_.

Once copied, Travis will tell EB that it's copied the files _(and the name of the bucket)_ and that they're available to use for deployment.

Buckets are created for you when creating your EB application _(thank god)_.

`bucket_name: "elasticbeanstalk-us-east-2-857496270468"`  
The name of your S3 bucket. Found on AWS by searching for S3 and looking through the list for the bucket created at the time you'd expect.

`bucket_path: "docker-react"`  
By default this is the same as the name of your application.

```
on:
  branch: main
```
This tells travis that we only want to do any of this when code is pushed to the `main` branch.

### Letting AWS and Travis get to know one another
We want to give Travis __programmatic access__ to our AWS environment, and so we must create a user and generate API keys it can use.

To do this, go to AWS and search for the IAM service. Here, create a new user _(name it something along the lines of docker-react-travis-cli as appropriate)_ and grant it programmatic access __only__.

This will generate both an access key and a secret key that you can then use to grant Travis access.

We then go to Travis CI, go to the settings of our project, and add the keys as AWS_ACCESS_KEY and AWS_SECRET_KEY.

We finally add the following to the `.travis.yml` file, under `deploy`:
```
deploy:
  ...
  access_key_id: $AWS_ACCESS_KEY
  secret_access_key: $AWS_SECRET_KEY
```

We may finally commit our changes and deploy our code.