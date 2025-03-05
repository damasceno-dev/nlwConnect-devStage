# Full-Stack Application Deployment Guide

This guide provides step-by-step instructions for deploying the complete full-stack application, including infrastructure, server, and web components.

## Project Structure

- **infra**: Terraform infrastructure code (AWS resources)
- **server**: Backend API application
- **web**: Frontend application

## Prerequisites

- AWS CLI installed and configured
- Node.js and npm installed
- Git
- GitHub account with access to this repository

## Deployment Steps

### 1. Infrastructure Setup (infra submodule)

#### 1.1 Initial AWS Configuration

1. Navigate to the infra directory

2. Review the `1-admin/main.tf` file to understand the required AWS console configuration:
   1. create the s3 bucket to manage terraform state
   2. create and attach to the AWS ADMIN profile the initial policy

3. Create a `.secrets` file based on the provided example:
```
cp .secrets.example .secrets
```

4. Edit the `.secrets` file with your AWS credentials and configuration:

* AWS ADMIN is the aws profile that is going to give the AWS RESOURCE CREATOR profile permission to create all resources needed by this app
* AWS RESOURCES CREATOR is the aws profile that is going to create this resources


5. Export the secrets to base64 and create and add them to a variable in github secrets called ENCODED_SECRETS:

```
base64 -i .secrets
```

#### 1.2 Deploy Admin and Resources Infrastructure (Step 1 and 2)
On commit or using some tool to run the workflow locally (i. e: act), the '1-deploy.yml' will run the steps defined in the folder 1-admin and 2-resources
If you using act on mac, this is the command to run the workflow locally. Run it in the root of the infra folder:

act -s ENCODED_SECRETS="$(base64 -i .secrets | tr -d '\n')" --container-architecture linux/amd64


After successful deployment, note down the following workflow outputs:
- *ECR Repository URL*
- *RDS Endpoint*

### 2. Server Configuration and Deployment

#### 2.1 Configure Server Settings

1. Navigate to the server directory

2. Create a `.secrets` file based on the example:
```
cp .secrets.example .secrets
```

3. Update the `.secrets` file with the *ECR Repository URL* obtained from the infrastructure deployment:
```
AWS_ECR_URL="your-ecr-repository-url"
```

4. Create an `appsettings.Production.json` file based on the example:
```
cp DevStage.API/appsettings.Example.json DevStage.API/appsettings.Production.json
```

5. Update the `appsettings.Production.json` file with the *RDS Endpoint* and other settings:
```
{
    "ConnectionStrings": {
    "DefaultConnection": "Server=your-rds-endpoint;Port=5432;Database=yourdb;User Id=youruser;Password=yourpassword;"
        ...
}
```

#### 2.2 Configure GitHub Secrets for CI/CD

1. Convert your `appsettings.Production.json` and `.secrets` file to base64:
```
base64 -i appsettings.Production.json 
base64 -i .secrets
```

2. Add these as secrets in your GitHub repository:
- Go to your GitHub repository → Settings → Secrets and variables → Actions
- Add `APPSETTINGS_PRODUCTION_JSON_B64` with the base64 encoded appsettings.Production.json
- Add `ENCODED_SECRETS` with the base64 encoded .secrets

#### 2.3 Deploy Server to ECR

1. Push to the repository to trigger the GitHub workflow, or manually run the workflow from the Actions tab.
2. The workflow will build the Docker image and push it to the ECR repository.

### 3. Deploy AppRunner (Step 3)

After running the deployment workflow and deploying the backed, run the app runner workflow manually in github:
1. Go to your GitHub repo → Click on “Actions”. On the left sidebar, locate “app-runner.yml”.
2. Click “Run workflow” (usually a dropdown button) and confirm.
3. After successful deployment, note down the AppRunner service URL from the outputs.

### 4. Web Configuration and Deployment

#### 4.1 Configure Web Environment

1. Navigate to the web directory

2. Update the `.env.production` file with the AppRunner URL:
```
NEXT_PUBLIC_API_URL=https://your-apprunner-service-url
```

3. Generate the api function from orval for the production environment, and push it to the repository

```
npm run generate-api:prod
```

#### 4.2 Deploy Web to AWS Amplify

1. Log in to the AWS Management Console.
2. Navigate to AWS Amplify.
3. Click "New app" → "Host web app".
4. Connect to your GitHub repository.
5. Configure the build settings as required.
6. Deploy the application.
7. Note down the Amplify application URL once deployment is complete.

### 5. Finalize Server Configuration

1. Update the `appsettings.Production.json` file in the server directory with the Web URL:
```
{
    ...
    "WebURL": {"https://your-amplify-web-url"}
}
```

2. Convert the updated file to base64:
```
cat DevStage.API/appsettings.Production.json | base64
```

3. Update the `APPSETTINGS_PRODUCTION_JSON_B64` secret in your GitHub repository.

4. Trigger a new server deployment to apply the changes.

## CI/CD Information

The application has continuous integration and deployment set up:

- **Server**: Any push to the server directory will trigger the GitHub Actions workflow that builds and deploys the new image to ECR. AppRunner will automatically detect the new image and deploy it.

- **Web**: Any push to the web directory will trigger AWS Amplify to build and deploy the new version of the web application.

## Troubleshooting

- For server deployment issues, check the GitHub Actions workflow logs.
- For AppRunner issues, check the AWS AppRunner console and service logs.
- For web deployment issues, check the AWS Amplify build logs.


## Destroy workflow:
1. In the infra submodule, execute workflows/destroy.yml manually to destroy all infra and server resources.
2. In AWS Amplify, App settings, click on Delete app 
3. Manually delete the s3 bucket and the initial policy created in step 1.1.2
