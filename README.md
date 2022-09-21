# CrowdStrike Project

VirusTotal is a online service owned by Google that allows the analysis of files, URLs,IP addresses, domains or file hashes. It facilitates the rapid detection of viruses, worms, Trojan horses and all kinds of malicious software.

VirusTotal aggregates many antivirus products and online scan engines called Contributors.

## Technologies

The tools proposed for the architecture are mainly fully managed AWS Services.

- **REST APIs**: API Gateway with lambda functions and EKS with python or Java containers

- **Microservices**: EKS (Kubernetes)

- **Authentication**: AWS Cognito or Auth0 with Api Gateway Lambda Authorizer

- **File Storage**: AWS S3 S3 Intelligent-Tiering

- **Messaging Service**: AWS SNS

- **Event Streaming**: AWS Kinesis or Kafka

- **Data Storage**: MongoDB Atlas

- **Transaction Caching**: Redis

- **Monitoring/Metrics/Logs**: CloudWatch

## Workflow


1. The user upload a file using the REST API. 
2. The lambda function or Container handle the file uploaded and Start an analysis.
3. Send a message to the relevant topic to notify subscribers
4. Subscribers handle the message, notify a topic that an analysis is in progress and run the script against the file.
5. Subscribers send results of scripts to notify that the analysis is done.
6. The finalizing service handles message comming from workers and add metadatas to the file in Database

## 	How and where will we store the data?

We are storing the data in mongoDB using sharding with Hashed index on id to distributed more evenly.

Object stored : Files, Analysis, Users

We also use a redis instance to storage temporarly Transaction (for large file uploading) in the form of <transaction_id>:user_id : bool Enabled

##	What Statistics can we track on uploads, internal metrics for understanding system health

- Uptime of each service for each request/processing.
- Scripts runtime for file based on size
- errors and failures
- number of request per endpoints
- time for processing end to end based on size (E2E health status).

## How will we handle failure in the system

For a failure in the system we can have logs, metrics and alerts to keep a trace, be able to debug it and maybe notify the creator or the people responsible.

As most of the things are async we can replay failed events (maybe have a count on the replay). if synchronous we can retry with exponential back-offs to leave some time for the service to recover.

We might have timeouts on the scripts also to not run forever and track it.

We also might have a timeout on the general analysis of the file to not wait indefinitely.

##  Where’s a good place to store logs and metrics

We are using AWS CloudWatch

## What auth do we need? 

We are using Oauth2 with Auth0 or AWS Cognito

Every request is going to be handled by a Lambda authorizer which will verify the bearer token submitted. It can also check the authorization of the user for protected endpoints.

# REST API

Description of each endpoints

## File API

### Upload and analyse a file

```
POST /api/v1/files
Authorization: Bearer <token>
Request Body: File
```
```
Responses: HTTP 200 {analysisId : <id>}
		   HTTP 417 "File too large."
```
### Get an URL to upload a large file

```
GET /api/v1/upload_url
Authorization: Bearer <token>
```
```
Response: HTTP 200 {upload_url : <endpoint_of_upload_api>/{transactionid}
```


### Upload a large file

```
POST /_ah/upload/{transaction_id}
Authorization: Bearer <token>
content-type: multipart/form-data
Request Body: Multi-part upload body 
```
```
Response: HTTP 200 {analysis_id : <id>} (deactivate transaction)
 	      HTTP 417 File too large
 	      HTTP 403 Forbidden
 	      HTTP 400 Bad request
```

### Download a file (redirects you to download URL)

```
GET /api/v1/files/{SHA-256, SHA-1 or MD5 identifying the file}/download 
Authorization: Bearer <token>
```
```
Response: HTTP 200 {The file}
```

### Get a download url for a file

```
GET /api/v1/files/{SHA-256, SHA-1 or MD5 identifying the file}/download_url 
Authorization: Bearer <token>
```
```
Response: HTTP 200 {download_url: <url>}
```

### Get the report of a file

```
POST /api/v1/files/{SHA-256, SHA-1 or MD5 identifying the file}
Authorization: Bearer <token>
```
```
Response: HTTP 200 {File : <file>}
```

### Re-Analyse an existing file

```
POST /api/v1/files/{SHA-256, SHA-1 or MD5 identifying the file}/analyse
Authorization: Bearer <token>
```
```
Response: HTTP 200 {analysisId : <id>}
          HTTP 400 Bad request
```

### Get all analyses associated with a file

```
GET /api/v1/files/{SHA-256, SHA-1 or MD5 identifying the file}/analyses
Authorization: Bearer <token>
```
```
Response: HTTP 200 {analyses: [list of analyses associated with that file]}
          HTTP 400 Bad request
```

### Get all analyses associated with a file

```
GET /api/v1/files/{SHA-256, SHA-1 or MD5 identifying the file}/analyses/analysis_id
Authorization: Bearer <token>
```
```
Response: HTTP 200 {analysis: Analysis object}
          HTTP 400 Bad request
```

## Analyses API

### Get a file analysis

```
GET /api/v1/analyses/{id}
Authorization: Bearer <token>
```
```
Response: HTTP 200 {analysis: Analysis object}
          HTTP 400 Bad request
```


# Object models

We store most of our object using MongoDB

With mongo you can enforce a schema on your object. This is not required but it's a good practice !!

## Analysis Object

```
{       
    "id": <string>
    "date": <int:timestamp>,
    "results": {
        "<string>": {
            "category": "<string>",
            "engine_name": "<string>",
            "engine_version": "<string>",
            "engine_update": "<string>",
            "method": "<string>",
            "result": "<string>"
        },
        "stats": {
            "confirmed-timeout": <int>,
            "failure": <int>,
            "harmless": <int>,
            "malicious": <int>,
            "suspicious": <int>,
            "timeout": <int>,
            "type-unsupported": <int>,
            "undetected": <int>
        },
        "status": "<string>"      
}
```

## File Object

```
{
    "metadata": {Objects},
    "downloadable": <bool>,
    "first_submission_date": <int:timestamp>,
    "last_analysis_date": <int:timestamp>,
    "last_analysis_results": {
        "<string:engine_name>": {
            "category": "<string>",
            "engine_name": "<string>",
            "engine_update": "<string>",
            "engine_version": "<string>",
            "method": "<string>",
            "result": "<string>"
        }
    },
    "last_analysis_stats": {
        "confirmed-timeout": <int>,
        "failure": <int>,
        "harmless": <int>,
        "malicious": <int>,
        "suspicious": <int>,
        "timeout": <int>,
        "type-unsupported": <int>,
        "undetected": <int>
    },
    "last_modification_date": <int:timestamp>,
    "last_submission_date": <int:timestamp>,
    "md5": "<string>",
    "meaningful_name": "<string>",
    "names": [
        "<strings>",...
    ],
    "reputation": <int>,
    "sandbox_verdicts": {
        "<string:sandbox_name>": {
            "category": "<string>",
            "confidence": <int>,
            "malware_classification": [
                "<string>"
            ],
            "malware_names": [
                "<string>"
            ],
            "sandbox_name": "<string>"
        },
    },
    "sha1": "<string>",
    "sha256": "<string>",
    "sigma_analysis_results": [{
      "rule_title": "<string>",
      "rule_source": "<string>",
      "match_context": [{
        "values": {
          "<string>": "<string>"}}],
      "rule_level": "<string>",
      "rule_description": "<string>",
      "rule_author": "<string>",
      "rule_id": "<string>"
    }],
    "sigma_analysis_stats": {
        "critical": <int>,
        "high": <int>,
        "low": <int>,
        "medium": <int>
    },
    "sigma_analysis_summary": {
        "<string:ruleset_name>": {
            "critical": <int>,
            "high": <int>,
            "low": <int>,
            "medium": <int>
        }
    },
    "size": <int>,
    "tags": [
        "<strings>",...
    ],
    "times_submitted": <int>,
    "total_votes": {
        "harmless": <int>,
        "malicious": <int>
    },
    "type_description": "<string>",
    "type_extension": "<string>",
    "type_tag": "<string>",
    "unique_sources": <int>,
    "vhash": "<string>"
}

```

## User Object

```
{
    "user_id": "<string>"
    "username": "<string>"
    "email_address": "<string>",
    "creation_date": <int:timestamp>,
}
```

# Detailed workflow

## I. The user upload a file using the REST API

- Alternative 1: The user upload the file using the regular endpoint. It is handled by a lambda function. 

- Alternative 2: The user asks for a upload URL to be able to upload a large file.   
A lambda function handles the request. It generates an id (random GUID ?) and associates it to the user and store it in Redis with a TTL of 10min (arbitrary time). 
```
"transaction:{guid}:{user_id}" : {enabled: true} 
```
And returns an url made of the large file upload API and the generated id.  
The user uses the url to upload the file. It is handled by a container. 

## II. The lambda function or Container handles the file uploaded

It computes the MD5, SHA1, SHA256 and determines if it already exists.  
If the file doesn't already exist, we create a file entity in DB and put it in the S3 bucket. The file is named after it's md5 hash.  
We created an Analysis, and create a relationship between the file and the Analysis.
We push a message into the corresponding SNS topic.
```
Message format example: 

{file_id, file_url, analysis_id, user_id}
```

Then it returns the analysis_id.

## III. Subscribers process the incoming message

It can check if the file_id and file_url are right and correspond to the analysis identified by the id.

notify a topic that an analysis is in progress.
Run the script against the file.
Subscribers send results of scripts and notify that the analysis is done over SNS.


## IV. The finalizing Service handles the results (Analysis Service?)

The finalizing service handles message comming from workers and add metadatas to the files and analysis objects.