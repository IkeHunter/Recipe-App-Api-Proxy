# DevOps Course Notes

## SSH Authentication
SSH keys are used to create secure connections between local machines and remote servers. Private keys are stored on the computer and are never shared, public keys are shared with the external servers. The public key matches with the private key to establish the connection.

### Create SSH Key Pair
In terminal:
```sh
ssh-keygen -t rsa -b 4096 -C "[name of key]"
```
`-t`: type of rsa  
`-b`: size of 4096 bytes  
`-C`: "comment", or name of key (ex: isaac@mbp) 

This creates a new ssh key in directory `~/.ssh`. Retrieve contents of public key with:
```sh
cat ~/.ssh/[file].pub
```

## AWS Terms
* **IAM**: sub user of account, should only contain permissions necessary to user's role.
* **Group**: Group of users, has default permissions
* **AWS-Vault**: local software provided by aws to securely authenticate and connect to your account. It should connect via IAM user. Keeps a connection open for a certain duration. Credentials are stored via Mac's keychain.
* **ARN**: long string representing a key, used to authenticate different services between each other.
* **Budget**: setting in aws that allows user to define different spending thresholds. Threshold will not stop spending, but will notify user when reached.
* **ECR**: service in aws to store docker images

## AWS Account Setup
It is important to always use an IAM user when managing settings, and to always have MFA setup on both root and IAM accounts.
### Setup Users / Groups
1. Go to Billing, scroll down and activate IAM access to Billing Info.
2. To create policy to enable MFA by default: 
    - Go to IAM > Policies > Create Policy
    - Insert following JSON:
    ``` json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AllowViewAccountInfo",
                "Effect": "Allow",
                "Action": [
                    "iam:GetAccountPasswordPolicy",
                    "iam:GetAccountSummary",
                    "iam:ListVirtualMFADevices",
                    "iam:ListUsers"
                ],
                "Resource": "*"
            },
            {
                "Sid": "AllowManageOwnPasswords",
                "Effect": "Allow",
                "Action": [
                    "iam:ChangePassword",
                    "iam:GetUser"
                ],
                "Resource": "arn:aws:iam::*:user/${aws:username}"
            },
            {
                "Sid": "AllowManageOwnAccessKeys",
                "Effect": "Allow",
                "Action": [
                    "iam:CreateAccessKey",
                    "iam:DeleteAccessKey",
                    "iam:ListAccessKeys",
                    "iam:UpdateAccessKey"
                ],
                "Resource": "arn:aws:iam::*:user/${aws:username}"
            },
            {
                "Sid": "AllowManageOwnSigningCertificates",
                "Effect": "Allow",
                "Action": [
                    "iam:DeleteSigningCertificate",
                    "iam:ListSigningCertificates",
                    "iam:UpdateSigningCertificate",
                    "iam:UploadSigningCertificate"
                ],
                "Resource": "arn:aws:iam::*:user/${aws:username}"
            },
            {
                "Sid": "AllowManageOwnSSHPublicKeys",
                "Effect": "Allow",
                "Action": [
                    "iam:DeleteSSHPublicKey",
                    "iam:GetSSHPublicKey",
                    "iam:ListSSHPublicKeys",
                    "iam:UpdateSSHPublicKey",
                    "iam:UploadSSHPublicKey"
                ],
                "Resource": "arn:aws:iam::*:user/${aws:username}"
            },
            {
                "Sid": "AllowManageOwnGitCredentials",
                "Effect": "Allow",
                "Action": [
                    "iam:CreateServiceSpecificCredential",
                    "iam:DeleteServiceSpecificCredential",
                    "iam:ListServiceSpecificCredentials",
                    "iam:ResetServiceSpecificCredential",
                    "iam:UpdateServiceSpecificCredential"
                ],
                "Resource": "arn:aws:iam::*:user/${aws:username}"
            },
            {
                "Sid": "AllowManageOwnVirtualMFADevice",
                "Effect": "Allow",
                "Action": [
                    "iam:CreateVirtualMFADevice",
                    "iam:DeleteVirtualMFADevice"
                ],
                "Resource": "arn:aws:iam::*:mfa/${aws:username}"
            },
            {
                "Sid": "AllowManageOwnUserMFA",
                "Effect": "Allow",
                "Action": [
                    "iam:DeactivateMFADevice",
                    "iam:EnableMFADevice",
                    "iam:ListMFADevices",
                    "iam:ResyncMFADevice"
                ],
                "Resource": "arn:aws:iam::*:user/${aws:username}"
            },
            {
                "Sid": "DenyAllExceptListedIfNoMFA",
                "Effect": "Deny",
                "NotAction": [
                    "iam:CreateVirtualMFADevice",
                    "iam:EnableMFADevice",
                    "iam:GetUser",
                    "iam:ListMFADevices",
                    "iam:ListVirtualMFADevices",
                    "iam:ResyncMFADevice",
                    "sts:GetSessionToken",
                    "iam:ListUsers"
                ],
                "Resource": "*",
                "Condition": {
                    "BoolIfExists": {
                        "aws:MultiFactorAuthPresent": "false"
                    }
                }
            }
        ]
    }

    ```
    - Give it name, description, create.
3. Create group by going to Groups > Create Group.
    - For admin user, select `AdministratorAccess` policy and newly created MFA policy.
4. To create IAM user, go to Users > Add User. Add name, select console access. Add to Group.
5. Save account ID. Log in as new IAM user (or whoever new account is for), and enter account id, username, password. Set up MFA. Log out, then log in again for full permissions.

### Set up AWS-Vault
1. In IAM account, go to IAM > Users > Security Credentials, create new Access Key. (Leave success dialog/window open to copy private key)
2. With aws-vault installed (using brew), enter following command
```sh
aws-vault add [username of IAM user]
```
Enter Access key and private key when prompted. This will create a new vault on mac, enter new password for that as well (if prompted).
3. Set up MFA for console by editing the config file. Access file by following command:
```sh
vim ~/.aws/config
```
Then add the following to file:
```
region=us-east-1
mfa_serial=[ARN for MFA device]
```
Find the ARN for the device in IAM > Users > Security Credentials, make sure to select the correct key - it should be in a section labeled Assigned MFA Device (or something similar). 
4. Enable secure session with the following command:
```sh
aws-vault exec [username] --duration=12h
```
Duration can be configured for any amount of hours up to 12.

### Create AWS Budget (optional)
Creating an budget will tell AWS to notify the user via email when a budget threshold is reached. It is possible to go past this threshold.

1. Go to Billing > Budgets, create custom cost budget. 
2. Name it, set it to monthly, make it fixed, and set the budget amount.
3. Create alerts to enable users when account has used up to certain amount of budget.


## Setup NGINX Proxy
Proxy servers exist to help django serve static files. It acts as a web server to make it more effecient to serve these files.

### Steps:
1. create new GitLab project
2. disable public pipelines, add protected branch `*-release` to ensure all branches flagged as `-release` are only able to be created/modified by certain users (maintainers). Add `*-release` to list of protected tags.
3. clone project via ssh to local. Passphrase is needed if it's set.
4. create new ECR repo, this is where the dynamic containers will be created. Enable `scan on push`.
5. Create a custom aws policy with the json below:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:*"
            ],
            "Resource": "arn:aws:ecr:us-east-1:*:repository/recipe-app-api-proxy"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken"
            ],
            "Resource": "*"
        }
    ]
}
```
6. create a new IAM user with the newly created policy, this will be the "user" that is used for dynamic ECR tasks. Create access key.
7. create the following protected/masked variables in gitlab:
    * AWS_ACCESS_KEY
    * AWS_SECRET_ACCESS_KEY
    * ECR_REPO (uri of the repo)
8. create new branch called `feature/nginx-proxy` in local project
9. create `default.conf.tpl` file, define proxy server config by setting the port, and location directories
10. create (copy) the uwsgi_params file, include it in base location of proxy server config. Additionally set the `client_max_body_size` variable on the base location to define max file upload size (10M)
11. create `entrypoint.sh` command file to pass proxy config file to remote, turn off docker's *daemon* mode - forcing it (and nginx) to run in the foreground
12. create base image with `Dockerfile` file. Build image.
13. understand CI/CD pipeline with GitLab
    * explains branch flow: https://docs.gitlab.com/ee/topics/gitlab_flow.html
    * predefined variables: https://docs.gitlab.com/ee/ci/variables/predefined_variables.html
    * `.gitlab-ci.yml` explainer: https://docs.gitlab.com/ee/ci/yaml/#rules
14. create `.gitlab-ci.yml` file to define services, stages, and jobs. specifically the Dev and Push stages
15. define the jobs for the Build stage, including scripts and artifacts (saved images)
16. define jobs for Push Dev (Push) stage, including scripts and rules
17. define jobs for Push Release (Push) stage, including scripts and rules
18. push to origin by setting upstream. Ensure this CI/CD job passes.
19. create new merge request. Merge to main. Ensure this CI/CD job passes. This should have created an ECR image (named dev).
20. create new minor version branch with release flag (ex: 1.0-release)
21. create new patch version branch with release flag (ex: 1.0.0-release). Ensure this CI/CD job passes. This should have created a new ECR image with version name and "latest" (ex: 1.0.0, latest)




