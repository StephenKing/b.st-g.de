---
layout: post
title:  "Tagging AWS Resources in Step Functions using TagSpecifications"
date:   2024-12-27 15:00:00 +0000
tags: [aws, step-functions]
description: How the AWS API confused me a lot... 

comments: true
share: true

---


Some types of AWS resources require the use of `TagSpecifications` in the AWS API and CLI. While I was attempting to tag newly created resources through SDK calls in Step Functions, I had a hard time figuring out, how to overcome the following error:

> Cannot deserialize value of type `java.util.ArrayList<software.amazon.awssdk.services.ec2.model.TagSpecification$BuilderImpl>` from Object value (token `JsonToken.START_OBJECT`) /States/CreateTransitGateway/Arguments

This parameter for the AWS CLI seems to have confused [already others](https://github.com/aws/aws-cli/issues/8211). Due to the lack of publicly documented examples of using `TagSpecifications` in AWS Step Functions state machines, this is a minimal working example for creating Transit Gateway:

```
{
  "QueryLanguage": "JSONata",
  "Comment": "A description of my state machine",
  "StartAt": "CreateTransitGateway",
  "States": {
    "CreateTransitGateway": {
      "Type": "Task",
      "Resource": "arn:aws:states:::aws-sdk:ec2:createTransitGateway",
      "Arguments": {
        "TagSpecifications": [
          {
            "ResourceType": "transit-gateway",
            "Tags": [
              {
                "Key": "Name",
                "Value": "Step Functions rock!"
              }
            ]
          }
        ]
      },
      "End": true
    }
  }
}
```

In contrast to the AWS CLI, where we can supply the `--tag-specifications` argument multiple times, we have to supply a list of `TagSpecifications` to the SDK call.