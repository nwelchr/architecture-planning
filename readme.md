# Introduction

**Describe basics of the project**

## Clarifying questions

- What is a project? Can you specify start and end dates for bidding? Can you specify start and stop dates for that project?

- “Equip GCs with the means to SEND files to their subs.” The image shows a contractor is able to have folders of their own. How is access granted to subs and who can actually see the project? Does the GC literally _send_ files over to the subs?

- There are [success-metrics](primary success metrics) at the bottom. Is it my goal to describe how I would track these pieces of data or should I just base my solution around these metrics?

## Assumptions

- Security isn't an issue and has already been configured through session and CSRF authentication tokens as well as DB, model, and frontend-level constraints. We can discuss possible implementations of security measures at the end if needed.

- Contractors and subcontractors aren't located too far away from any data centers in place. We can discuss solutions to this if this is an important consideration to make.

## Basic Setup

### Client

### Server

**Routes**:

**_ASSUMPTION_**: There is already some type of authentication in place

**CLARIFYING QUESTION**: Do I need to be able to delete files?

- `GET /:project_id` - get project
- `GET /:project_id/:folder_id` - get particular folder
- `POST /:project_id/:folder_id` - create folder
- `DELETE /:project_id/:folder_id` - delete folder
- `POST /:project_id/:folder_id/upload` - upload file(s)
- `GET /:project_id/:folder_id/download` - download file(s)
- `DELETE /:project_id/:folder_id/:file_id` - delete file(s)

### Database

The main issue is we have uploaded files and other data (about GCs, subs, projects, bids, etc.).

**Two possibilities**:
1.) Blob storage (AWS S3) + DB (MongoDB, DynamoDB, etc.)

- AWS S3 storage for individual files:
  - Unlimited file storage and incredible scalability
  - Easily configurable with other AWS services

So for this DB, we can choose a NoSQL vs. SQL database. NoSQL is more scalable.

- Sacrifices [acid](ACID-compliancy) for greater freedom
- No joins / transactions, lower latency and faster write sped
- Easier to configure. SQL databases can scale, but require experts
- Sharding and horizontal scaling

# Deliverable 1: File upload

**Problem**: when a user makes requests it has to upload the file to S3, return info about the uploaded object, then persist that information to the database under that project. This potentially slows things down because every upload request requires multiple transactions.

What comes back as payload from S3 upload: folder_name, file_name, file_type, file_size, last_updated, etc.

Possible solution: store it all in one DB

- Store the bites of a dile directly in a column / record on a table / collection
- Add whatever metadata you want to the table / collection

**What if my file is too large?**

- S3 has 5GB limit for PUT, 5TB limit for file
- Use multi-part uploading to split upload into smaller byte chunks
- Each mini-upload returns an ETag that can be used to determine the order of byte chunks

**What if I want multi-file upload?**

- Just send multiple files over and have the transaction iterate over each. Two options for persisting:
  1.) Wait until all files have uploaded successfully. Roll back any uploads if one fails. (Promise.all)
  2.) (More reasonable): Update information in DB as each resolves. Ones that fail ned to be individually re-uploaded

**What if I accidentally overwrite a file?**

- S3 offers versioning, which allows you to keep multiple variants of an object in the same bucket. You can easily recover from unintended user actions and application failures this way.

# Deliverable 2: File download

- Additional problems:

**How do I improve read times on my site?**
Caching. Using a cache like Redis or Memcache can vastly speed up read times by eliminating the need to query the entire database.
We could cache projects that

**How can I speed up downloads?**
Using ETags for file requests improves caching speed. The server can just check whether the version of a file is the same as a previously downloaded version (rather than all of its contents) and access a cached version of the file if it's been downloaded before.

**So I just keep storing files forever? Do I ever want to migrate content off AWS or delete it?**

**How do I backup my data?**
Backup data to a data warehouse every hour. Pipe data from S3 into a data warehouse (e.g. Alooma). Restore data from the warehouse if there are problems with S3.

**What about security and authentication?**
We can assume that auth is already embedded in the app. Each request will be signed with CSRF.

**What if my users are all far away from each other?**
We need multiple data centers to distribute the load geographically. This will improve read time at the expense of storing more data in more places. This is best managed through a CDN or Edge interface (AWS Cloudfront or AWS Lambda@Edge)

# Extra notes

<a name="success-metrics"></a>

### Primary Success Metrics:

<a name="acid"></a>

### ACID:

- Atomicity: all of transaction saved or none of it
- Consistency: constraints will not be violated
- Isolation: transactions occur as if in serial order
- Durability: data will not be lost
