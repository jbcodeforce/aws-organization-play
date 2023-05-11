# AWS Organization playgound

This repository is a sandbox to do some AWS Organization play and security control. 

This is related to the IAM tutorial about delegating access across AWS accounts using IAM roles. 

## Requirements

* Create Organization with [AWS organizations CLI](https://docs.aws.amazon.com/cli/latest/reference/organizations/index.html#cli-aws-organizations) from my main account. An AWS Organization is a collection of AWS accounts that you can manage centrally. Use all features.

    ```sh
    aws organizations create-organization
    # See existing org
    aws organizations describe-organization
    ```

* Build following structure of AWS accounts within the main org:

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


    The IAM policy can be:

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

    Create the policy

    ```sh
    # Be logged on specific account. 
    aws iam create-policy --policy-name s3-list --policy-document file://iam-res/s3-list.json
    ```

    Create an IAM Role (account type) in the new accounts that allows access to the resources that the group of users needs to access.  The trusted account is the `main account` of the organization.
    

    We can apply to other resources by adding content to the policy. But we can reach the policy size limit.

* Set up a trust relationship between the new account and the AWS account where the group of users are located. This allows the users to assume the IAM Role in the new account.
* Create a Cross-Account IAM Role in the AWS account where the group of users are located that allows them to assume the IAM Role in the new account.
* Grant access to the Cross-Account IAM Role to the IAM Group that the group of users belong to.

## Resources

* [IAM CLI for user and groups](https://docs.aws.amazon.com/cli/latest/userguide/cli-services-iam-new-user-group.html)
* [Organization CLI](https://docs.aws.amazon.com/cli/latest/reference/organizations/index.html#cli-aws-organizations)
* [IAM tutorial: Delegate access across AWS accounts using IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html)