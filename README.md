# Multi-branch CodePipeline strategy with event-driven architecture

*Henrique Bueno, DevOps Consultant, Professional Services*

This blog post presents a solution for automated pipelines creation in AWS CodePipeline when a new branch is created in an AWS CodeCommit repository. A use case for this solution is when a GitFlow approach using CodePipeline is required. The strategy presented here is used by AWS customers to enable the use of GitFlow using only AWS tools.

CodePipeline is a fully managed continuous delivery service that helps you automate your release pipelines for fast and reliable application and infrastructure updates. CodePipeline automates the build, test, and deploy phases of your release process every time there is a code change, based on the release model you define.

CodeCommit is a fully managed source control service that hosts secure Git-based repositories. It makes it easy for teams to collaborate on code in a secure and highly scalable ecosystem.

GitFlow is a branching model designed around the project release. This provides a robust framework for managing larger projects. Gitflow is ideally suited for projects that have a scheduled release cycle.



## Applicability

When using CodePipeline to orchestrate pipelines and CodeCommit as a code source, in addition to setting a repository, you must also set which branch will trigger the pipeline. This configuration works perfectly for the trunk-based strategy, in which you have only one main branch and all the developers interact with this single branch. However, when you need to work with a multi-branching strategy like GitFlow, the requirement to set a pipeline for each branch brings additional challenges.

It’s important to note that trunk-based is, by far, the best strategy for taking full advantage of a DevOps approach; this is the branching strategy that AWS recommends to its customers. On the other hand, many customers like to work with multiple branches and believe it justifies the effort and complexity in dealing with branching merges. This solution is for these customers.

## Solution Overview

One of the great benefits of working with Infrastructure as Code is the ability to create multiple identical environments through a single template. This example uses AWS CloudFormation templates to provision pipelines and other necessary resources, as shown in the following diagram.

The template is hosted in an Amazon S3 bucket. An AWS Lambda function deploys a new AWS CloudFormation stack based on this template. This Lambda function is trigged for an Amazon CloudWatch Events rule that looks for events at the CodePipeline repository.

### The CreatePipeline Events rule

The AWS CloudFormation snippet that creates the Events rule follows. The Events rule monitors create and delete branches events in all repositories, triggering the CreatePipeline Lambda function.

`````
#----------------------------------------------------------------------#
# EventRule to trigger LambdaPipeline lambda
#----------------------------------------------------------------------#
  CreatePipelineRule:
    Type: AWS::Events::Rule
    Properties: 
      Description: "EventRule"
      EventPattern:
        source:
          - aws.codecommit
        detail-type:
          - 'CodeCommit Repository State Change'
        detail:
          event:
              - referenceDeleted
              - referenceCreated
          referenceType:
            - branch
      State: ENABLED
      Targets: 
      - Arn: !GetAtt CreatePipeline.Arn
        Id: CreatePipeline
`````

### The CreatePipeline Lambda function

The Lambda function receives the event details, parses the variables, and executes the appropriate actions. If the event is referenceCreated, then the stack is created; otherwise the stack is deleted. The stack name created or deleted is the junction of the repository name plus the new branch name. This is a very simple function.

`````
#----------------------------------------------------------------------#
# Lambda for Stack Creation
#----------------------------------------------------------------------#
import boto3
def lambda_handler(event, context):
    Region = event['region']
    Account = event['account']
    RepositoryName = event['detail']['repositoryName']
    NewBranch = event['detail']['referenceName']
    Event = event['detail']['event']
    if NewBranch == "main":
        quit()
    if Event == "referenceCreated":
        cf_client = boto3.client('cloudformation')
        cf_client.create_stack(
            StackName=f'Pipeline-{RepositoryName}-{NewBranch}',
            TemplateURL=f'https://s3.amazonaws.com/{Account}-templates/TemplatePipeline.yaml',
            Parameters=[
                {
                    'ParameterKey': 'RepositoryName',
                    'ParameterValue': RepositoryName,
                    'UsePreviousValue': False
                },
                {
                    'ParameterKey': 'BranchName',
                    'ParameterValue': NewBranch,
                    'UsePreviousValue': False
                }
            ],
            OnFailure='ROLLBACK',
            Capabilities=['CAPABILITY_NAMED_IAM']
        )
    else:
        cf_client = boto3.client('cloudformation')
        cf_client.delete_stack(
            StackName=f'Pipeline-{RepositoryName}-{NewBranch}'
        )
`````

The logic for creating only the CI or the CI+CD is on the AWS CloudFormation template. The Conditions section of AWS CloudFormation analyzes the new branch name.

`````
Conditions: 
    Branchmain: !Equals [ !Ref BranchName, "main" ]
    BranchDevelop: !Equals [ !Ref BranchName, "develop"]
    Setup: !Equals [ !Ref Setup, true ]
`````

* If the new branch is named main, then a stack will be created containing CI+CD pipelines, with deploy stages in the homologation and production environments.
* If the new branch is named develop, then a stack will be created containing CI+CD pipelines, with a deploy stage in the Dev environment.
* If the new branch has any other name, then the stack will be created with only a CI pipeline.

*NOTE: Since the purpose of this blog post is to present only a sample of automated pipelines creation, the pipelines used here are for examples only: they don't deploy to any environment.*



## Applicability

This event-driven strategy permits pipelines to be created or deleted along with the branches. Since the entire environment is created using Infrastructure as Code and the template is the same for all pipelines, there is no possibility of different configuration issues between environments and pipeline stages.

A GitFlow simulation could resemble that shown in the following diagram:


1. First, a CodeCommit repository is created, along with the main branch and its respective pipeline (CI+CD).
2. The developer creates a branch called develop based on the main branch. The pipeline (CI+CD at Dev) is automatically created.
The developer creates a feature-branch called feature-a based on the develop branch. The CI pipeline for this branch is automatically created.
3. The developer creates a Pull Request from the feature-a branch to the develop branch. As soon as the Pull Request is accepted and merged and the feature-a branch is deleted, its pipeline is automatically deleted.
4. The same process can be followed for the release branch and hotfix branch. Once the branch is created, a new pipeline is created for it which follows its branch lifecycle.



## Implementation

Before you start, make sure that the AWS CLI is installed and configured on your machine by following these steps:

1. Clone the repository.
2. Create the prerequisites stack.
3. Copy the AWS CloudFormation template to Amazon S3.
4. Copy the seed.zip file to the Amazon S3 bucket.
5. Create the first repository and its pipeline.
6. Create the develop branch.
7. Create the first feature branch.
8. Create the first Pull Request.
9. Execute the Pull Request approval.
10. Cleanup.


### 1. Clone the repository

Clone the repository with the sample code.

The main files are:
* Setup.yaml: an AWS CloudFormation template for creating pipeline prerequisites.
* TemplatePipeline.yaml: an AWS CloudFormation template for pipeline creation.
* seed/buildspec/CIAction.yaml: a configuration file for an AWS CodeBuild project at the CI stage.
* seed/buildspec/CDAction.yaml: a configuration file for a CodeBuild project at the CD stage.


`````
# Command to clone the repository
git clone https://github.com/aws-samples/aws-codepipeline-multi-branch-strategy.git
cd aws-codepipeline-multi-branch-strategy
`````


### 2. Create the prerequisite stack

The Setup stack creates the resources that are prerequisites for pipeline creation, as shown in the following chart.

These resources are created only once and they fit all the pipelines created in this example.


`````
# Command to create Setup stack
aws cloudformation deploy --stack-name Setup-Pipeline \
--template-file Setup.yaml --region us-east-1 --capabilities CAPABILITY_NAMED_IAM
`````


### 3. Copy the AWS CloudFormation template to Amazon S3

For the Lambda function to deploy a new pipeline stack, it needs to get the AWS CloudFormation template from somewhere. To enable it to do so, you need to save the template inside the Amazon S3 bucket that you just created at the Setup stack.

`````
# Command that copy Template to S3 Bucket
aws s3 cp TemplatePipeline.yaml s3://"$(aws sts get-caller-identity --query Account --output text)"-templates/ --acl private
`````


### 4. Copy the seed.zip file to the Amazon S3 bucket

CodeCommit permits you to populate a repository at the moment of its creation as a first commit. The content of this first commit can be saved in a .zip file in an Amazon S3 bucket. Use this CodeCommit option to populate your repository with BuildSpec files for CodeBuild.

`````

# Command to create zip file with the Buildspec folder content.
zip -r seed.zip buildspec

# Command that copy seed.zip file to S3 Bucket.
aws s3 cp seed.zip s3://"$(aws sts get-caller-identity --query Account --output text)"-templates/ --acl private
`````


### 5. Create the first repository and its pipeline

Now that the Setup stack is created and the seed file is stored in an Amazon S3 bucket, create the first CodeCommit repository. Every time that you want to create a new repository, execute the command below to create a new stack.

`````
# Command to create the stack with the CodeCommit repository,
# CodeBuild Projects and the Pipeline for the main branch.
# Note: Change "myapp" by the name you want.

RepoName="myapp"
aws cloudformation deploy --stack-name Repo-$RepoName --template-file TemplatePipeline.yaml \
--parameter-overrides RepositoryName=$RepoName Setup=true \
--region us-east-1 --capabilities CAPABILITY_NAMED_IAM
`````

When the stack is created, in addition to the CodeCommit repository, the CodeBuild projects and the main branch pipeline are also created. By default, a CodeCommit repository is created empty, with no branch. When the repository is populated with the seed.zip file, the main branch is created.


Access the CodeCommit repository to see the seed files at the main branch. Access the CodePipeline console to see that there's a new pipeline with the name as the repository. This pipeline contains the CI+CD stages (homolog and prod).



### 6. Create the develop branch

To simulate a real development scenario, create a new branch called develop based on the main branch. In the GitFlow concept these two (main and develop) branches are fixed and never deleted.

When this new branch is created, the Events rule identifies that there's a change on this repository and triggers the CreatePipeline Lambda function to create a new pipeline for this branch. Access the CodePipeline console to see that there's a new pipeline with the name of the repository plus the branch name. This pipeline contains the CI+CD stages (Dev).

`````
# Configure Git Credentials using AWS CLI Credential Helper
mkdir myapp
cd myapp
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true

# Clone the CodeCommit repository
# You can get the URL in the CodeCommit Console
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/myapp .

# Create the develop branch
# For more details: https://docs.aws.amazon.com/codecommit/latest/userguide/how-to-create-branch.html
git checkout -b develop
git push origin develop
`````


### 7. Create the first feature branch

Now that there are two main and fixed branches (main and develop), you can create a feature branch. In the GitFlow concept, feature branches have a short lifetime and are frequently merged to the develop branch. This type of branch only exists during the development period. When the feature development is finished, it is merged to the develop branch and the feature branch is deleted.

`````
# Create the feature-branch branch
# make sure that you are at develop branch
git checkout -b feature-abc
git push origin feature-abc
`````

Just as with the develop branch, when this new branch is created, the Events rule triggers the CreatePipeline Lambda function to create a new pipeline for this branch. Access the CodePipeline console to see that there's a new pipeline with the name of the repository plus the branch name. This pipeline contains only the CI stage, without a CD stage.



### 8. Create the first Pull Request

It’s possible to simulate the end of a feature development, when the branch is ready to be merged with the develop branch. Keep in mind that the feature branch has a short lifecycle:  when the merge is done, the feature branch is deleted along with its pipeline.

`````
# Create the Pull Request
aws codecommit create-pull-request --title "My Pull Request" \
--description "Please review these changes by Tuesday" \
--targets repositoryName=$RepoName,sourceReference=feature-abc,destinationReference=develop \
--region us-east-1
`````


### 9. Execute the Pull Request approval

To merge the feature branch to the develop branch, the Pull Request needs to be approved. In a real scenario, a teammate should do a peer review before approval.

`````
# Accept the Pull Request
# You can get the Pull-Request-ID in the json output of the create-pull-request command
aws codecommit merge-pull-request-by-fast-forward --pull-request-id <PULL_REQUEST_ID_FROM_PREVIOUS_COMMAND> \
--repository-name $RepoName --region us-east-1

# Delete the feature-branch
aws codecommit delete-branch --repository-name $RepoName --branch-name feature-abc --region us-east-1
`````

The new code is integrated with the develop branch. If there's a conflict, it needs to be solved. After that, the featurebranch is deleted together with its pipeline. The Event Rule triggers the CreatePipeline Lambda function to delete the pipeline for its branch. Access the CodePipeline console to see that the pipeline for the feature branch is deleted.


### 10. Cleanup

To remove the resources created as part of this blog post, follow these steps:

#### Delete Pipeline Stacks
`````
# Delete all the pipeline Stacks created by CreatePipeline Lambda
Pipelines=$(aws cloudformation list-stacks --stack-status-filter --region us-east-1 --query 'StackSummaries[? StackStatus==`CREATE_COMPLETE` && starts_with(StackName, `Pipeline`) == `true`].[StackName]' --output text)
while read -r Pipeline rest; do aws cloudformation delete-stack --stack-name $Pipeline --region us-east-1 ; done <<< $Pipelines
`````

#### Delete Repository Stacks
`````
# Delete all the Repository stacks Stacks created by Step 5.
Repos=$(aws cloudformation list-stacks --stack-status-filter --region us-east-1 --query 'StackSummaries[? StackStatus==`CREATE_COMPLETE` && starts_with(StackName, `Repo-`) == `true`].[StackName]' --output text)
while read -r Repo rest; do aws cloudformation delete-stack --stack-name $Repo --region us-east-1 ; done <<< $Repos
`````

#### Delete Setup-Pipeline Stack
`````
# Cleaning Bucket before Stack deletion 
aws s3 rm s3://"$(aws sts get-caller-identity --query Account --output text)"-templates --recursive

# Delete Setup Stack
aws cloudformation delete-stack --stack-name Setup-Pipeline --region us-east-1 
`````



## Conclusion

This blog post discussed how you can work with event-driven strategy and Infrastructure as Code to implement a multi-branch pipeline flow using CodePipeline. It demonstrated how an Events rule and Lambda function can be used to fully orchestrate the creation and deletion of pipelines.



## License

This library is licensed under the MIT-0 License. See the LICENSE file.
