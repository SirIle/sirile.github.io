---
layout: post
title:  "Building multiarch Boto3 Lambda layer with AWS CodeBuild"
---

{::options parse_block_html="true" /}
<div class="toc">
Contents

* toc
{:toc}
</div>

I've been experimenting with [Amazon Bedrock](https://aws.amazon.com/bedrock/). As the technology is rapidly evolving,
the APIs change often. Calling the APIs requires a recent version of the Boto3 library. The version included in the Python 3.12 runtime
offered by the Lambda doesn't update that often, so sometimes trying to call the APIs using the included Boto3 leads into errors.

Enabling the Lambda to use the newest version of the library can be done in multiple ways. Having the library as a Lambda layer
means that the Lambda code doesn't need to include the quite large library and the built-in Lambda editor in the console
can be used to modify the code and the updates are quick.

This article demonstrates a way to use [AWS CodeBuild](https://aws.amazon.com/codebuild/) to automate the creation of the Lambda layer.
The created layer is suitable for both x86_64 and arm64 architectures.

## Prerequisites

AWS account is required.

## Creating the CodeBuild Build project

The CodeBuild project is created and configured so that it creates the zip file for the Lambda layer and then
publishes the layer.

### Create the project

Navigate to CodeBuild and press "Create project" button. Enter a name for the project. Give a name for the service role used to build so
it is easier to identify later. Also the description can be given. For the build commands select the "Switch to editor" and replace the code
with the following:

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.12
  build:
    commands:
      - python3.12 -m venv venv
      - source venv/bin/activate
      - mkdir python
      - cd python
      - pip install boto3 -t .
      - rm -rf *.dist-info
      - export BOTO3_VERSION=$(python -c "import boto3; print(boto3.__version__)")
      - cd ..
      - zip -qr layer.zip python
      - echo "Boto3 version $BOTO3_VERSION zipped to the file layer.zip, publishing as Lambda layer.."
      - export BOTO3_LAYER_NAME=boto3-$(echo $BOTO3_VERSION | sed 's/\./_/g')
      - aws lambda publish-layer-version --layer-name $BOTO3_LAYER_NAME
        --description "Lambda layer for Boto3 version $BOTO3_VERSION"
        --zip-file fileb://layer.zip
        --compatible-runtimes python3.12
        --compatible-architectures x86_64 arm64
      - echo "Lambda layer $BOTO3_LAYER_NAME published"
```

- Lines 9 and 10 create and activate a virtualenv so that the packages aren't installed as root user
- Lines 11 to 14 create the directory structure, install the Boto3 library and remove unnecessary files
- Line 15 extracts the version number from the installed library
- Lines 16 and 17 zip the library so that it can be published as a Lambda layer
- Line 19 modifies the version string so that it contains underscores instead of commas as they're prohibited in a Lambda layer name
- Lines 20 to 24 publish the Lambda layer using AWS cli

<img src="/images/Boto3-create-CodeBuild-project.png" alt="Create the CodeBuild project" style="box-shadow: 3px 3px 3px gray;">

Press "Create build project".

### Modify the IAM role

Press the service role link which navigates to the IAM role. Press "Add permissions->Create inline policy". Replace the JSON with

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Action": "lambda:PublishLayerVersion",
			"Resource": "arn:aws:lambda:*:<account_id>:layer:*"
		}
	]
}
```

Replace the placeholder "\<accountid\>" with the account id that is used. Press "next" and give a name for the policy, for example "upload-lambda-layers".
Press "Create policy".

### Run the build

The build can be started with "Start build"-button. The progress can be followed in the log and it should finish successfully in about a minute.

## Testing with Lambda

To test the created layer, a simple Lambda that displays the version of the Boto3 library is created. A test is run with the default
Lambda runtime which displays the Boto3 version. Then the newly created layer is added to the Lambda. The test is run again
and the displayed Boto3 version should be different.

### Create the Lambda through the Console

Navigate to the Lambda->Functions in the AWS Console. Press "Create function" and enter the basic information.

<img src="/images/Boto3-create-function.png" alt="Create the function" style="box-shadow: 3px 3px 3px gray;">

Replace the code with

```python
import json
import boto3

def lambda_handler(event, context):
    print(boto3.__version__)
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }
```

### Run the test

Press Deploy and then Test with the default values. The output should be:

<img src="/images/Boto3-test-1.png" alt="Result of the run without Lambda layer" style="box-shadow: 3px 3px 3px gray;">

For this run the version of the Boto3 library was 1.34.42.

### Add the Lambda layer for Boto3

Scroll to the bottom of the Lambda definition and press "Add a layer.." under Layers,
choose "Custom layers" and the layer that was created by the CodeBuild.

<img src="/images/Boto3-add-layer.png" alt="[Add the Lambda layer" style="box-shadow: 3px 3px 3px gray;">

### Run the test again

After adding the layer the Lambda is automatically deployed. Run the test again and output should be:

<img src="/images/Boto3-test-2.png" alt="Result of the run with the Lambda layer" style="box-shadow: 3px 3px 3px gray;">

The Boto3 library version was now 1.34.151 which is significantly newer than the default version included in the Lambda runtime.
