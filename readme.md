# Introduction

- Describe main ideas
- Describe user flow
- Ask clarifying questions
- Read vs. Write oriented

Main idea: Contractors can upload files for their projects. Subcontractors can download these files. Data is collected on how people use this file service. How would you design this system?

## Clarifying questions

- What is a "project" exactly? Can you specify start and end dates for bidding? Can you specify start and stop dates for that project?

- “Equip GCs with the means to SEND files to their subs.” The image in the briefing PDF shows a contractor is able to have folders of their own. How is access granted to subs and who can actually see the project? Or do all subs have access to all projects? Does the GC literally _send_ files over to the subs?

- There are [primary success metrics](success-metrics) at the bottom. Is it my goal to describe how I would track these pieces of data or should I just base my solution around these metrics?

## Assumptions

- GCs have fast enough networks speeds to upload files at a reasonable rate. Subs have fast enough networks to download files at a reasonable rate.

- Security isn't an issue and has already been configured through session and CSRF authentication tokens as well as DB, model, and frontend-level constraints. We can discuss possible implementations of security measures at the end if needed.

- Contractors and subcontractors aren't located too far away from any data centers in place. We can discuss solutions to this if this is an important consideration to make.

## Basic Setup

My setup has a client layer that will allow the user to interact with the rest of my application. I will have a server layer that handles routing of requests and backend logic. I will also have a persistence layer that takes care of any data I want to save, namely files and information about my models (contractors, projects, bids, files, etc.). We can assume this is already set up so that our file architecture can be integrated into it. To get into a bit more detail about each layer:

### Client:

Self-explanatory.

### Server:

**Routes**:

**_ASSUMPTION_**: There is already some type of authentication in place

- `GET /:project_id` - get project
- `GET /:project_id/:folder_id` - get particular folder
- `POST /:project_id/:folder_id` - create folder
- `DELETE /:project_id/:folder_id` - delete folder
- `POST /:project_id/:folder_id/upload` - upload file(s) (file in params)
- `GET /:project_id/:folder_id/download` - download file(s) (file_names in params)
- `DELETE /:project_id/:folder_id/delete_files` - delete file(s) (file_names in params)

**How do I handle all these requests coming in? We're talking about terabytes of file transfers and tens of thousands of construction companies interacting with our app in real time.**

Scaling!!! Smaller startups would normally use vertical scaling to get an initial proof of concept done, but you guys are already a Series B company so we'll be looking into scaling horizontally, i.e. more servers!

- Good way to do this: Amazon EC2. You guys are still growing fast, so you'll need something that can scale with your website without you needing to configure everything.

**Okay, now I have more servers... but how do I make sure they're all synchronized? How do I ensure all those servers are actually receiving requests? What happens if some server crashes?**

Use a metadata manager / load balancer. Some possibilities:

- Elastic Load Balancer. Easy to configure with other AWS services

Two ways to keep track of servers:

- Zookeeper. Really good at synchronizing many servers. Can also use range-based data allocation to prevent data collision while still synchronizing between thousands of servers
- Round-robin load balancing (MAJOR CON: will ping servers that are already overloaded with requests. This is especially important to avoid because file uploads could take a while, and the server will have to be active as a file is getting uploaded, which we will discuss in a bit)

### Database:

Two kinds of data: files and metadata about files/users/etc.
Blob storage (AWS S3) + DB (MongoDB, DynamoDB, etc.)

- AWS S3 storage for individual files:

  - Unlimited file storage and incredible scalability
  - Easily configurable with other AWS services

- DB for other data (about GCs, subs, projects, bids, etc.). We can choose a NoSQL vs. SQL database. NoSQL is more scalable:

- Sacrifices [ACID-compliancy](acid) for greater freedom
- No joins / transactions, lower latency and faster write speed
- Easier to configure. SQL databases can scale, but require experts
- Easier for sharding and horizontal scaling

# Deliverable 1: File upload

1.) Get file from computer and put it into the server

First way
2.) Create a read stream
3.) Pipe the file into the bucket

Second way
2.) Convert the file to base64
3.) Store in S3

What comes back as payload from S3 upload: folder_name, file_name, file_type, file_size, last_updated, etc.

**Problem**: when a user makes requests it has to upload the file to S3, return info about the uploaded object, then persist that information to the database under that project. This potentially slows things down because every upload request requires multiple transactions.

Possible solution: store it all in one DB

- Store the bytes of a file directly in a column
- Add whatever metadata you want to the table / collection

**How do I ensure that the GC is the only one with write access to a project?**

`project.creatorId === currentUser.id`

**How do we make sure there are no collisions in filenames?**

- Because they are persisted to different folders, ensure you're uploading them into the right folder, so if two projects have a file or folder with the same name it's ok!

Three possibilities for file names in the same folder:

- Persist as Date.now + file_name (e.g. '19208310floor_plans.png') to ensure different file names
- Use versioning with ETags (same key, multiple versions of a file under that key, organized by most recently updated)
- Allow the the file to be overwritten for space purposes

**What if I accidentally overwrite a file, or want to update files while having access to previous versions?**

- Same as above: S3 offers versioning, which allows you to keep multiple variants of an object in the same bucket. You can easily recover from unintended user actions and application failures this way.

**What if my file is too large?**

- S3 has 5GB limit for PUT, 5TB limit for file
- Use multi-part uploading to split upload into smaller byte chunks
- Each mini-upload returns an ETag that can be used to determine the order of byte chunks

**What if I want multi-file upload?**

- Just send multiple files over and have the transaction iterate over each. Two options for persisting:
  1.) Wait until all files have uploaded successfully. Roll back any uploads if one fails. (Promise.all)
  2.) (More reasonable): Update information in DB as each resolves. Ones that fail need to be individually re-uploaded

# Deliverable 2: File download

Two possibilities:

1.) Proxy download from S3 service

- Get file from S3 and bring it to server
- Create a read stream
- Pipe the response stream into a file
- Download that file

OR

2.) Direct download from S3

- Get the download url for the file from your DB
- Redirect to the download URL

**What if I want multi/all-file download?**

Folders and files are all considered objects in S3 (even though it might not seem that way to us!). There are libraries out there that allow you to created a zipped file containing a directory. You make a `GET` request to obtain all of the files (or the whole project) from S3, iterate over those objects, and append them to a zipped file. Then just download that file.

# Data model

Contractor: name, etc.: projects: [1, 2, 3]

Subcontractor: name, etc., projects: [1, 2, 3]

Project: creator_id, name, files: [1, 2, 3]

File Object: file path ('blah/blah/blah.png'), file_type, last_updated

**_Assumption_**: This system assumes that each project will look the same to contractors and subcontractors. However, we might want projects to look different, for example we might want some metadata about projects:

Contractors:

Subcontractors:

- Which files have they downloaded?
- Which files have they downloaded that are the most recent version of the file?
- How many times have they downloaded files?

# Additional problems

**Microservice vs. Monolith**

Pros of Microservices:

- Easier to scale, test, debug, etc. individual pieces
- Less training needed for a developer to understand that piec of the service
- Bugs don't bring down the whole system

Pros of Monolith:

- Service is simple to think of as a whole
- Faster (all logic is in one place, network calls are needed less)

**How do I improve read times on my site?**

Caching. Using a cache like Redis or Memcache can vastly speed up read times by eliminating the need to query the entire database.
We could cache projects that

**How can I speed up downloads?**

Using ETags for file requests improves caching speed. The server can just check whether the version of a file is the same as a previously downloaded version (rather than all of its contents) and access a cached version of the file if it's been downloaded before.

**So I just keep storing files forever? Do I ever want to migrate content off AWS or delete it?**

- If you want to save space from the start on S3, you can upload smaller files there and larger files to a data warehouse (harder to access)
- If the project is expired or bidding is finished, you can migrate all the project files to a data warehouse. That way you are saving space on S3 for what's most important.

**How do I backup my data?**

Backup data to a data warehouse every hour. Pipe data from S3 into a data warehouse (e.g. Alooma). Restore data from the warehouse if there are problems with S3.

**What about security and authentication?**

We can assume that auth is already embedded in the app. Each request will be signed with CSRF.

**What if my users are all far away from each other?**

We need multiple data centers to distribute the load geographically. This will improve read time at the expense of storing more data in more places. This is best managed through a CDN or Edge interface (AWS Cloudfront or AWS Lambda@Edge)

**What if I want to track whether a file has been updated? (UI question)**

- Attach ETag onto response header (identifier for a specific version of a resource)
  - Also allows caches to be more efficient as mentioned before!
- Append some metadata onto subcontractor’s instance of a project
- Use a notification server that connects a project with all of the subcontractors that have interacted with that project

# Extra notes

<a name="success-metrics"></a>

### Primary Success Metrics:

General Contractors:

- how many of their projects have files uploaded to them
- how many files are uploaded per project
- file types uploaded

Subcontractors:

- when files are downloaded
- when files are downloaded in bulk vs. one at a time
- how long downloads take

\*_Maybe MapReduce to organize this data? Or just do it dynamically since we know what we're looking for\*_

<a name="acid"></a>

### ACID:

- Atomicity: all of transaction saved or none of it
- Consistency: constraints will not be violated
- Isolation: transactions occur as if in serial order
- Durability: data will not be lost
