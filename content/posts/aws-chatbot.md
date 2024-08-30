---
title: "Aws Chatbot - Part I - Integrating with MS Teams"
date: 2024-01-29T21:27:59+05:30
tags:
    - aws
---

 AWS Chatbot is a powerful integration tool that connects with Microsoft Teams, Slack, and Amazon Chime, bringing AWS notifications and commands right into your chat channels.

 This guide will walk you through the simple yet impactful process of setting up AWS Chatbot with Microsoft Teams. Whether you're a developer, a project manager, or just an AWS enthusiast, this step-by-step tutorial is designed to help you leverage AWS Chatbot to its full potential.

This service is relatively new so there is not quite enough documentation on how to set it up. So I decided to write this blog to help you set up AWS Chatbot with Teams / Slack and Chime in Subsequent parts. This part will be about setting up **AWS Chatbot with MS Teams**.

> Note : The source code for this blog is available in [Github]().

## Prerequisites


- AWS Account with required access
- Slack / Teams / Chime Account
- AWS CLI installed and configured


## Setting up AWS Chatbot with MS Teams

### Create a Teams Channel
- Create a new channel in Teams and add the AWS Chatbot app to the channel.
- Once the app is added, We will have to get the teams channel link. To get the link, click on the three dots next to the channel name and select **Get link to channel**. Copy the link and save it for later use.

![Get link to channel](/get_link_to_channel.png)

### Creating AWS Chatbot MS Teams configuration

- Go to AWS Chatbot service in AWS console and click on **Create configuration**. Select **Microsoft Teams channel** and click on **Configure**.

![Select AWS Chatbot](/select_aws_chatbot.png)

- Here give the URL that we copied from the teams channel and click on **Next**.

![Add Teams Channel URL](/add_teams_channel_url.png)

- Now the team configuration is done. We will have to create a channel configuration. In the chatbot configuration screen click on the **Configure New Channel** button.

![Configure New Channel](/configure_new_channel.png)

- Next we will have to specify the channel name and channel URL for the channel that we created. We will also specify the IAM role that will be used by the chatbot. Here we have 2 types of roles `channel level roles` and `user level roles`.
    - Channel level roles will be applied to all users in a channel
    - In User level roles the users will have to choose an IAM role to perform the action.
    - We will be going with channel level roles.

- We can set guardrails to restrict permissions for the chatbot. This adds an an extra layer of deny permissions to the chatbot. We will be skipping this step for now.

![Add Chatbot Config](/add_chatbot_config.png)

- We will also have to create an SNS topic that will be used by the chatbot to send notifications. This step is optional though and can be skipped if we are not planning to run chatbot to send notifications. But we will have this enabled because we are going to explore that usecase too.

![Add SNS Topic](/add_sns_topic.png)

- Once all the details are added we can review and create the channel configuration.


## Using AWS Chatbot to send notifications and run commands

### Sending Notifications

To send notifications we will have to send a message in the following format to the SNS topic that we created.

```json

{
  "version": "1.0",
  "source": "custom",
  "content": {
      "title": "Title of the message",
      "description": "Description of the message"
  },
  "metadata": {
      "additionalContext": {
          "key": "value" // This can be used as a variable that can be used to trigger a command
      }
  }
}

```

Typically this can be done in a lambda function that is triggered by an event. For example, we can trigger a lambda function when a new EC2 instance is created and send a notification to the SNS topic. In the variable section we can send the instance id of the newly created instance. This can be used to run commands on the instance via AWS CLI. We will see how to do that in the next section.

We will be creating a lambda function that will send a notification to the SNS topic that we created. Here is the lambda function that we are going to use.

```python
import json
import boto3

def lambda_handler(event, context):
    print(event)
    instance_id = event['detail']['instance-id']
    print(instance_id)
    sns = boto3.client('sns')
    response = sns.publish(
        TopicArn='<topic-arn>',
        Message=json.dumps({
            "version": "1.0",
            "source": "custom",
            "content": {
                "title": f"Instance Created - {instance_id}",
                "description": f"An instance has been created in the account. Here is the full event - {event}"
            },
            "metadata": {
                "additionalContext": {
                    "instanceID": instance_id
                }
            }
        })
    )
    print(response)
    return {
        'statusCode': 200,
        'body': json.dumps('Message Published to SNS')
    }
```

Here is a event filter that can be used to trigger the lambda function.

```json
{
  "source": [
    "aws.ec2"
  ],
  "detail-type": [
    "EC2 Instance State-change Notification"
  ],
  "detail": {
    "state": [
      "running"
    ]
  }
}
```

Once the lambda function is created we can test it by creating a new EC2 instance. Once the instance is created we will get a notification in the teams channel.

### Running Commands

To run cli commands on AWS chatbot you will have to send a message to the channel wih the prefix as `@aws`. Here is the format for the commands.

```bash
@aws service command --options
```

For example, if you want to get the list of instances in your account you can send the following message to the channel.

```bash
@aws ec2 describe-instances
```

However there are some limitations to this. There are a few non supported actions that cant be performed via chatbot. To find more info on this and how the commands can be used, check out the [official documentation](https://docs.aws.amazon.com/chatbot/latest/adminguide/chatbot-cli-commands.html).


### Custom actions

In the custom action we are going to call a lambda that will get the tags of the instance and adds that to a DynamoDB resource tracking table. Here is the lambda function that we are going to use.

```python
import json
import boto3

def lambda_handler(event, context):
    print(event)
    instance_id = event['instanceID']
    print(instance_id)
    ec2 = boto3.client('ec2')
    response = ec2.describe_tags(
        Filters=[
            {
                'Name': 'resource-id',
                'Values': [
                    instance_id,
                ]
            },
        ]
    )
    print(response)
    tags = response['Tags']
    print(tags)
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('resource_tracking')
    table.put_item(
        Item={
            'resource_id': instance_id,
            'tags': tags
        }
    )
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }
```

Lets send a test message from the lambda to create a custom action now. Once the message is recieved in the channel. Click on the **Create custom action** button.

![Create Custom Action](/create_custom_action.png)

In the next screen we will have to specify a `Name` for the action, `Display text` on the button and the type of action. It can be a `Lambda action` or a `CLI action`. We will be going with the `Lambda action` for now.

![Add Custom Action](/add_custom_action.png)

In the next steps we will also have to specify the AWS account id and the region where the lambda function is present. We will also have to specify the lambda function name and the input that we are going to send to the lambda function. In this case we are going to send the instance id as the input. This will be available as a parameter in the post that we sent.

![Add Custom Action](/add_custom_action_2.png)

Once the custom action is created we will get a confirmation message in the channel.

Lets create an instance and test now. Once the instance is created we will get a notification in the channel. We will also get the custom action button.
On clicking the button we will get a prompt to run the lambda. If we proceed with the run, the lambda will be triggered and the tags will be added to the DynamoDB table. Also the lambda output will be posted in the channel as a reply in the thread.


## Cloudformation Support

AWS Chatbot can be configured using cloudformation. Here is a sample cloudformation template that can be used to create a chatbot configuration is given in the following github repo.


[Cloudformation - AWS Chatbot integration with MS Teams]()

**Note:** Before creating the channel configuration and other resources, we will have to create the chatbot team configuration. This has to be done initially via the console. And is not yet supported via cloudformation. More details on the templates can be found in the repo.

## References

- [AWS Chatbot](https://docs.aws.amazon.com/chatbot/latest/adminguide/what-is.html)

- [AWS Chatbot - MS Teams](https://docs.aws.amazon.com/chatbot/latest/adminguide/notifications-ms-teams.html)

- [AWS Chatbot - Cloudformation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/AWS_Chatbot.html)

---

Tadaaa 🥳.

We have a chatbot integrated with MS Teams. This can be used to automate a lot of tasks like sending out alerts on monitoring events, cost management / budget alerts etc.

You're now ready with the knowledge to integrate AWS Chatbot with Microsoft Teams! Experiment with its features and see how it transforms your workflow. If you have questions or need a hand, don't hesitate to connect with me on LinkedIn. Found this guide useful? Share it with your network and help others discover the benefits of AWS Chatbot. **Happy automating!** 🚀
