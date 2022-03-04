# WildRydes
Command line instructions for AWS' WildRydes serverless demo
![Original AWS site](https://aws.amazon.com/getting-started/hands-on/build-serverless-web-app-lambda-apigateway-s3-dynamodb-cognito/)
We are using this repo to learn about a basic Cloud serverless deployment.

This deployment leverages a database tier (AWS DynamoDB), application (Lambda), Cognito(Identity), Amplify w/ CodeCommit (code management), and REST front end (API Gateway).

### General environment variables
```
# Note: Environment variables are lost when Cloudshell times out.
APPNAME=WildRydes
USER=`aws sts get-caller-identity --query Arn --output text | cut -d '/' -f 2`

# Uncomment if not using AWS Cloudshell
#AWS_REGION=`aws ec2 describe-availability-zones --output text \
#    --query 'AvailabilityZones[0].[RegionName]'`
```

### IAM
Create HTTPS Git credentials for AWS CodeCommit
```
aws iam create-service-specific-credential --user-name $USER \
    --service-name codecommit.amazonaws.com
# Note retain the ServiceUserName and ServicePassword for later use with git commands
aws iam list-service-specific-credentials --user-name $USER
```
ID=`aws iam list-service-specific-credentials --user-name $USER \
    --service-name codecommit.amazonaws.com --output text  \
    --query "ServiceSpecificCredentials[].ServiceSpecificCredentialId" `
```

### CodeCommit
```
URL=`aws codecommit create-repository --repository-name $APPNAME \
	--output text  --query "repositoryMetadata.cloneUrlHttp" `
git clone $URL
cd $APPNAME
# Store Codecommit username/password to prevent future prompting
git config --global credential.helper store
git pull # Prompted for creds
git pull # not
# Load demo code from S3
aws s3 cp s3://wildrydes-us-east-1/WebApplication/1_StaticWebHosting/website ./ --recursive
# Load AWS code from kengraf/WildRydes
wget https://github.com/kengraf/WildRydes/archive/refs/heads/main.zip
unzip -j main.zip -d .
rm main.zip
git add .
git commit -m 'initial create'
git push
```

### Amplify Console
```
Get started with Web Hosting
Enter name "WildRydes", Click next
Select AWS CodeCommit, Click Next
Select your repo, Click next
Check auto deploy, Click next
Click "Save and Deploy"

Validate app deployed
```

### Cognito
Manage User Pool | Create User Pool
Name pool ‘WildRydes’
Review defaults and click Create
```
POOLID=`aws cognito-idp create-user-pool --pool-name $APPNAME \
	--auto-verified-attributes email --output text --query "UserPool.Id" `
CLIENTID=`aws cognito-idp create-user-pool-client --user-pool-id $POOLID \
	--no-allowed-o-auth-flows-user-pool-client --client-name $APPNAME \
	--output text --query "UserPoolClient.ClientId" `
sed -i "/userPoolId:/ s/'.*'/'$POOLID'/" js/config.js
sed -i "/userPoolClientId:/ s/'.*'/'$CLIENTID'/" js/config.js
sed -i "/region:/ s/'.*'/'us-east-2'/" js/config.js
git add .
git commit -m 'user pool update'
git push
```

Validate Cognito is active, in browser click "Giddy up" registration button.


### DynamoDB
```
# Create a new table
aws dynamodb create-table \
    --table-name Rides \
    --attribute-definitions AttributeName=RideId,AttributeType=S  \
    --key-schema AttributeName=RideId,KeyType=HASH  \
    --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
    
```

### Lambda: used to select a ride
```
# Create role for Lambda function
aws iam create-role --role-name $APPNAME \
    --assume-role-policy-document file://lambdatrustpolicy.json
# Attach policy for DynamoDB access to role
aws iam put-role-policy --role-name $APPNAME --policy-name $APPNAME \
    --policy-document file://lambdapolicy.json
ARN=`aws iam list-roles --output text \
    --query "Roles[?RoleName==${APPNAME}].Arn" `

# Create Lambda
zip function.zip -xi index.js
aws lambda create-function --function-name $APPNAME \
    --runtime nodejs14.x --role $ARN --zip-file fileb://function.zip \
    --runtime nodejs14.x --handler index.handler
# Give the API Gateway permission to invoke the Lambda
aws lambda add-permission --function-name $APPNAME \
    --action lambda:InvokeFunction --statement-id AllowGateway \
    --principal apigateway.amazonaws.com
```
### API Gateway
```
# Create the Gateway
APIID=`aws apigateway create-rest-api --name $APPNAME \
    --endpoint-configuration types=REGIONAL --output text --query "id"`
# Create authorizer
POOLARN=`aws cognito-idp describe-user-pool --user-pool-id $POOLID \
    --output text --query "UserPool.Arn"`
aws apigateway create-authorizer --name $APPNAME --rest-api-id $APIID \
	--type "COGNITO_USER_POOLS" \
    --identity-source 'method.request.header.Authorization' \
    --provider-arns $POOLARN
# Create a GET method for a Lambda-proxy integration
APIID=`aws apigateway get-rest-apis --output text \
    --query "items[?name==${APPNAME}].id" `
PARENTID=`aws apigateway get-resources --rest-api-id $APIID \
	--query 'items[0].id' --output text`
aws apigateway put-method --rest-api-id $APIID --resource-id $PARENTID \
	--http-method POST --authorization-type "NONE"

# Create integration with Lambda
ARN=`aws lambda get-function --function-name $APPNAME \
    --query Configuration.FunctionArn --output text`
URI='arn:aws:apigateway:'$REGION':lambda:path/2015-03-31/functions/'$ARN'/invocations'
aws apigateway put-integration --rest-api-id $APIID \
   --resource-id $PARENTID --http-method POST --type AWS_PROXY \
   --integration-http-method POST --uri $URI
aws apigateway put-integration-response --rest-api-id $APIID \
    --resource-id $PARENTID --http-method POST \
    --status-code 200 --selection-pattern "" 

# Push out deployment
aws apigateway create-deployment --rest-api-id $APIID --stage-name prod

# Update js/config.js with new endpoint
URL="https:\/\/${APIID}.execute-api.${REGION}.amazonaws.com\/prod"
sed -i "/invokeUrl:/ s/'.*'/'$URL'/" js/config.js
git add .
git commit -m 'user pool update'
git push
```

### Run the game.  Each refresh will return a different name.
```
curl -v https://$APIID.execute-api.$REGION.amazonaws.com/prod/
```

# Clean up
By deleting all the resources created
```
# Delete Codecommit repo
aws codecommit delete-repository --repository-name $APPNAME

# Delete Amplify
ID=`aws amplify list-apps --output text --query "apps[?name==${APPNAME}].appId" `
aws amplify delete-app --app-id $ID

# Delete Cognito User pool
POOLID=`aws cognito-idp list-user-pools --max-results 10 \
	--output text --query "UserPools[?Name==${APPNAME}].Id" `
aws cognito-idp delete-user-pool --user-pool-id $POOLID

# Delete API Gateway
APIID=`aws apigateway get-rest-apis --output text \
    --query "items[?name==${APPNAME}].id" `
aws apigateway delete-rest-api --rest-api-id $APIID

# Delete Lambda function
aws lambda delete-function --function-name $APPNAME

# Delete DynamoDB table
aws dynamodb delete-table --table-name $APPNAME

# Delete Role and Policy
aws iam delete-role-policy --role-name $APPNAME \
    --policy-name $APPNAME
aws iam delete-role --role-name $APPNAME

# Remove creds
aws iam delete-service-specific-credential --user-name $USER \
    --service-specific-credential-id $ID

```

### Extra credit
- Monitoring and alerts
- Use Route53 to provide a friendly domain name for the APIGateway
