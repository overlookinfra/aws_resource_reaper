# AWS EC2 Reaper 

This AWS EC2 Reaper works using tags set on the instance itself. The Reaper is 
composed of two AWS Lambdas: the "Schema Enforcer" and the "Terminator". The 
Schema Enforcer ensures that all new instances have been correctly tagged, and
the Terminator runs periodically to terminate all running instances past their
termination date.

## Rules and Usage

**TL;DR** Tag an instance with a `lifetime` tag on creation.  A valid `lifetime` 
tag is a string of an integer value with a 1 letter unit of w(weeks), d(days), 
h(hours). For example, `1w` is 1 week, `2d` is 2 days, and `3h` is 3 hours. 

1. The Schema Enforcer ensures that a newly created EC2 instance has a valid 
future date set for termination. The Schema Enforcer looks for a `lifetime` tag
to determine that date. A valid `lifetime` tag is a string of an integer 
value with a 1 letter unit of w(weeks), d(days), h(hours). For example, `1w` is 
1 week, `2d` is 2 days, and `3h` is 3 hours. The Schema Enforcer will calculcate 
a future date based upon the `lifetime` tag and set a new `termination_date` tag 
on that instance.
    * Instead of setting the `lifetime` tag, you can set a `termination_date` 
    tag directly to specify the date the instance expires. A `termination_date` 
    must be a valid IS0 8601 value with a UTC offset defined.
2. If there is an error determining the future termination date, the instance is 
terminated. If 4 minutes elapse and no future termination date has been 
determined, the instance is terminated.
3. The Terminator runs periodically to ensure that all EC2 instances are
terminated if they are past their `termination_date`. If an instance needs its 
lifetime extended beyond its original future terminatation date, the 
`termination_date` tag should be updated directly.

### Example:

Let's say you're using Packer to build an AMI. If you don't tag it properly, the
Reaper will probably identify it as an untagged rogue instance before it's completed
and terminate it. To prevent that, make sure you set the `run_tags` key in your build
JSON file.

``` json
{
  "builders": [
    {
      "type": "amazon-ebs",
      "region": "us-west-2",
      "source_ami": "ami-c7d092f7",
      "instance_type": "m3.large",
      "ssh_username": "centos",
      "ami_name": "Example AMI",
      "ssh_pty": "true",
      "run_tags":  {
        "created_by": "<your email>",
        "department": "<your dept>",
        "project": "vm_publish",
        "lifetime": "1h"
      }
    }
  ],
  "..."
}
```
 
## Implementation and Details
The following sections are details meant for people implementing the AWS
EC2 Reaper.

### Installation

Installation of the reaper is accomplished by using cloudformation templates found
in the reaperfiles S3 folder. These templates are designed to be used with stacksets
to deploy the reaper across several accounts.

#### Prequisites

The directions from AWS [here](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-prereqs.html)
should be completed before use of the Cloudformation template. 

A few things to note:
1. The *administrative account* needs the administrative role as well as the
execution role. This ensures that deploying the `deploy_to_s3.yaml` can create the
necessary S3 buckets in each region in the administrative account for the Reaper 
Lambdas to read from.
2. Both roles should be created by deploying cloudformation stacks using the templates in the documentation linked above. You can download the templates by clicking the link in the documentation to the yaml files. You can follow the steps to create a new stack [here](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html)
3. Make sure to set a stack name that relates to what the stack does (like aws-cf-ar for the aws cloudformation administrator role stack), and use the administrative account ID for the AdministratorAccountID field. 
4. The Administrator role stack only needs to be deployed into the administrative account. The Execution role needs to be deployed into all accounts and the administrative account.

#### deploy_to_s3 Cloudformation template

This template places the lambda zip resources in S3 buckets in every region so that
the `deploy_reaper` template can read them for Reaper deployment. In order to use
this template, you must first manually create an S3 bucket that contains the
resources to copy across all regions. You will need to do this once per region;
S3 resources can be read between accounts but not between regions for AWS Lambda.
This only needs to be done one time for the administrative account. If new regions
are added you can just perform step 3 and list the new region.

1. Manually create an S3 bucket accessible from the administrative account. Zip up the
two python reaper files, `reaper.py` and `slack_notifier.py` and place them in the 
bucket, naming them `reaper.zip` and `slack_notifier.zip`. 

2. From the administrative account, create a new stack set and use the `deploy_to_s3`
template. An example CLI invocation would look like:

```
aws cloudformation create-stack-set --stack-set-name reaper-assets --template-body 
file://path/to/deploy_to_s3.yaml --capabilities CAPABILITY_IAM --parameters
ParameterKey=OriginalS3Bucket,ParameterValue=reaperfiles
```

3. Deploy stack-set-instances for this stack set, one per region in the administrative
account. Check the Amazon documentation for the most up-to-date region list.

```
aws cloudformation create-stack-instances --stack-set-name --accounts 123456789012
--regions "us-west-1" "us-west-2" "eu-west-1" ...
```

#### deploy_reaper Cloudformation template

After the resources for the reaper have been distributed, you can use the `deploy_reaper`
Cloudformation template to deploy the reaper into an account. That template can be found
in the folder lambdas/ec2/ in this repository. 

The template has five parameters it uses when creating a stack set. They are listed below
along with their default values (parameters are case sensitive):

1. SLACKWEBHOOK=none
3. TerminatorRate=rate(1 hour)
4. LIVEMODE=FALSE
5. S3BucketPrefix=ec2-reaper

In order to deploy the reaper you must supply the `SLACKWEBHOOK` parameter 
value for the `slack_notifier` Lambda to communicate to the Slack channel. 
Setting LIVEMODE to TRUE will enable the reaper to terminate instances immediately upon stack 
instance deployment.
The TerminatorRate and S3BucketPrefix values can be left as is, and shouldn't ever need set explicitly

You will need to follow the steps below for each account you are deploying the reaper into.

1. First, create a stack set representing the account you wish to run the reaper in.

```
aws cloudformation create-stack-set --stack-set-name reaper-aws-account --template-body
file://path/to/deploy_reaper.yaml --capabilities CAPABILITY_IAM --parameters
ParameterKey=SLACKWEBHOOK,ParameterValue=1234567 ...
```

2. Deploy the reaper into the account.

```
aws cloudformation create-stack-instances --stack-set-name reaper-aws-account --accounts
098765432109 --regions "us-west-1" "us-west-2" "eu-west-1" ...
```

### Turning the Reaper On

Don't fear the Reaper is turned on after the stack instances are created; they will not
reap anything unless the environment variable `LIVEMODE` is set to `TRUE`. It will
only report what it would have done to Slack. 

When the time comes to activate the Reaper, update the parameter value `LIVEMODE` to
"TRUE"(the regex is case-insensitive). 

```
aws cloudformation update-stack-set --stack-set-name reaper-aws-account
--use-previous-template --parameters ParameterKey=LIVEMODE,ParameterValue=TRUE --capabilities CAPABILITY_IAM
```

#### Testing

Run the reaper in no-op mode and ensure that it is behaving as expected; use the 
slack_notifier to get clean reporting, or look at the AWS logs directly. To run
the unit tests, install nose on your system and run `nose tests/lambdas/` from the
root of this repository.

### Components

#### Schema Enforcer
The Schema Enforcer is an AWS Lambda that is designed to be triggered when an EC2 
instance enters the 'pending' state. This AWS Lambda waits for an EC2 instance to 
have a valid `termination_date` tag associated with it. This AWS Lambda also 
listens for a `lifetime` tag; if found, it calculates a new future date and adds 
that date as the `termination_date` for the instance.

The Schema Enforcer terminates instances that do not have valid tags, or if the 
timeout period MINUTES_TO_WAIT has elapsed. Unhandled errors are raised, but the 
Schema Enforcer does not terminate the instance in these cases. The Schema 
Enforcer does not terminate instances after the schema has been enforced; the 
Terminator is responsible for that.

#### Terminator
The Terminator is a simple AWS Lambda that looks for a `termination_date` tag on
an instance and terminates it if it is past its `termination_date`. If the 
`termination_date` is missing or malformed, the script logs those instances in its
output. This AWS Lambda is designed to be run periodically; depending on 
your needs, every 15 minutes should be more than sufficient. The Terminator does
not ensure that EC2 instances have valid tags; the Schema Enforcer is responsible 
for that.

The `termination_date` must be in a IS0-8601 format with a UTC offset.

#### Slack Notifier
The Slack Notifier is a separate Lambda that can run and post data about 
terminated instances; it runs in its own Lambda, tied to the output of both the
Schema Enforcer and Terminator looking for a "REAPER TERMINATION" string match in
the output of the either Lambda. A Cloudwatch Log trigger with a filter pattern
or `REAPER TERMINATION` should be attached to this Lambda, and a Slack channel webhook
should be set as an environment variable. You will need to create a Slack workflow
in each channel that should receive notifications. The variables to use are as follows:

- *account:* The account's alias according to AWS.
- *message:* The REAPER TERMINATION string, i.e. the log entry
- *region:* The AWS region the reaper's running in.
