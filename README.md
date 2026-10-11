1. Namma project-oda overall architecture









1. GitHub — Source Code
Spring Boot application + Dockerfile + buildspec.yml

Source retrieval

2. AWS CodeBuild
Fetch code → Maven build → Tests → Docker image build

Docker image push

3. Amazon ECR
Store and version Docker images in a private registry

Next stage — deployment

4. Deployment — Next step
EC2 + Docker, ECS or EKS to run the application


Important: the first three stages are supported by the work and screenshots you've shared. The deployment stage is our next step; a successful image build does not yet prove that the application is running.
2. Step by step — what we configured
Step 1: GitHub repository









Repository: sarathi03/springboot-aws-deploy
GitHub is where we keep the application source code and build-related files. We configured AWS CodeBuild to retrieve the source from this repository.


What this gives us:
- Central place to store application code.
- Version history through Git commits.
- A source for repeatable builds.
- A foundation for triggering future builds automatically when code changes.
Step 2: SSH key configuration
Nee Ubuntu EC2 instance-la indha commands run pannina:
ssh-keygen -t ed25519 -C "sarathi03"
cat ~/.ssh/id_ed25519.pub
ssh -T git@github.com


We created an SSH key pair:
- id_ed25519 — private key; must remain secret.
- id_ed25519.pub — public key; added to GitHub.
The authentication test returned:
Hi sarathi03! You've successfully authenticated,
but GitHub does not provide shell access.


That means GitHub accepted your SSH key.
Why did we do this? To allow your EC2 Ubuntu environment to authenticate with GitHub over SSH, for example, when cloning or pulling your repository.
One important distinction: CodeBuild's GitHub connection is a separate authentication setup. The SSH key on EC2 does not automatically authenticate CodeBuild. CodeBuild uses the GitHub connection/credential configured in the AWS console.
Step 3: AWS CodeBuild project









Project name: springboot-aws-build
AWS CodeBuild is the managed build service. Instead of manually running Maven and Docker commands each time, we define the build process and let AWS execute it in a temporary build environment.


We configured these main settings:
Setting	What it means
Source provider	GitHub
Repository	sarathi03/springboot-aws-deploy
Buildspec file	buildspec.yml
Service role	codebuild-springboot-aws-build-service-role
Region	Mumbai — ap-south-1
The service role is important because CodeBuild needs AWS permissions to perform its work, including pushing an image to ECR.
Step 4: GitHub credentials and webhook problem
Initially, the console showed a message that GitHub was not connected. You configured a GitHub personal access token (PAT) connection, and CodeBuild showed that the account was successfully connected.
Then we encountered this error:
Webhook creation failed — GitHub API limit reached or a permission issue encountered when creating the webhook.

What does that mean?
A webhook lets GitHub notify CodeBuild when an event occurs, such as a push to a repository. The error meant that the automatic webhook setup had failed; it did not necessarily mean that CodeBuild could not retrieve your source code.
You started a build manually, and it succeeded. So we were able to validate the build independently of the webhook.
Step 5: buildspec.yml — the build instructions
This is one of the most important files in our project, da.
AWS CodeBuild needs to know which commands to execute. We provide those instructions in buildspec.yml, written in YAML. AWS reads this file and runs the specified commands in the build environment. 

AWS CodeBuild
+1



A typical build has these phases:
Phase	Purpose
install	Prepare Java, Maven or other required tools
pre_build	Authenticate to ECR and prepare the build
build	Compile the application, run tests and build the Docker image
post_build	Push the image to ECR and perform final tasks
Your build logs showed CodeBuild downloading the GitHub source and locating buildspec.yml. The exact commands in your repository determine what happened in each phase.
The basic flow looks like this:
GitHub source code
       |
       v
buildspec.yml
       |
       v
Maven build and tests
       |
       v
Docker image creation
       |
       v
Push image to Amazon ECR


Why is this useful? You can reproduce the same build process without manually entering every command on an EC2 instance.
Step 6: IAM service role
Your CodeBuild project displays this role:
codebuild-springboot-aws-build-service-role
IAM controls what an AWS service is allowed to do. This role lets CodeBuild obtain the AWS permissions it needs.
For our workflow, the role needs the appropriate permissions to:
- Write build logs to CloudWatch Logs.
- Authenticate to Amazon ECR.
- Upload Docker image layers and image manifests to ECR.
- Retrieve any additional resources the build requires.
The permissions must be configured correctly. We should not grant unrestricted administrator access just to make the build work.
Step 7: The first successful build









Build succeeded
Project: springboot-aws-build
Your screenshot shows build number 2 with status Succeeded, and the reported duration was approximately one minute.
This is evidence that CodeBuild completed the configured build process successfully.


We also saw a resolved source commit in the build details. That matters because a build should be traceable to the exact version of code it processed.
One important distinction: A successful build means the configured commands completed. It does not automatically prove that the deployed application works correctly or that every test covers all possible failures.

3. Amazon ECR — our Docker image registry









Repository: spring-demo-ecr
Registry type: Private
Region: ap-south-1 — Mumbai
Image size: approximately 195.9 MB


Your latest screenshot shows one image in ECR, with the tags latest and build-3ac095b3-da07-42c3-92dd-da9dd8f43451.
That confirms the Docker image was pushed to the repository.
What does ECR actually do?
Imagine a Docker image as a packaged application containing the application artifact and its required runtime components.
- Docker builds the image.
- ECR stores the image.
- A server or container platform pulls the image.
- Docker or the container platform starts a container from that image.
ECR is a registry, not the place where your Spring Boot application automatically runs. AWS documents the push process as authenticating the Docker client, tagging the image for the registry, and pushing it to the repository. 

Amazon ECR
+1



Why are there two tags?
Tag	Purpose
latest	A convenient tag that usually points to the image considered current
build-3ac095b3-...	A build-specific identifier for tracing the image back to a particular build
A unique build tag is especially useful for rollbacks and debugging. For production deployments, prefer deploying a known version or immutable image digest rather than relying only on latest.
4. What struggles did we face?
These are the key issues visible in the screenshots and commands you shared.
Issue 1: SSH authentication
You initially had authorized_keys and known_hosts, but no GitHub client key pair. We generated an Ed25519 key pair, registered the public key with GitHub and verified authentication using ssh -T git@github.com.



Issue 2: CodeBuild GitHub connection
CodeBuild initially reported that GitHub was not connected. You configured the connection, and the console subsequently showed a successful connection.



Issue 3: Webhook creation failed
AWS reported a GitHub API limit or permission issue. You started a build manually, which succeeded. Automatic triggering after a GitHub push still needs separate verification.



Issue 4: Image delivery to ECR
The image appeared in the private ECR repository after the successful build. That verifies the image delivery stage for this build.



Important security check
One of your screenshots showed a value beginning with SHA256: entered into the GitHub personal access token field. That format is an SSH key fingerprint, not a GitHub PAT.
If that value was actually saved as the PAT, correct the connection configuration before relying on future builds. Use a proper GitHub connection or a valid token with the minimum required permissions. Never paste a private SSH key or an actual PAT into chat or logs.
Also, the EC2 SSH key you created and the credentials used by CodeBuild serve different purposes.

5. What can we use this project for in a real company?
Use case	Benefit
Continuous Integration (CI)	Build and test code consistently
Automated Docker builds	Avoid manual image creation
Versioned images	Identify which application version is being deployed
Private image storage	Control access to container images
Continuous Delivery (CD)	Supply a verified image to the deployment stage
Rollback	Redeploy an earlier known-good image
Team collaboration	Use the same build process across development and operations
For example, if a developer updates the Spring Boot application and pushes a commit, an automated workflow can build the new version, run tests, create a Docker image and publish it to ECR. A subsequent deployment stage can then release that image to an environment.
At present, we have demonstrated the manual build path. Automatic triggering and application deployment are the remaining things to validate or implement.
6. What have we completed, and what is next?
Project progress
GitHub repository
Source code available


SSH authentication
Verified from EC2


CodeBuild project
Created and configured


Buildspec execution
Build succeeded


Docker image in ECR
Image visible in private repository



Automatic GitHub webhook
Resolve and verify push-triggered builds


Application deployment
Pull and run the image on a target


Application health check
Verify the Spring Boot endpoint


End-to-end CI/CD
Connect source, build and deployment



My recommendation is to do the EC2 deployment next. We already have an image in ECR, so we can configure the EC2 instance to authenticate to ECR, pull the image, run the container, open the required security-group port and verify the application from a browser.
That will take us from simply building a Docker image to actually running our Spring Boot application on AWS.
7. Interview-la epdi explain pannalaam?
I built a Spring Boot application deployment workflow using AWS CodeBuild, Amazon ECR, Docker and GitHub.
I configured a GitHub repository as the source for AWS CodeBuild and used a buildspec.yml file to define the build process. I also configured the required AWS service role and GitHub connection.
During setup, I faced a GitHub webhook creation issue related to API limits or permissions. I started a manual build to validate the process. The build succeeded, and the Docker image was pushed to a private Amazon ECR repository.
The next step is to automate the build trigger and deploy the image to an EC2 instance, followed by application health checks.




Dei, indha explanation-la important thing enna-na, nee just AWS console-la buttons click pannala. Source control, authentication, build automation, IAM permissions, container image management nu oru DevOps workflow-oda different components-ah connect pannirukka.

Step 1: Open IAM
1. AWS Console-la search bar-la IAM nu search pannu.
2. Click Roles.
3. Search for this role:
codebuild-springboot-aws-build-service-role
4. Andha role-a open pannu.
5. Click Add permissions → Create inline policy.
Step 2: Add ECR permissions
Select the JSON tab and paste this policy:

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage"
      ],
      "Resource": "arn:aws:ecr:ap-south-1:594980084703:repository/spring-demo-ecr"
    }
  ]
}



This allows the CodeBuild role to authenticate with ECR and push an image to your spring-demo-ecr repository. The repository-specific permissions assume that repository already exists in Mumbai (ap-south-1).
6. Click Next.
7. Policy name: CodeBuildECRPushPolicy
8. Click Create policy.
Step 3: Retry the build
Return to CodeBuild → springboot-aws-build → Start build.

