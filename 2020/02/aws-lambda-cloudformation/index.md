# Automated and Authenticated APIs stack using Lambda, API Gateway and Cloudformation (AWS)


In this article, I will show how you can set up a protected API endpoint using AWS Lamda, API Gateway, and automate the deployment of the stack using AWS CloudFormation.

If you don’t have prior knowledge of the resources/components mentioned above, read the high-level description below.

AWS Lambda
==========

AWS Lambda lets you run code without provisioning or managing servers. With Lambda, you can run code for virtually any type of application or backend service — all with zero administration. Read [more](https://aws.amazon.com/lambda/)

Amazon API Gateway
==================

Amazon API Gateway is a fully managed service that makes it easy for developers to create, publish, maintain, monitor, and secure APIs at any scale. APIs act as the “front door” for applications to access data, business logic, or functionality from your backend services. Read more

AWS CloudFormation
==================

AWS CloudFormation provides a common language for you to model and provision AWS and third-party application resources in your cloud environment. AWS CloudFormation allows you to use programming languages or a simple text file to model and provision, in an automated and secure manner, all the resources needed for your applications across all regions and accounts. Read [more](https://aws.amazon.com/cloudformation/)

In the example, I have a sample inline API service deployed as Lambda and then expose the endpoint from that via the API gateway. For protecting the APIs, I will be using a managed API Key provided by AWS. There are many ways to protect an API endpoint depending on different use cases; I am not going to cover those in this article.

CloudFormation Template
=======================

Below is the template file, we will be using.

Let’s take a look at the important definitions.

Lambda function
---------------

I am using an inline lambda function based on NodeJS. The function responds with \`Hello World!\` for the API invocation.

API Key
-------

As mentioned before, I am using an API Key to protect the API. Below code, block define an API key, Usage Plan, and Usage Plan Key. You can see that I am adding `ApiKeyRequired: true` as part of the `AWS::ApiGateway::Method`

API Gateway
-----------

Defining the Gateway includes many resources, including API stage and Deployment.

In addition to this, you can also see a couple of IAM groups (`AWS::IAM::Role`) are defined on the template. One group is for the Gateway to access the Lambda function and another one for Cloud Watch log creation.

I also defined a few parameters; these are optional.

Create the Stack
================

Now that you understand the resource definitions on the template file, we can proceed to create the stack. I am going to create the stack using \`aws cli\`. Refer [this](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html) to setup your \`aws cli\`. You can also create the stack via [AWS Console](https://console.aws.amazon.com/cloudformation/home?region=us-east-1)

#### Validate the template file

`aws cloudformation validate-template — template-body file://stack.yaml`

####  Deploy the stack

```
aws cloudformation deploy — stack-name lambda-api — template-file stack.yaml — capabilities CAPABILITY\_IAM
```

Once the stack is created, you will below in your console

```
Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack — lambda-api
```

####  List the APIs

`aws apigateway get-rest-apis`

Get the `API Id` from above and construct your api url using the format `https://{restapi_id}.execute-api.{region}.amazonaws.com/{stage_name}/`

If you are unsure about the region, you can run `aws configure get region`

####  Get your API Key

Execute below command and copy `value`

`aws apigateway get-api-keys — include-value`

####  Invoke the API

```
curl -X GET \\
https:/{restapi\_id}.execute-api.{region}.amazonaws.com/v1/hello \\
 -H ‘x-api-key: {your api key}’
```

For successful requests, you can see a response like below.

`Hello World!`

Conclusion
==========

You now have a protected API with a Lambda integration! As a next step, you can replace the inline Lambda function with a real-world Lambda code. You can upload your Lambda functions to the Amazon S3 bucket and use that in your template. Check the S3 example [here](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-function.html) . Good luck!
