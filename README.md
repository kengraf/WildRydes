# WildRydes
Command line instructions for AWS' WildRydes serverless demo

We are using this repo to learn about a basic Cloud serverless deployment.

This deployment leverages a database tier (AWS DynamoDB), application (Lambda), Cognito(Identity), Amplify w/ CodeCommit (code management), and REST front end (API Gateway).

### IAM
Create HTTPS Git credentials for AWS CodeCommit
```
aws iam create-service-specific-credential --user-name Ken --service-name codecommit.amazonaws.com
```
# Note retain the ServiceUserName and ServicePassword for later use with git commands
aws iam list-service-specific-credentials --user-name Ken
DELETE
```
ID=`aws iam list-service-specific-credentials --user-name Ken --service-name codecommit.amazonaws.com --output text  --query "ServiceSpecificCredentials[].ServiceSpecificCredentialId" `
aws iam delete-service-specific-credential --user-name Ken  --service-specific-credential-id $ID
```

### CodeCommit
```
URL=`aws codecommit create-repository --repository-name "WildRydes" --output text  --query "repositoryMetadata.cloneUrlHttp" `
git clone $URL
cd WildRydes
# Store Codecommit username/password to prevent future prompting
git config --global credential.helper store
git pull # Prompted for creds
git pull # not
# Load demo code from S3
aws s3 cp s3://wildrydes-us-east-1/WebApplication/1_StaticWebHosting/website ./ --recursive
$ git add .
$ git commit -m 'new'
$ git push
```

DELETE
```
aws codecommit delete-repository --repository-name "WildRydes"
```

### Amplify
```
ID=`aws amplify create-app --name "WildRydes" --repository $URL --platform WEB --iam-service-role-arn  --enable-branch-auto-build --output text --query "app.appId" `
aws amplify create-branch --app-id $ID --branch-name master
aws amplify create-deployment --app-id $ID --branch-name master
arn:aws:iam::788715698479:role/service-role/AWSAmplifyCodeCommitExecutionRole-d2w52rz60jdyl8
```
DELETE
```
aws amplify delete-app --app-id $ID
```
“new web app”, select CodeCommit repo, accept default build configuration.
Cognito
Manage User Pool | Create User Pool
Name pool ‘WildRydes’
Review defaults and click Create
```
POOLID=`aws cognito-idp   create-user-pool --pool-name WildRydes2 --output text --query “UserPool.Id” `
POOLID=`aws cognito-idp list-user-pools --max-results 10 --output text --query “UserPools[?Name==’WildRydes2’].Id`
aws cognito-idp delete-user-pool –user-pool-id $POOLID
```
Cognito add app client
CLIENTID=`aws cognito-idp create-user-pool-client --user-pool-id $POOLID --no-allowed-o-auth-flows-user-pool-client --client-name WildRydesApp --output text --query “UserPoolClient.ClientId” `
sed -i "/userPoolId:/ s/'.*'/'$POOLID'/" js/config.js
sed -i "/userPoolClientId:/ s/'.*'/'$CLIENTID'/" js/config.js

----------------------------------------- cut here --------------------

### DynamoDB: used to store your friends
```
# Create a new table named `EenyMeenyMinyMoe`
aws dynamodb create-table \
    --table-name EenyMeenyMinyMoe \
    --attribute-definitions AttributeName=Name,AttributeType=S  \
    --key-schema AttributeName=Name,KeyType=HASH  \
    --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
    
```
    
```
# Add friend records for testing.  
friends=("Alice" "Bob" "Charlie")
for i in "${friends[@]}"
do
   : 
  aws dynamodb put-item --table-name EenyMeenyMinyMoe --item \
    '{ "Name": {"S": "'$i'"} }' 
done

```

### Lambda: used to select a friend
AWS CLI to create a Lambda function require files for packages, roles, and policies.  The example here assumes you have cloned this Github repo and are in the proper working directory

```
# Create role for Lambda function
aws iam create-role --role-name EenyMeenyMinyMoe \
    --assume-role-policy-document file://lambdatrustpolicy.json
```
```
# Attach policy for DynamoDB access to role
aws iam put-role-policy --role-name EenyMeenyMinyMoe \
    --policy-name EenyMeenyMinyMoe \
    --policy-document file://lambdapolicy.json
ARN=`aws iam list-roles --output text \
    --query "Roles[?RoleName=='EenyMeenyMinyMoe'].Arn" `
```
```
# Create Lambda
zip function.zip -xi index.js
aws lambda create-function --function-name EenyMeenyMinyMoe \
    --runtime nodejs14.x --role $ARN \
    --zip-file fileb://function.zip \
    --runtime nodejs14.x --handler index.handler
```
```
# Give the API Gateway permission to invoke the Lambda
aws lambda add-permission \
    --function-name EenyMeenyMinyMoe \
    --action lambda:InvokeFunction \
    --statement-id AllowGateway \
    --principal apigateway.amazonaws.com
```

### API Gateway
```
# Create the Gateway
aws apigateway create-rest-api --name 'EenyMeenyMinyMoe' \
    --endpoint-configuration types=REGIONAL
```

```
# Create a GET method for a Lambda-proxy integration
APIID=`aws apigateway get-rest-apis --output text \
    --query "items[?name=='EenyMeenyMinyMoe'].id" `
PARENTID=`aws apigateway get-resources --rest-api-id $APIID \
    --query 'items[0].id' --output text`
aws apigateway put-method --rest-api-id $APIID \
    --resource-id $PARENTID --http-method GET \
    --authorization-type "NONE"
            
# Create integration with Lambda
ARN=`aws lambda get-function --function-name EenyMeenyMinyMoe \
    --query Configuration.FunctionArn --output text`
REGION=`aws ec2 describe-availability-zones --output text \
    --query 'AvailabilityZones[0].[RegionName]'`
URI='arn:aws:apigateway:'$REGION':lambda:path/2015-03-31/functions/'$ARN'/invocations'
aws apigateway put-integration --rest-api-id $APIID \
   --resource-id $PARENTID --http-method GET --type AWS_PROXY \
   --integration-http-method POST --uri $URI
aws apigateway put-integration-response --rest-api-id $APIID \
    --resource-id $PARENTID --http-method GET \
    --status-code 200 --selection-pattern "" 

# Push out deployment
aws apigateway create-deployment --rest-api-id $APIID --stage-name prod
```

### Run the game.  Each refresh will return a different name.
```
curl -v https://$APIID.execute-api.us-east-2.amazonaws.com/prod/
```

### Clean Up by removing all the resources created
```
# Delete API Gateway
APIID=`aws apigateway get-rest-apis --output text \
    --query "items[?name=='EenyMeenyMinyMoe'].id" `
aws apigateway delete-rest-api --rest-api-id $APIID

# Delete Lambda function
aws lambda delete-function --function-name EenyMeenyMinyMoe

# Delete DynamoDB table
aws dynamodb delete-table --table-name EenyMeenyMinyMoe

# Delete Role and Policy
aws iam delete-role-policy --role-name EenyMeenyMinyMoe \
    --policy-name EenyMeenyMinyMoe
aws iam delete-role --role-name EenyMeenyMinyMoe 
```

### Extra credit
- Error handling
- Monitoring and alerts
- Add authorization to the API using Cognito
- Use Route53 to provide a friendly domain name for the APIGateway
- Expand the API to allow adding and removing names
