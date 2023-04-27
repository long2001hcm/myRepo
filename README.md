1.  [Back-end Development Handbook](index.html)
2.  [Back-end Development
    Handbook](Back-end-Development-Handbook_262349.html)
3.  [HANDBOOK](HANDBOOK_25100313.html)
4.  [04. MANUAL](04.-MANUAL_25100304.html)

# Back-end Development Handbook : M10-1.AWS-Datapack-Management

Created by Huu Phan Trong, last modified by Thao Tran on Jan 18, 2023

# Concept

![](attachments/55541772/55541785.jpg)

# S3 Storage Config

-   Create new bucket

![](attachments/55541772/55672859.png)

-   Set bucket-name, AWS region,....

    ![](attachments/55541772/55246894.png)

-   Block all public access

![](attachments/55541772/74907695.png)

-   Create bucket

    ![](attachments/55541772/55738379.png)

-   Edit Cross-origin resource sharing

![](attachments/55541772/55246902.png?width=680)

**Note**: Change `AllowedOrigins`

``` {.syntaxhighlighter-pre syntaxhighlighter-params="brush: java; gutter: false; theme: Confluence" theme="Confluence"}
[
    {
        "AllowedHeaders": [
            "*"
        ],
        "AllowedMethods": [
            "PUT",
            "POST",
            "DELETE"
        ],
        "AllowedOrigins": [
            "https://app.dn01-tecq.gotecq.net",
        ],
        "ExposeHeaders": []
    }
]
```

-   Update config in `[fii-datapack]`

``` {.syntaxhighlighter-pre syntaxhighlighter-params="brush: java; gutter: false; theme: Confluence" theme="Confluence"}
[fii_datapack]
DB_DSN = <db_dsn>
FILESYSTEM_S3_BUCKET = <your_bucket_name>
FILESYSTEM_S3_ACCESS_KEY = <your_access_key>
FILESYSTEM_S3_SECRET_KEY = <your_secret_key>
FILESYSTEM_S3_REGION = <s3_region>
FILESYSTEM_BACKEND = S3
```

*Get your access key and secret key:*
[https://docs.aws.amazon.com/powershell/latest/userguide/pstools-appendix-sign-up.html](https://docs.aws.amazon.com/powershell/latest/userguide/pstools-appendix-sign-up.html)

# Lambda Config

-   Setup

-   Create lamdata function

![](https://drive.google.com/uc?id=15lS8vvZh7cXzQcCqsRBoRGb0Qe1wqnDq)![](attachments/55541772/55476280.png?width=680)

-   Set Function name

![](attachments/55541772/55574576.png?width=680)

-   Paste code below or code in `src/lambda/share_file_request.py`

![](attachments/55541772/55541802.png?width=680)

``` {.syntaxhighlighter-pre syntaxhighlighter-params="brush: java; gutter: false; theme: Confluence" theme="Confluence"}
import hashlib
import json
import os
import urllib.parse

import boto3
import requests

print("Loading function")

s3 = boto3.client("s3")

# Config
webhook_url = os.environ.get("WEBHOOK_URL")
secret_token = os.environ.get("WEBHOOK_SECRET_TOKEN")


def analysis_key(key: str) -> str:
    arr = key.split("/")
    datapack_identifier = arr[0]
    display = arr[-1]
    path = "/" + "/".join(arr[1:-1])
    return (datapack_identifier, display, path)


def hash(streambody):
    sha256_hash = hashlib.sha256()
    for chunk in streambody.iter_chunks():
        sha256_hash.update(chunk)
    return sha256_hash.hexdigest()


def lambda_handler(event, context):
    bucket = event["Records"][0]["s3"]["bucket"]["name"]
    key = urllib.parse.unquote_plus(
        event["Records"][0]["s3"]["object"]["key"], encoding="utf-8"
    )
    eTag = event["Records"][0]["s3"]["object"]["eTag"]
    try:
        response = s3.get_object(Bucket=bucket, Key=key)

        headers = response["ResponseMetadata"]["HTTPHeaders"]
        if ("x-amz-meta-uploaded-by" in headers) and (
            headers["x-amz-meta-uploaded-by"] == "presigned-url"
        ):
            content_type = response["ContentType"]
            datapack_identifier, display, path = analysis_key(key)
            length = response["ContentLength"]

            requests.post(
                webhook_url,
                data=json.dumps(
                    {
                        "datapack_identifier": datapack_identifier,
                        "display": display,
                        "path": path,
                        "content_type": content_type,
                        "length": length,
                        "etag": eTag,
                        "secret_token": secret_token,
                        "sha256": hash(response["Body"]),
                        "submitter_name": headers["x-amz-meta-uploader-name"],
                        "submitter_email": headers["x-amz-meta-uploader-email"],
                        "_updated": str(response["LastModified"]),
                    }
                ),
            )
        return response["ContentType"]
    except Exception as e:
        print(
            "Error getting object {} from bucket {}. Make sure they exist and your bucket is in the same region as this function.".format(
                key, bucket
            )
        )
        raise e
```

-   Edit Environment variables in Configuration tab

    -   `WEBHOOK_SECRET_TOKEN`: secret token is used to authen webhook
        in datapack service

        -   In env config:

        -   ``` {.syntaxhighlighter-pre syntaxhighlighter-params="brush: java; gutter: false; theme: Confluence" theme="Confluence"}
            [data_foundation]
            WEBHOOK_SECRET_TOKEN = AUiXLlHRVihlG1rSAqNaRorLi6DkgHTf
            ```

        -   Note: `WEBHOOK_SECRET_TOKEN` can be anything (random string,
            UUID), but `WEBHOOK_SECRET_TOKEN` in app env config and
            lambda configuration must be the same.

    -   `WEBHOOK_URL:` your API domain

        -   Example
            `https://admin.gotecq.com/api/gotecq.data-foundation/hook/add-datapack-resource`

![](attachments/55541772/55509031.png)

## APPENDIX: Ngrok for multiple services in TESTING ENVIRONMENT

-   Install ngrok

-   Connect your account

``` {.syntaxhighlighter-pre syntaxhighlighter-params="brush: java; gutter: false; theme: Confluence" theme="Confluence"}
ngrok config add-authtoken <your_authtoken>
```

Running this command will add your authtoken to the default `ngrok.yml`
configuration file. This will grant you access to more features and
longer session times. Running tunnels will be listed on the endpoints
page of the dashboard.

-   Edit `ngrok.yml` file (host04: `/home/gotecqvn/.ngrok2/ngrok.yml`)

Example: You want to use ngrok for data-foundation-api (port 8230) and
contact-center-api (port 8190)

``` {.syntaxhighlighter-pre syntaxhighlighter-params="brush: java; gutter: false; theme: Confluence" theme="Confluence"}
authtoken: <your_authtoken>
version: "2"
region: us
tunnels:
  dtp:
    proto: http
    addr: 8230
    bind_tls: false
  ctc:
    proto: http
    addr: 8190
    bind_tls: false
```

-   Run them with

``` {.syntaxhighlighter-pre syntaxhighlighter-params="brush: java; gutter: false; theme: Confluence" theme="Confluence"}
ngrok start --all
```

![](attachments/55541772/55738405.png)

## Attachments:

![](images/icons/bullet_blue.gif)
[auto_sync_datapack-webhook.jpg](attachments/55541772/55541785.jpg)
(image/jpeg)\
![](images/icons/bullet_blue.gif)
[image-20220930-040945.png](attachments/55541772/55672859.png)
(image/png)\
![](images/icons/bullet_blue.gif)
[image-20220930-041340.png](attachments/55541772/55246894.png)
(image/png)\
![](images/icons/bullet_blue.gif)
[image-20220930-041731.png](attachments/55541772/55509023.png)
(image/png)\
![](images/icons/bullet_blue.gif)
[image-20220930-041814.png](attachments/55541772/55738379.png)
(image/png)\
![](images/icons/bullet_blue.gif)
[image-20220930-042411.png](attachments/55541772/55738387.png)
(image/png)\
![](images/icons/bullet_blue.gif)
[image-20220930-042419.png](attachments/55541772/55771160.png)
(image/png)\
![](images/icons/bullet_blue.gif)
[image-20220930-043114.png](attachments/55541772/55246902.png)
(image/png)\
![](images/icons/bullet_blue.gif)
[function_lamda_1.png](attachments/55541772/55771170.png) (image/png)\
![](images/icons/bullet_blue.gif)
[function_lamda_2.png](attachments/55541772/55476280.png) (image/png)\
![](images/icons/bullet_blue.gif)
[function_lamda_3.png](attachments/55541772/55574576.png) (image/png)\
![](images/icons/bullet_blue.gif)
[function_lamda_4.png](attachments/55541772/55541802.png) (image/png)\
![](images/icons/bullet_blue.gif)
[function_lamda_5.png](attachments/55541772/55509031.png) (image/png)\
![](images/icons/bullet_blue.gif)
[image-20220930-045801.png](attachments/55541772/55738405.png)
(image/png)\
![](images/icons/bullet_blue.gif)
[image-20221110-094228.png](attachments/55541772/75628547.png)
(image/png)\
![](images/icons/bullet_blue.gif)
[image-20221110-095323.png](attachments/55541772/74907695.png)
(image/png)\

Document generated by Confluence on Apr 26, 2023 08:24

[Atlassian](http://www.atlassian.com/)
