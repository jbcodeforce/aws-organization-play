# AWS Organization playgound

This repository is a sandbox to do some AWS Organization play and security control. 

This is related to the IAM tutorial about delegating access across AWS accounts using IAM roles. 

## Requirements

* Create Organization with [AWS organizations CLI](https://docs.aws.amazon.com/cli/latest/reference/organizations/index.html#cli-aws-organizations) from my main account. An AWS Organization is a collection of AWS accounts that you can manage centrally. Use all organization features.

    ```sh
    aws organizations create-organization
    # See existing org
    aws organizations describe-organization
    ```

* Build following AWS accounts structure within the main org:

    ![](./docs/diagrams/orgs-sandbox.drawio.png)

    ```sh
    aws organizations create-organization-unit --parent-id o-cyxzj... --name Class-A
    aws organizations create-organization-unit --parent-id o-cyxzj... --name Class-B
    ```

    

* Create the new AWS accounts in the organization we want to grant access to the group of users.

    ```sh
    aws organizations create-account --email userid+1@amazon.com --account-name cust-A1
    aws organizations create-account --email userid+2@amazon.com --account-name cust-A2
    aws organizations create-account --email userid+3@amazon.com --account-name cust-B1
    aws organizations list-accounts 
    # move each accounts under the good OU with command like 
    aws organizations describe-organizational-unit --organizational-unit-id ou-kxk2-9j73quif
    aws organizations move-account --account-id <value> --source-parent-id <value> --destination-parent-id <value>
    ```

    We can imagine to have thousand of accounts.

    AWS automatically creates a IAM role in the accounts that grants administrator permissions to IAM users in the management account who can assume the role.

    ![](./docs/org-acct-access.png)

    We may remove this role or modify the policy to authorize only admin users from the management account.

* Be able to limit access to different accounts for group of user as in the following diagram:

    ![](./docs/diagrams/group-access.drawio.png)

* Create two groups of users in main account: `devops, developers`.

    ```sh
    aws iam create-group --group-name devops
    aws iam create-group --group-name developers
    aws iam list-groups
    ```

    Output:

    ```json
    {
    "Groups": [
        {
            "Path": "/",
            "GroupName": "developers",
            "GroupId": "AGPAV4D67BJ6CIQQ2HE3M",
            "Arn": "arn:aws:iam::4....6:group/developers",
            "CreateDate": "2022-10-13T14:21:46Z"
        },
        {
            "Path": "/",
            "GroupName": "devops",
            "GroupId": "AGPAV4D67BJ6DHBSM4TBI",
            "Arn": "arn:aws:iam::4...6:group/devops",
            "CreateDate": "2023-05-11T19:09:08Z"
        }
    ]
    }
    ```


* Add the users to the IAM Groups in the main AWS account

    ```sh
    aws iam create-user --user-name Ray
    aws iam create-user --user-name Bill
    aws iam add-user-to-group --user-name Ray --group-name developers
    aws iam add-user-to-group --user-name Bill --group-name devops
    # Verify 
    aws iam list-groups-for-user  --user-name Bill
    ```

* To be able to authorize access to resources within the different account we need to add role and policies. For example we have S3 bucket in cust-a account like:

    ![](./docs/res-cust-a.png)


    We need to define a set of security controls that can be summarized in the figure below:

    ![](./docs/diagrams/security-control.drawio.png)


    The IAM policy to control access to s3 bucket may look to at least the following declarations:

    ```json
    {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement1",
            "Effect": "Allow",
            "Action": "s3:ListAllMyBuckets",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetBucketLocation"
            ],
            "Resource": "*"
        }
        ]
    }
    ```

    Create this policy under the `cust-*` account:

    ```sh
    # Be logged on specific account. 
    aws iam create-policy --policy-name s3-list --policy-document file://iam-res/s3-list.json
    ```

* Set up a trust relationship between the new account and the AWS account where the group of users are located. This allows the users to assume the IAM Role in the new account. For that modify the trusted entity policy (file `iam-res/trusted-entity.json`) with the management account ID:

    ```json
    {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::mgt_account_id:group/devops"
            },
            "Action": "sts:AssumeRole",
            "Condition": {}
        }
    ]
    }
    ```

    Create an IAM Role (account type) in the new accounts that allows access to the resources that the group of users needs to access.  The trusted account is the `main account` of the organization.
    
    ```sh
    # be sure to be logged in target account
    aws iam create-role --role-name UpdateApp --assume-role-policy-document file://iam-res/trusted-entity.json
    ```

    We can apply to other resources by adding content to the policy. But we can reach the policy size limit.

* Grant access to the Cross-Account IAM Role to the IAM Group that the group of users belong to.

    * Create an inline policy to modify the permissin of the group:

    ```json
    {
    "Version": "2012-10-17",
    "Statement": {
        "Effect": "Allow",
        "Action": "sts:AssumeRole",
        "Resource": "arn:aws:iam::<custom-account-id>:role/UpdateApp"
        }
    }
    ```

    * Attach this inline policy to the needed group:

    ```sh
    aws iam put-group-policy --group-name devops --policy-document file://iam-res/permission.json --policy-name RemoteAccountAccess
    ```

### Testing the IAM solution


Need to test the user in devops group can list S3 bucket on the target accounts for example. The test steps look like:

* Configure user

    ```sh
    # with the user key in the .aws/config
    aws configure --profile Bill
    aws sts assume-role --role-arn "arn:aws:iam::813165465392:role/UpdateApp" --role-session-name "Bill-Update"
    ```

    Output looks like:

    ```
    {
    "Credentials": {
        "AccessKeyId": "....",
        "SecretAccessKey": "...",
        "SessionToken": "FwoGZXIvYXdzEKn//////////wEaDG9TAu/hxTq1MoAY9SKvATuzvLLaN4OVW7knkPrW3m8TmtPvQAVZ8v3osjj3PZg1Tvn9oXrkZo1gv+7tNG+daaxgVwcEcKK1SQ+m3y0ZxIDHN7jhe5ggF6/trAEb+c6VWeAq2+zjtJyQv8cGcvwkzg02KBW1DGbZJSo296u/6NNDp/4q6TCaOulYMk1AKsuo1u0oa1EAkB9yzvmT5W/hUEx8cZeg9P1hzlSkA2+aSH+XEiFCVweq61mERYI9CekoyPf1ogYyLXALUQGf0iFXn4Dyb5H/Sqs0eCBYewwAR809ER3btXoZ9V3AAFlZ5c5I+HCGng==",
        "Expiration": "2023-05-12T00:35:36Z"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROA32VDQJ4YH62LBP4KS:Bill-Update",
        "Arn": "arn:aws:sts::813165465392:assumed-role/UpdateApp/Bill-Update"
    }
    }
    ```

* The output include access key, temporary token... To swap role we need to export those value as environment variable

    ```sh
    export AWS_ACCESS_KEY_ID=
    ```

    ```sh
    export AWS_SECRET_ACCESS_KEY=
    ```

    ```sh
    export AWS_SESSION_TOKEN=
    ```

* Execute the command on s3

    ```sh
    aws s3 ls
    > 2023-05-10 17:45:37 my-customer-a-bucket
    ```

Ray is not able to swith role to the cust-A1 ot cust-A2 account.

## Limitations of the IAM solution

As we can see in previous section, for each group of user in the main account, we need a role with trusted entity principal = to the new group.

## Using Service Control Policy in Organizations

SCPs may be attached to the root, Organization Unit or AWS accounts. 

* Be sure to have enabled SCP in the management account.

## Resources

* [IAM CLI for user and groups](https://docs.aws.amazon.com/cli/latest/userguide/cli-services-iam-new-user-group.html)
* [AWS Organizations CLI](https://docs.aws.amazon.com/cli/latest/reference/organizations/index.html#cli-aws-organizations).
* [IAM tutorial: Delegate access across AWS accounts using IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html).
* [Managing multi-account AWS env with Organization - re:Inforce 2019](https://www.youtube.com/watch?v=fxo67UeeN1A)
* [My summary of AWS security-Organizations.](https://jbcodeforce.github.io/aws-studies/infra/security/#aws-organizations)
* [SCP examples](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_examples_general.html)