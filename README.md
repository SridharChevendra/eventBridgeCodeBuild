# Event Bridge CodeBuild Choreography

Using AWS EventBridge, AWS CodeBuild to build scalable artifacts
Refer article for more details on the architecture  https://www.linkedin.com/pulse/event-bridge-codebuild-choreography-sridhar-chevendra?trk=public_profile_article_view

![Proposed Architecture_Choreography](https://github.com/dssc1/eventBridgeCodeBuild/assets/32348488/a809d49a-bb19-4f5d-b80a-4257466acfec)


## Getting Started
### Pre-requisites
* AWS Account
* AWS Cloud9 IDE
Note: If Cloud9 IDE terminal has error message temporary credentials expired, 
1. Disable it by Preferences --> AWS Settings --> Credentials --> AWS Managed Temporary Credentials
2. Create and attach AWS role with appropriate permissions.
3. Once roles are created, change it to the default settings
#### Build Steps
#### 1. Clone GitRepo
Open the new Terminal with in the Cloud9 IDE environment.

Create a folder names "eventBridge"

```
mkdir eventBridge && cd eventBridge

```
Clone the GitRepo to your local machine.
#### 2. Environment Variables 

Set the following Variables for AWS CLI.

```
export AWS_REGION=us-east-1                   # Replace it with your region
export AWS_ACCOUNT_ID=111122221111            # Replace it with your AWS account ID
export REPONAME=eb-${AWS_ACCOUNT_ID}          # ECR Repository name 
export CODEBUILD_ROLENAME=eb-codebuild-${AWS_ACCOUNT_ID}   #AWS Codebuild role name
export S3_BUCKET=eb-${AWS_ACCOUNT_ID}         # S3 bucket for manifest files
export EVENTBRIDGE_ROLENAME=eb-s3events-codebuild-${AWS_ACCOUNT_ID}    #EventBridge Role to invoke codebuild
```
#### 3. AWS Code Repository

It is a repository to host your code privately with in your AWS environment. In the subsequent steps, AWS CodeBuild build process uses the code in resposity to build artifacts.

```
aws codecommit create-repository --region ${AWS_REGION} --repository-name ${REPONAME} --output json

Output:
{
    "repositoryMetadata": {
        "repositoryName": "<REPONAME>", 
        "cloneUrlSsh": "ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/<REPONAME>", 
        "lastModifiedDate": 1684294898.421, 
        "repositoryId": "9996841a-8e68-426e-8294-ce7acc774e41", 
        "cloneUrlHttp": "https://git-codecommit.us-east-1.amazonaws.com/v1/repos/<<REPONAME>>", 
        "creationDate": 1684294898.421, 
        "Arn": "arn:aws:codecommit:us-east-1:<AWS_ACCOUNT_ID>:<REPONAME>", 
        "accountId": "<AWS_ACCOUNT_ID>"
    }
}
```
Note: Take a note of the "cloneUrlSsh" for future reference

#### 4. Create Code Build IAM roles
This role enables AWS CodeBuild to interact with other services.
```
cd eventBridgeCodeBuild

aws iam create-role --role-name ${CODEBUILD_ROLENAME} --assume-role-policy-document file://Resources/role-policy.json 
 
# To add an inline permissions policy and permissions

aws iam put-role-policy --role-name ${CODEBUILD_ROLENAME} --policy-name ${CODEBUILD_ROLENAME}_policy --policy-document file://Resources/policy.json 
```
#### 5. Setup AWS CodeBuild Projects
For this usecase, create 3 codebuild projects. Each project is a triggered by a corresponding manifest file and uses respective python environment.

######  ManifestFileName -->     CodeBuildProject -->  PythonEnv
  
manifest8.json   -->    codebuild-project8 -->     python3.8
  
manifest9.json  -->     codebuild-project9 -->    python3.9
  
manifest10.json -->      codebuild-project10 -->    python3.10  

```
# Update the input files with the respective environment variables

# CodeBuild Project8

sed -i -e "s/<CODEBUILD_ROLENAME>/${CODEBUILD_ROLENAME}/g" \
-e "s/<REPONAME>/${REPONAME}/g" \
-e "s/<AWS_ACCOUNT_ID>/${AWS_ACCOUNT_ID}/g" \
-e "s/<AWS_REGION>/${AWS_REGION}/g" Resources/codebuild-project8.json


aws codebuild create-project --region ${AWS_REGION} --cli-input-json file://Resources/codebuild-project8.json

# CodeBuild Project9

sed -i -e "s/<CODEBUILD_ROLENAME>/${CODEBUILD_ROLENAME}/g" \
-e "s/<REPONAME>/${REPONAME}/g" \
-e "s/<AWS_ACCOUNT_ID>/${AWS_ACCOUNT_ID}/g" \
-e "s/<AWS_REGION>/${AWS_REGION}/g" Resources/codebuild-project9.json

aws codebuild create-project --region ${AWS_REGION} --cli-input-json file://Resources/codebuild-project9.json

# CodeBuild Project10

sed -i -e "s/<CODEBUILD_ROLENAME>/${CODEBUILD_ROLENAME}/g" \
-e "s/<REPONAME>/${REPONAME}/g" \
-e "s/<AWS_ACCOUNT_ID>/${AWS_ACCOUNT_ID}/g" \
-e "s/<AWS_REGION>/${AWS_REGION}/g" Resources/codebuild-project10.json

aws codebuild create-project --region ${AWS_REGION} --cli-input-json file://Resources/codebuild-project10.json  
```

#### 6. Upload to AWS CodeCommit
  To Upload the updated configuration file, clone AWS code commit repository. Copy the "cloneUrlSsh" from Step 3.
  ```
  git clone <<cloneUrlSsh>>
  ```
Copy the file from the code folder to gitclone 

```
cp Code/* eb-${AWS_ACCOUNT_ID}
cd eb-${AWS_ACCOUNT_ID}

# Commit and push the changes to Code commit respository
git add .
git commit -m "First Commit"
git push
```
#### 7. Setup manifest repository
create s3 bucket the manifest repository. With the Eventbridge enablement on S3, When manifest files are added, it will generate an event into Amazon Eventbridge default Bus.
```
  aws s3api create-bucket \
    --bucket ${S3_BUCKET} \
    --region ${AWS_REGION}
  ```
  Enable S3 Eventbridge notification with the following CLI. There are some known issues based AWS CLI version.
  If that is the case, login to the console, S3 Service, Go to <<S3 BUCKETNAME>> properties,Amazon EventBridge to Enable Event bridge notification.
  
  ```
aws s3api put-bucket-notification-configuration --bucket ${S3_BUCKET} --notification-configuration='{ "EventBridgeConfiguration": {} }'
  ```
  
#### 8.  Create AWS EventBridge Roles
Create a role and attach policy
```
cd ~/environment/eventBridge
    
aws iam create-role --role-name ${EVENTBRIDGE_ROLENAME} --assume-role-policy-document file://Resources/eventbridge-role.json 

sed -i -e "s/<AWS_ACCOUNT_ID>/${AWS_ACCOUNT_ID}/g" \
-e "s/<AWS_REGION>/${AWS_REGION}/g" Resources/eventbridge-role-attach-policy.json
    
aws iam put-role-policy --role-name ${EVENTBRIDGE_ROLENAME} --policy-name ${EVENTBRIDGE_ROLENAME}_policy --policy-document file://Resources/eventbridge-role-attach-policy.json 
```
#### 9. EventRules
  ```

sed -i -e "s/<AWS_ACCOUNT_ID>/${AWS_ACCOUNT_ID}/g" \
-e "s/<S3_BUCKET>/${S3_BUCKET}/g" Resources/eventbridge-rule-project8.json
    
sed -i -e "s/<AWS_ACCOUNT_ID>/${AWS_ACCOUNT_ID}/g" \
-e "s/<S3_BUCKET>/${S3_BUCKET}/g" Resources/eventbridge-rule-project9.json
    
sed -i -e "s/<AWS_ACCOUNT_ID>/${AWS_ACCOUNT_ID}/g" \
-e "s/<S3_BUCKET>/${S3_BUCKET}/g" Resources/eventbridge-rule-project10.json
 
  # Event Bridge Rules based on the event pattern definitions
    
  aws events put-rule --name "codebuild-project8" --region ${AWS_REGION} --event-pattern file://Resources/eventbridge-rule-project8.json
  aws events put-rule --name "codebuild-project9" --region ${AWS_REGION} --event-pattern file://Resources/eventbridge-rule-project9.json
  aws events put-rule --name "codebuild-project10" --region ${AWS_REGION} --event-pattern file://Resources/eventbridge-rule-project10.json
  ```
#### 10. Assign a Target for this rule to execute 
```
# Once an event rule matches a defined manifest file, it invokes associated Target.
# Assigning codebuild-project8 as Target
aws events put-targets --rule "codebuild-project8" --targets "Id"=1,"Arn"="arn:aws:codebuild:us-east-1:${AWS_ACCOUNT_ID}:project/codebuild-project8","RoleArn"="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${EVENTBRIDGE_ROLENAME}" --region ${AWS_REGION} --output json

# Assigning codebuild-project9 as Target
aws events put-targets --rule "codebuild-project9" --targets "Id"=1,"Arn"="arn:aws:codebuild:us-east-1:${AWS_ACCOUNT_ID}:project/codebuild-project9","RoleArn"="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${EVENTBRIDGE_ROLENAME}" --region ${AWS_REGION} --output json

# Assigning codebuild-project10 as Target
    
aws events put-targets --rule "codebuild-project10" --targets "Id"=1,"Arn"="arn:aws:codebuild:us-east-1:${AWS_ACCOUNT_ID}:project/codebuild-project9","RoleArn"="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${EVENTBRIDGE_ROLENAME}" --region ${AWS_REGION} --output json
```
#### 11.Tests
Use the manifest files in code folder and upload them to S3 bucket.
    
    Once the files are uploaded, Go to AWS Console, AWS codebuild, CodeBuild projects to see the projects exected.
    Refer to the CodeBuild Project Logs. Based on the project name, it displays the image's python version.
```
    aws s3 cp Code/manifest8.json s3://S3_BUCKET
    aws s3 cp Code/manifest9.json s3://S3_BUCKET
    aws s3 cp Code/manifest10.json s3://S3_BUCKET
```

