# amplify-project-customization

## Add typescript support for function

### Options

* Use function build option https://docs.amplify.aws/cli/function/build-options
* Use a custom plugin

### Custom plugin version

For that one I simply copied the official  [amplify-nodejs-function-runtime-provider](https://github.com/aws-amplify/amplify-cli/tree/master/packages/amplify-nodejs-function-runtime-provider) stripped all the dependencies I didn't needed and fixed the code that is preventing `amplify function build` and therefore `amplify push`, to support Typescript build:

https://github.com/aws-amplify/amplify-cli/blob/master/packages/amplify-nodejs-function-runtime-provider/src/utils/legacyBuild.ts#L15 

=> 

https://github.com/flochaz/amplify-project-customization/blob/main/plugin/amplify-nodejs-function-runtime-provider/src/utils/legacyBuild.ts#L23

### Usage

1. build the plugins
    ```
    cd plugin/amplify-nodejs-function-runtime-provider
    npm install
    npm run build
    cd ../
    ```
1. Register the plugin (make sure to always answer Y to questions)
    ```
    amplify plugin add plugin/amplify-nodejs-function-runtime-provider

    Plugin package found.
    ? Confirm to add the plugin package to your Amplify CLI. Yes
    Successfully added plugin package.
    ? Run a fresh scan for plugins on the Amplify CLI pluggable platform Yes
    Scanning for plugins...
    Plugin scan successful
    ```
1. add function
    ```
    amplify add function 
    ? Select which capability you want to add: Lambda function (serverless function)
    ? Provide an AWS Lambda function name: myFutureTypescriptFunction
    ? Choose the runtime that you want to use: (Use arrow keys)
      .NET Core 3.1 
      Go 
      Java 
    ❯ NodeJS 
      Python 
    ? Choose the function template that you want to use: (Use arrow keys)
      CRUD function for DynamoDB (Integration with API Gateway) 
    ❯ Hello World 
      Lambda trigger 
      Serverless ExpressJS function (Integration with API Gateway) 
    ```
1. Transform the generated code to typescript project
    1. create tsconfig file
        ```
        cd amplify/backend/function/myFutureTypescriptFunction/src 
        tsc --init
        ```
    1. Move to TS
        ```
        mv index.js intex.ts
        ```
    1. Add build step to packages.json
        ```
        {
          "name": "myFutureTypescriptFunction",
          "version": "2.0.0",
          "description": "Lambda function generated by Amplify",
          "main": "index.js",
          "license": "Apache-2.0",
          "scripts": {
            "build": "tsc"
          }
        }
        ```
    1. Add .gitignore
        ```
        # dependencies
        /node_modules
        
        # Built file
        *.js
        ```
    1. Test your build, It should failed to to missing explicit type
        ```
        npm run build

        > tsc

        intex.ts:3:26 - error TS7006: Parameter 'event' implicitly has an 'any' type.

        3 exports.handler = async (event) => {
                                  ~~~~~
        ```
    1. fix index.ts
        ```
        exports.handler = async (event: any) => {
          ...
        ```
    1. Test amplify build
        ```
        cd - # back to your project root
        amplify function build myFutureTypescriptFunction
        ```
    1. Push it !
        ```
        amplify push
        ```


PS: This method can be used as well to extend/modify standard function template as well by forking [amplify-nodejs-function-template-provider](https://github.com/aws-amplify/amplify-cli/tree/master/packages/amplify-nodejs-function-template-provider)


## Add DynamoDB seeder


### Options

* Dedicated Plugin
    * Doc: https://docs.amplify.aws/cli/plugins/authoring
* Custom CF category
    * Doc: https://docs.amplify.aws/cli/usage/customcf#n2-modify-folder-structure-for-custom-resource
* Amplify CRON function with exact date and time in the near future
    * Doc: https://docs.amplify.aws/cli/function#schedule-recurring-lambda-functions


### Custom CF resource version

1. Init project
    ```
    mkdir newProject
    cd newProject
    amplify init
    ```
1. add api (here called **myapi**)

    ```
    amplify add api

    ? Please select from one of the below mentioned services: GraphQL
    ? Provide API name: myapi
    ? Choose the default authorization type for the API API key
    ? Enter a description for the API key: 
    ? After how many days from now the API key should expire (1-365): 7
    ? Do you want to configure advanced settings for the GraphQL API No, I am done.
    ? Do you have an annotated GraphQL schema? No
    ? Choose a schema template: Single object with fields (e.g., “Todo” with ID, name, descripti
    on)

    The following types do not have '@auth' enabled. Consider using @auth with @model
            - Todo
    Learn more about @auth here: https://docs.amplify.aws/cli/graphql-transformer/auth


    GraphQL schema compiled successfully.

    Edit your schema at /Users/chazalf/amazon/specreqs/amplify/newProject/amplify/backend/api/myapi/schema.graphql or place .graphql files in a directory at /Users/chazalf/amazon/specreqs/amplify/newProject/amplify/backend/api/myapi/schema
    ? Do you want to edit the schema now? No
    Successfully added resource myapi locally

    Some next steps:
    "amplify push" will build all your local backend resources and provision it in the cloud
    "amplify publish" will build all your local backend and frontend resources (if you have hosting category added) and provision it in the cloud
    ```

1. Create custom category folder structure

    ```
    mkdir -p amplify/backend/seeder/dynamodbseeder
    ```

1. Add amplify/backend/seeder/dynamodbseeder/parameters.json 
    1. => **The first parameter of the `Fn::GetAtt` function needs to be `api<YOUR API NAME CHOSEN BEFORE LOWER CASED>`**
    2. That is were you can put your seeding information as **escaped JSON array** under **DataSeeds**

    ```
    {
      "GraphQLAPIId": {
      "Fn::GetAtt": [
                "apimyapi",
                "Outputs.GraphQLAPIIdOutput"
            ]
    },
      "DataSeeds": "[{ \"id\": \"Clean your room\"}]"
    }
    ```

1. Add **amplify/backend/seeder/dynamodbseeder/template.yaml**

    ```
    AWSTemplateFormatVersion: "2010-09-09"

    Parameters:
      GraphQLAPIId:
        Type: String
      env:
        Type: String
      DataSeeds:
        Description: The data  to ingest into DDB
        Type: String
        MinLength: 1
        MaxLength: 255

    Resources:
      LambdaExecutionRole:
        Type: AWS::IAM::Role
        Properties:
          AssumeRolePolicyDocument:
            Version: '2012-10-17'
            Statement:
            - Effect: Allow
              Principal:
                Service:
                - lambda.amazonaws.com
              Action:
              - sts:AssumeRole
          Path: "/"
          Policies:
          - PolicyName: LambdaExecutionRole
            PolicyDocument:
              Version: '2012-10-17'
              Statement:
              - Effect: Allow
                Action:
                - logs:CreateLogGroup
                - logs:CreateLogStream
                - logs:PutLogEvents
                - dynamodb:*
                Resource: 
                - "*"

      LambdaFunction:
        Type: "AWS::Lambda::Function"
        Properties:
          Code:
            ZipFile: |
              const AWS = require('aws-sdk')
              const https = require("https")
              const url = require("url")
              exports.handler = async function (event, context) {
                console.log(event)
                console.log(context)
                if (event.RequestType === 'Delete') {
                  sendResponse(event, context, 'SUCCESS', { data:'data'})
                }
                try {
                  const targetTable = event.ResourceProperties.TargetTable
                  const dataSeeds = event.ResourceProperties.DataSeeds
                  await writeDynamoData(dataSeeds, targetTable)
                  await sendResponse(event, context, 'SUCCESS', { data:'data'})
                } catch(e) {
                  console.log('Failed to write data', e)
                  sendResponse(event, context, 'FAILED', { data:'data'})
                } finally {
                  console.log('done I am')
                }            
              }
              async function writeDynamoData(dataSeeds, targetTable) {
                const ingestData = JSON.parse(dataSeeds)
                console.log("Ingesting data into " + targetTable + ":")
                const documentClient = new AWS.DynamoDB.DocumentClient();
                try {
                  return Promise.all(ingestData.map(async (element) => {
                    console.log(`element ${JSON.stringify(element)}`)
                    await documentClient.put({
                      TableName:targetTable,
                      Item:{
                        ...element
                      }
                    }).promise()
                  }))
                } catch (e) {
                  console.log('Error ingesting data:')
                  console.log(e)
                }
              }
              async function sendResponse(event, context, responseStatus, responseData) {
                  var responseBody = JSON.stringify({
                      Status:responseStatus,
                      Reason:"See the details in CloudWatch Log Stream:"+context.logStreamName,
                      PhysicalResourceId:context.logStreamName,
                      StackId:event.StackId,
                      RequestId:event.RequestId,
                      LogicalResourceId:event.LogicalResourceId,
                      Data:responseData
                  })
                  console.log("RESPONSE BODY:\n", responseBody)
                  var parsedUrl = url.parse(event.ResponseURL)
                  var options = {
                    hostname:parsedUrl.hostname,
                    port:443,
                    path:parsedUrl.path,
                    method:"PUT",
                    headers:{
                      "content-type":"",
                      "content-length":responseBody.length
                    }
                  }
                  console.log("SENDING RESPONSE...\n");
                  return makeRequest(options, responseBody)
                }
                function makeRequest(options, body) {
                  return new Promise((resolve, reject) => {
                    const request = https.request(options, response => {
                      console.log("STATUS:" + response.statusCode)
                      console.log("HEADERS:" + JSON.stringify(response.headers))
                      resolve()
                    })
                    request.on("error", error => {
                      console.log("sendResponse Error:" + error)
                      reject()
                    })
                    request.write(body)
                    request.end()
                  })
                }
          Description: Lambda function that ingests data into a DynamoDB table.
          Handler: index.handler
          Role : 
            Fn::GetAtt: LambdaExecutionRole.Arn
          Runtime: nodejs12.x
          Timeout: 5

      DataIngestLambdaTrigger:
        Type: Custom::DataIngestLambdaTrigger
        Properties:
            ServiceToken: 
              Fn::Sub: 'arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:${LambdaFunction}'
            TargetTable: 
              Fn::ImportValue: 
                Fn::Sub: '${GraphQLAPIId}:GetAtt:TodoTable:Name'
            DataSeeds: 
              Ref: DataSeeds
              
    ```

1.  Register your new category into amplify 
    1. by adding the category in **amplify/backend/backend-config.json**
        ```
        "seeder": {
            "dynamodbseeder": {
              "service": "Seeder",
              "providerPlugin": "awscloudformation"
            }
          }
        ```
        final result should be
        ```
        {
          "api": {
            "myapi": {
              "service": "AppSync",
              "providerPlugin": "awscloudformation",
              "output": {
                "authConfig": {
                  "defaultAuthentication": {
                    "authenticationType": "API_KEY",
                    "apiKeyConfig": {
                      "apiKeyExpirationDays": 7,
                      "description": ""
                    }
                  },
                  "additionalAuthenticationProviders": []
                }
              }
            }
          },
          "seeder": {
            "dynamodbseeder": {
              "service": "Seeder",
              "providerPlugin": "awscloudformation"
            }
          }
        } 
        ```
    1. Forcing a scan 

      ```
      amplify env checkout dev**

      amplify status
      
      Current Environment: dev

      | Category | Resource name  | Operation | Provider plugin   |
      | -------- | -------------- | --------- | ----------------- |
      | Api      | myapi          | Create    | awscloudformation |
      | Seeder   | dynamodbseeder | Create    | awscloudformation |
      ```

1. Deploy it
    ```
    amplify push
    ```

