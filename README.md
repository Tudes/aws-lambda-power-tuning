# AWS Lambda Power Tuning

[![Build Status](https://travis-ci.com/alexcasalboni/aws-lambda-power-tuning.svg?branch=master)](https://travis-ci.org/alexcasalboni/aws-lambda-power-tuning)
[![Coverage Status](https://coveralls.io/repos/github/alexcasalboni/aws-lambda-power-tuning/badge.svg)](https://coveralls.io/github/alexcasalboni/aws-lambda-power-tuning)
[![GitHub license](https://img.shields.io/github/license/alexcasalboni/aws-lambda-power-tuning.svg)](https://github.com/alexcasalboni/aws-lambda-power-tuning/blob/master/LICENSE)
[![Maintenance](https://img.shields.io/badge/Maintained%3F-yes-green.svg)](https://GitHub.com/alexcasalboni/aws-lambda-power-tuning/graphs/commit-activity)
[![GitHub issues](https://img.shields.io/github/issues/alexcasalboni/aws-lambda-power-tuning.svg)](https://github.com/alexcasalboni/aws-lambda-power-tuning/issues)
[![Open Source Love svg2](https://badges.frapsoft.com/os/v2/open-source.svg?v=103)](https://github.com/ellerbrock/open-source-badges/)
[![GitHub stars](https://img.shields.io/github/stars/alexcasalboni/aws-lambda-power-tuning.svg)](https://github.com/alexcasalboni/aws-lambda-power-tuning/stargazers)

AWS Lambda Power Tuning is an AWS Step Functions state machine that helps you optimize your Lambda functions in a data-driven way.

The state machine is designed to be **quick** and **language agnostic**. You can provide **any Lambda function as input** and the state machine will **run it with multiple power configurations, analyze execution logs and suggest you the best configuration to minimize cost**.

The input function will be executed in your AWS account (i.e. real HTTP calls, SDK calls, cold starts, etc.). The state machine also supports cross-region access and you can enable parallel execution to generate results in just a few seconds.

![state-machine](state-machine-screenshot.png?raw=true)


## How to deploy the state machine (AWS Serverless Application Repository)

You can find this app in the [Serverless Application Repository](https://serverlessrepo.aws.amazon.com/applications/arn:aws:serverlessrepo:us-east-1:451282441545:applications~aws-lambda-power-tuning) and deploy it with just a few clicks.

## How to deploy the state machine (AWS SAM)

In case you want to deploy it "manually", you can run `deploy.sh`.

The script uses AWS SAM CLI to create a new CloudFormation stack in your account.

First, install AWS SAM and configure your AWS credentials:


```
$ pip install aws-sam-cli
$ aws configure
```

Now, you can clone this repository as follows:

```
$ git clone https://github.com/alexcasalboni/aws-lambda-power-tuning.git
```

Configure your bucket name and stack name in the deployment script: 


```
# config
BUCKET_NAME=your-sam-templates-bucket
STACK_NAME=lambda-power-tuning
PowerValues='128,512,1024,1536,3008'
```

You can finally run it:

```
$ bash deploy.sh
```


## How to execute the state machine (programmatically)

You can simply run the `execute.sh` script. It will start a state machine execution, wait for the execution to complete, and then show the execution result in case of success.

## How to execute the state machine (manually)

Once the state machine and all the Lambda functions have been deployed, you can execute the state machine and provide an input object.

You will find the new state machine in the [Step Functions Console](https://console.aws.amazon.com/states/) or in your app's `Resources` section.

The state machine name will depend on the stack name (default: `aws-lambda-power-tuning`). Find it and click "**Start execution**". 

Here you can provide the execution input and an execution id (see section below for the full documentation):

```
{
    "lambdaARN": "your-lambda-function-arn",
    "num": 10
}
```

As soon as you click "**Start Execution**" again, you'll be able to visualize the execution.

Once the execution has completed, you will find the execution results in the "**Output**" tab of the "**Execution Details**" section. The output will contain the optimal power configuration and its corresponding average cost per execution.


## State Machine Input

The AWS Step Functions state machine accepts the following parameters:

* **lambdaARN** (required, string): unique identifier of the Lambda function you want to optimize
* **num** (required, integer): the # of invocations for each power configuration (minimum 5, recommended: between 10 and 100)
* **payload** (string or object): the static payload that will be used for every invocation
* **parallelInvocation** (false by default): if true, all the invocations will be executed in parallel (note: depending on the value of `num`, you may experience throttling when setting `parallelInvocation` to true)


## State Machine Output

The AWS Step Functions state machine will return the following outputs:

* **power**: the optimal power configuration
* **cost**: the corresponding average cost (per invocation)
* **duration**: the corresponding average duration (per invocation)


## State Machine Internals

The AWS Step Functions state machine is composed of four Lambda functions:

* **initializer**: create N versions and aliases corresponding to the power values provided as input (e.g. 128MB, 256MB, etc.)
* **executor**: execute the given Lambda function `num` times, extract execution time from logs, and compute average cost per invocation
* **cleaner**: delete all the previously generated aliases and versions
* **finalizer**: compute the optimal power value (current logic: lowest average cost per invocation)

Initializer, cleaner and finalizer are executed only once, while the executor is used by N parallel branches of the state machine (one for each configured power value). By default, the executor will execute the given Lambda function `num` consecutive times, but you can enable parallel invocation by setting `parallelInvocation` to `true`. Please note that the total invocation time should stay below 300 seconds (5 min), which means that the average duration of your functions should stay below 3 seconds with `num=100`, 30 seconds with `num=10`, and so on.

## Note about the previous version of this project

This project used to require a generation step to dynamically create the required steps based on which memory/power configurations you wanted to test.

The new version of this project doesn't require any generation step, but you may want to fine-tune the state machine in the `template.yml` file to add additional configurations.

Please note that you can specify a subset of configuration values in the `PowerValues` CloudFormation parameter. Only those configurations will be tested. Even though you will still see them in the stete machine chart, the `executor` steps corresponding to non-tested configurations will not run the input function (they'll be skipped).


## CHANGELOG (SAR versioning)

* *1.2.0*: updated IAM permissions (least privilege for actions)
* *1.1.1*: updated docs
* *1.1.0*: cross-region invocation support
* *1.0.1*: new README for SAR
* *1.0.0*: AWS SAM refactor (published on SAR)
* *0.0.1*: previous project (serverless framework)

## Contributing
Contributors and PRs are always welcome!

### Tests and coverage

Install dev dependencies with `npm install --dev`. Then run tests with `npm test`, or coverage with `npm run coverage`.
