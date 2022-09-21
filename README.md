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