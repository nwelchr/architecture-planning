# Introduction

## Clarifying questions

- What is a project? Can you specify start and end dates for bidding? Can you specify start and stop dates for that project's ?

- “Equip GCs with the means to SEND files to their subs.” The image shows a contractor is able to have folders of their own. How is access granted to subs and who can actually see the project?

- Where are contractors and subcontractors located? Is it just in the SF area or is it worldwide?

- There are [success-metrics](primary success metrics) at the bottom. Is it my goal to describe how I would implement these or should I just base my solution around the fact that these metrics are the most important?

## Assumptions

-

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
  - 5GB limit for PUT, 5TB limit for file (if you use multi-part uploading)
  - Easily configurable with other AWS services

So for this DB, we can choose a NoSQL vs. SQL database. NoSQL is more scalable.

- Sacrifices [acid](ACID-compliancy) for greater freedom
- No joins / transactions, lower latency and faster write sped
- Easier to configure. SQL databases can scale, but require experts
- Sharding and horizontal scaling

============================================================

# Deliverable 1: File upload

**Problem**: when a user makes requests, it has to visit both the S3 bucket and the DB in some sort of order. The problem is, blob storage generally gives you metadata that you'd want access to without having to query that blob storage (file names/paths, last updated,etc.). Some information would probably be reduplicated. E.g.:

What comes automatically with S3 upload: folder_name, file_name,

2.) Possible solution: store it all in one DB

- Store the bites of a dile directly in a column / record on a table / collection
- Add whatever metadata you want to the table / collection

============================================================

# Deliverable 2: File download

============================================================

- Additional problems:

**How do I improve read times on my site?**
Caching. Using a cache like Redis or Memcache can vastly speed up read times by eliminating the need to query the entire database.
We could cache projects that

**How can I speed up downloads?**

**So I just keep storing files forever? Do I ever want to migrate content off AWS or delete it?**

**How do I backup my data?**
Backup data to a data warehouse every hour

**What about security and authentication?**
Each request will be signed with CSRF

**What if my users are all far away from each other?**
We need multiple data centers to distribute the load geographically. This will improve read time at the expense of storing more data in more places. This is best managed through a CDN or Edge interface (AWS Cloudfront or AWS Lambda@Edge)

============================================================

# Extra notes

<a name="success-metrics"></a>

### Primary Success Metrics:

<a name="acid"></a>

### ACID:

- Atomicity: all of transaction saved or none of it
- Consistency: constraints will not be violated
- Isolation: transactions occur as if in serial order
- Durability: data will not be lost
