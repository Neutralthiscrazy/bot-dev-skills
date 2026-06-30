---
name: aws-lambda-discord-friends
description: >-
  Companion digest for AWS Lambda as bot host. Event handling,
  Lambda-Web adapter for fastify, secrets via SSM / Secrets
  Manager, IAM tight scoper per bot, and bundle size for
  discord.js.
version: 0.1.0
user-invocable: false
author: neutralthiscrazy
license: MIT
---

# AWS Lambda as bot host

## When to load

- AWS-native deployment (existing AWS infra)
- Need longer per-request timeout (Lambda allows 15 min)
- Python bot (Lambda is Python-friendly)

If you don't already use AWS, Vercel or CF Workers are simpler.

## Handler template (Python)

```python
# bot.py
import os
import json
import boto3
from botocore.exceptions import ClientError

SECRET_NAME = os.environ.get("SECRET_NAME", "telegram/bot-token")

# Cached client
_secrets = boto3.client("secretsmanager")

def get_token():
    r = _secrets.get_secret_value(SecretId=SECRET_NAME)
    return r["SecretString"]

# Lazy import after token grab to keep cold-start fast
_bot = None

def handler(event, context):
    global _bot
    if _bot is None:
        from aiogram import Bot, Dispatcher
        _bot = Bot(token=get_token())
    update = json.loads(event["body"])
    # Process synchronously - quick reply (Lambda webhook expects fast ack)
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"ok": True}),
    }
```

## SAM / CDK / Terraform

Without infra-as-code, API Gateway HTTP API + Lambda + Secrets Manager
is a manual 5-step console click. SAM template below:

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  BotFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./bot/
      Handler: bot.handler
      Runtime: python3.13
      MemorySize: 256
      Timeout: 30
      Environment:
        Variables:
          SECRET_NAME: telegram/bot-token
      Policies:
        - AWSSecretsManagerGetSecretValuePolicy:
            SecretArnLists: ["arn:aws:secretsmanager:*:*:secret:telegram/*"]
      Events:
        Webhook:
          Type: HttpApi
          Properties:
            Path: /webhook
            Method: post
```

SAM deploys with `sam deploy --guided`.

## Secrets

SSM Parameter Store (free, simple):

```python
import boto3
ssm = boto3.client("ssm")
token = ssm.get_parameter(Name="/my-bot/token", WithDecryption=True)["Parameter"]["Value"]
```

Secrets Manager (paid, rotation support) is overkill for most.

## Common pitfalls

- Cold start: ~200ms-2s for Node, longer for Python. Long package.
- API Gateway returns before Lambda finishes unless configured
  differently. Use async invocation for long handlers.
- IAM scope wide-open by default — restrict to `secretsmanager:
  GetSecretValue` for just `arn:aws:secretsmanager:*:*:secret:telegram/*`.
- VPC-bound Lambda has even longer cold starts. Don't put Lambda
  in VPC unless you need to reach private resources.

## Sources

- AWS Lambda — https://docs.aws.amazon.com/lambda/
- Lambda + Python — https://docs.aws.amazon.com/lambda/latest/dg/lambda-python.html
- Secrets Manager — https://docs.aws.amazon.com/secretsmanager/
- API Gateway HTTP API — https://docs.aws.amazon.com/apigatewayv2/
- SAM — https://docs.aws.amazon.com/serverless-application-model/
