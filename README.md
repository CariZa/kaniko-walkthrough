# kaniko-walkthrough

This is a walkthrough for the tool kaniko:

https://github.com/GoogleContainerTools/kaniko

This walkthrough has 3 parts:

- Part 1: 100% local - Just build the image with Kaniko using Docker
- Part 2: Include the cloud - Build image and send to aws ecr
- Part 3: Kubernetes!

![Image that says No Prob Llama with a cool Llama](/images/noprobllama-kaniko.jpg)

Image reference: https://www.freepik.com/premium-vector/jumping-alpaca-llama-with-no-prob-llama-motivational-quote_6410008.htm

## Why use Kaniko?

Don't build images directly, for example using:

    $ docker build ... <-- Don't do this for your final images (for local testing this is fine).

You risk building containers with root access still inside. This is a security risk. Kaniko helps you remove root from the containers you build. 

Kaniko is a great tool to use inside CI/CD pipelines because you can use it to build more secure images to tag and send to registries (eg aws' ecr).

Full disclosure though, Kaniko itself still has root. Which means it might not fit in with certain company comliances that expect no root at all in any images on an environmnet.

Read more:

https://zwischenzugs.com/2018/04/23/unprivileged-docker-builds-a-proof-of-concept/

# Part 1

## Using just Docker & caching locally

This is just a quick warmup exercise. Get to know Kaniko in an simpler context.

### Requirements:

Docker

https://www.docker.com/products/docker-desktop

Make sure you have a working Docker environment and you can run and you can run:

    $ docker version

### Docker & Kaniko Example

https://github.com/GoogleContainerTools/kaniko#running-kaniko-in-docker

You can test Kaniko using just Docker to start. This is a quick warmup before we get into working with cloud based registry like ecr.

Provide a Dockerfile:

I've got this Dockerfile in the ./example-1 folder:

```
# Dockerfile
FROM ubuntu
ENTRYPOINT ["/bin/bash", "-c", "echo hello"]
```

Then run (this will just build the image, but not do anything with it):

```
docker run --rm \
--name test-kaniko \
--volume $(pwd)/example-1/Dockerfile:/workplace/Dockerfile \
gcr.io/kaniko-project/executor:latest --no-push --dockerfile /workplace/Dockerfile
```

Just a quick overview of the above command (feel free to skip this section if it all makes sense already).

Volume:

    $(pwd)/example-1/Dockerfile:/workplace/Dockerfile

Volume your Dockerfile inside the Kaniko instance at /workplace/Dockerfile

Then tell Kaniko where to find the Dockerfile you want Kaniko to build:

    --dockerfile /workplace/Dockerfile

Don't push to a registry just yet (this will just build the image, but do nothing further):

    --no-push

A step like this could be used to notify you if a build stops working, so even this simplified example of using Kaniko could help you in a CI pipeline to do a simple check.

# Part 2

## Use Kaniko to push to ecr

This section is going to take a little bit longer to work through, put on some good music (if that's your vibe) and just work through it bit by bit.

To build an image, tag and push it to ecr you need to setup a few things, and get a few credentials in place. 

Please note: setting up AWS resources will incur some costs - be aware, and delete resources you don't need when you are done.

### Requirements:

AWS CLI

https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html

### Amazon Elastic Container Registry - ECR

On AWS, Select Services, then Select **Elastic Container Registry**

Create a Repository

![Elastic Container Registry](/images/ecr1.png)

(If it's your first registry, click the get started button. Else just click create repository)

Insert the name of your repo, example "kaniko-test", and then click "create repository":

![Elastic Container Registry Create](/images/ecr2.png)

Copy your full repo url, keep it somewhere useful:

    xxxxxxxx.dkr.ecr.eu-central-1.amazonaws.com

### Authentication layers

Best way to handle this is create a new IAM.

Login to the aws console: 

Go to Services and select **IAM**

Then create a new user:

![Elastic Container Registry Add User](/images/add-user.png)

Fill in these details:

![Elastic Container Registry Set User Details](/images/set-user-details.png)

Select 

![Elastic Container Registry Set User Details](/images/set-user-permissions.png)

For this example we're going to attach an existing policy directly. Select "Attach existing policies directly" and search for and select "AmazonEC2ContainerRegistryFullAccess".

Do not select the initial choice of admin access. Rather be more specific with access, especially when it comes to cloud.

For the rest of the steps, just click next till you get the option to **Create User**. Click the **Create User** button.

After the user gets created, you need to copy and paste these values somewhere safe:

- aws_access_key_id
- aws_secret_access_key

These values will be used later.

### AWS CLI

Remember to make sure you have the aws cli installed.

If you don't have the aws cli installed yet, you'll need to set that up.

https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html

Test you have it working:

    $ aws --version

I'm using version 1 still. 

# TODO UPGRADE TO VERSION 2

You need a few values:

- aws_access_key_id
- aws_secret_access_key

Configure a new aws profile:

    $ aws configure --profile aws-kaniko-test

Fill in the values you copied:

AWS Access Key ID []: <paste aws_access_key_id>
AWS Secret Access Key []: <paste aws_secret_access_key>
Default region name []: <insert the same region you setup the ecr on>
Default output format [None]: 

This will add to 

    ~/.aws/credentials
    ~/.aws/config

~/.aws/credentials

```
[aws-kaniko-test]
aws_access_key_id = XXXXXXXXXX...
aws_secret_access_key = xXxXxXxXxXxXxXxX...
```

~/.aws/config

```
[profile aws-kaniko-test]
region = us-east-1
```

Test aws user:

$ aws sts get-caller-identity --profile aws-kaniko-test

You should get a response like this:

```
{
    "Account": "XXXXXXXX", 
    "UserId": "XXXXXXXXX", 
    "Arn": "arn:aws:iam::XXXXXXXXX:user/kaniko-user"
}
```

If you get errors you may need to relook your IAM user you created.

Test Docker login:

$ aws ecr get-login --region eu-central-1 --profile aws-kaniko-test --no-include-email

Run this command when you get it (this will trigger the Docker login):

$ $(aws ecr get-login --region eu-central-1 --profile aws-kaniko-test  --no-include-email)

Once you've confirmed your Docker login works, lets set up the files we need to mount inside the running Kaniko instance.

### Setup files

Your Docker config

    $ touch config.json

Add:

```
{
    "credsStore":"ecr-login",
    "credHelpers": {
    "XXXXXXXXX.dkr.ecr.us-east-1.amazonaws.com": "ecr-login"
    }
}
```

You have your aws credentials file:

    $ touch credentials

Add:

```
[default]
aws_access_key_id = <your aws_access_key_id>
aws_secret_access_key = <your aws_secret_access_key>
```

Note: change "aws-kaniko-test" to "default". You need to change the profile to default for Kaniko.

The files to mount to:

    /kaniko/.docker/config.json
    /kaniko/.aws/credentials

$ touch config.json


Create a variable called "ECR_REPO" with you ecr repo value:

    export ECR_REPO="XXXXXXXX.dkr.ecr.eu-central-1.amazonaws.com/kaniko-test"

Your Kaniko command to send tagged images to ecr:

```
docker run --rm --name test-kaniko \
--volume $(pwd)/example-1/Dockerfile:/workplace/Dockerfile \
--volume $(pwd)/credentials:/kaniko/.aws/credentials \
--volume $(pwd)/config.json:/kaniko/.docker/config.json \
--env AWS_SHARED_CREDENTIALS_FILE="/kaniko/.aws/credentials" \
--env DOCKER_CONFIG="/kaniko/.docker" \
gcr.io/kaniko-project/executor:latest --dockerfile /workplace/Dockerfile --destination=$ECR_REPO:0.1 --destination=$ECR_REPO:latest
```

If you have slow internet (as I do at the moment), this may take a while. Go make tea. Reflect. Say hi to yourself. 

Once successfully completed you will see your image in your registry \o/

![Elastic Container Registry Image Sent](/images/kaniko-test-ecr-image-sent.png)

So you can use kaniko to just build, and add the --no-push flag so you don't push to ecr. When you're ready to push, you run a command without the --no-push.



## A Quick note on Kaniko images:

There are some kaniko images worth noting:

- gcr.io/kaniko-project/executor:latest -> for final production readiness builds
- gcr.io/kaniko-project/executor:debug -> for debugging (this helped me solve my issues with ecr)
- gcr.io/kaniko-project/warmer -> caching locally

# Part 3

## Using minikube to run Kaniko the kubernetes way 

If you've managed to get the Docker examples working, let's attempt to get it working with kubernetes using minikube. It will be very similar, we just have to convert the Docker approach to a kubernetes approach:

# TODO


