# [fit] Intro

^
- High-level overview of the project
  - About Go Free Range
  - GOV.UK assets, how they're created and served
  - Problems we were asked to solve and project goals
  - Our approach, what we've done, doing and what comes next
  - Finish with a couple of challenges and useful tips
- We're expecting to have a more detailed handover with Platform Support
- We can do another one of these in more more detail if that’d be useful/interesting

---

# [fit] Go Free Range

^
- I'm part of Go Free Range
- We are:

---

## [fit] James Mead

## [fit] @floehopper

---

## [fit] Chris Lowis

## [fit] @chrislo

---

## [fit] Chris Roos

## [fit] @chrisroos

---

## [fit] 2012

^
- We've worked with GDS since 2012
- We've worked on:

---

## [fit] Whitehall

---

## [fit] Smart Answers

---

## [fit] Manuals Publisher

---

## [fit] Asset Manager

---

# [fit] GOV.UK assets

^
- Overview of GOV.UK assets

---

## [fit] Static assets

^
- E.g. JS, CSS
- Served by individual apps
- See RFC 91 about some changes to how these are going to work

---

## [fit] Uploaded assets

^
- E.g. PDF, XLS
- See the assets page on the developer docs for more info

---

## [fit] 700GB

## [fit] 2,500,000

^
- Of Uploaded assets in Whitehall and Asset Manager

---

## [fit] /media

^
- Asset Manager assets

---

## [fit] /government/uploads

^
- Whitehall assets

---

# [fit] Creating assets

---

## [fit] Specialist Publisher

^
Via Asset Manager

---

## [fit] Manuals Publisher

^
Via Asset Manager

---

## [fit] Service Manual

^
Via Asset Manager

---

## [fit] Travel Advice Publisher

^
Via Asset Manager

---

## [fit] Whitehall

^
Via Whitehall admin

---

# [fit] Serving assets

---

## [fit] NFS

---

## [fit] Rails + X-Sendfile

---

## [fit] nginx

---

## [fit] Fastly CDN

---

# [fit] Problems

---

## [fit] Two apps

^
- Doing similar things
- Asset Manager
- Whitehall

---

## [fit] Three types of asset

---

### [fit] Asset Manager assets

^
- Public by default

---

### [fit] Whitehall non-attachments

^
- Equivalent to Asset Manager assets
- Public by default

---

### [fit] Whitehall attachments

^
- Draft or public
- Optionally access-limited to subset of users

---

## [fit] Virus scanning

^
- Asset Manager uses a queue
- Whitehall uses cron to regularly virus scan

---

## [fit] NFS overhead

^
- Additional infrastructure and management for hosting these assets
- E.g. Having to ensure no files are being uploaded before it can be updated/rebooted

---

## [fit] Complicated routing

---

### [fit] Asset Manager

* assets fastly
* assets-origin nginx
* static nginx
* asset-manager nginx
* asset-manager Rails
* NFS

---

### [fit] Whitehall

* www.gov.uk fastly
* www-origin nginx
* Varnish
* Router
* whitehall-frontend nginx
* whitehall-admin nginx
* whitehall Rails
* NFS

^
- Even more complicated because requests could either come via gov.uk or via the asset host

---

# [fit] Project goals

---

## [fit] Single app

---

## [fit] NFS to S3

---

# [fit] Our approach

---

## [fit] General approach

^
- Make incremental improvements
- Minimise work in progress
- So that:
  - Easier to keep everything working all the time
  - Allow us to switch focus when necessary
  - We should be able to stop at any time

---

## [fit] Prioritisation

^
- We decided to move Asset Manager to S3 early on
- If we ran out of time, we could move Whitehall to S3 but keep two separate apps
- This felt more important than having a single app

---

## [fit] Investigation

^
- Alternatives considered
  - Point assets-origin at S3
  - Redirect assets-origin to S3
  - Proxy assets-origin to S3 in nginx
  - Proxy assets-origin to S3 in Rails
  - Model attachments in the Content-Store

---

## [fit] Decision

^
- Proxy from nginx to S3
- Our performance tests suggested it was going to be performant enough
- There are links to gists in our notes that contain more info about performance testing

---

## [fit] Serving from S3

^
- Added environment variable to allow us to control the percentage of requests that the Rails app would proxy to S3 vs serving from NFS
  - Gave us a way of dialling up the rollout, and a way of turning it off if we discovered any real problems
- Avoid changing etag, last-modified and cache-control headers (by storing metadata in database) when serving from s3
  - To avoid large scale cache invalidation

---

## [fit] Whitehall to Asset Manager

^
- Started with the simpler non-attachment assets
- Started with smallest uploaders first
- Hook into CarrierWave by creating storage engine
  - Allowed us to save files to disk and AM in parallel before switching.
- Continued to support Whitehall's url structure in asset-manager in order to change as little as possible
- Redirect all non-assets-origin asset requests to assets-origin from Whitehall
  - So that we have a single place to make changes

---

# [fit] Done

---

## [fit] Storing on S3

^
- All 700GB Asset Manager and Whitehall assets are in S3
- S3 production bucket is replicated to another region and has versioning enabled
- S3 production bucket is synced to staging and integration each night
- Fake-S3 in development attempts to make it as realistic in possible without having to connect to S3

---

## [fit] Serving from S3

^
- Asset Manager serving all assets from S3
- Whitehall non-attachments being served by Asset Manager + S3
- Assets are all private on S3 - Rails generates a signed URL and passes that to nginx for it to retrieve the file

---

## [fit] Reducing NFS use

^
- Asset Manager only uses NFS for virus scanning & upload to S3
- Whitehall still using NFS for attachments

---

# [fit] Doing

---

## [fit] Asset Manager serving Whitehall attachments

^
- We're adding functionality for draft-stack like behaviour to Asset Manager

---

# [fit] Next

---

## [fit] Consolidate behaviour

^
- Between mainstream and Whitehall assets in Asset Manager
- Between Whitehall and Asset Manager
  - E.g. add image resizing & PDF thumbnailing to Asset Manager

---

## [fit] Improve virus scanning

^
- E.g.
  - Upload to S3 before virus scanning
  - Rescan when new definitions are available

---

## [fit] Improve URL structure

^
- We discussed this at length but didn't get round to doing it

---

## [fit] Rewrite content

^
- To always use the assets host
- To use the new URL structure

---

## [fit] Assets in Content Store

^
- We discounted this early on
- The changes we're making to Asset Manager will make this easier to do in future

---

# [fit] Challenges

---

## [fit] Asset hosts

^
- Whitehall assets were available at both gov.uk and the asset host
- Took a while to work out whether we could serve them all from the asset host

---

## [fit] HMRC assets

^
- For their Basic PAYE Tool.
- Continue to be served from gov.uk to avoid breaking the software

---

## [fit] Limited nginx tests

^
- It's quite easy to accidentally change the behaviour
- Nested manipulation of headers can cause problems

---

## [fit] CarrierWave versions

^
- Different versions of CarrierWave in Whitehall and Asset Manager
- Different treatment of non-ascii characters lead to exceptions when uploading from Whitehall to Asset Manager

---

## [fit] Missing Content-Types

^
- IE displaying binary content for some assets
- Turned out to be due to a missing Content-Type header in the asset response
- We had to update Asset Manager to handle conditional GETs so that we didn't proxy to S3

---

# [fit] Useful tips

^
- A couple of useful tips while working with nginx and the GOV.UK stack

---

## [fit] nginx debug logging

^
- Really helped us understand the proxied requests/responses to/from upstream servers

---

## [fit] $RANDOM

^
- Use $RANDOM to append querystring when making a request and use that $RANDOM value in Kibana to trace it

---

# [fit] More information

---

## [fit] Asset Manager GitHub issues

---

## [fit] Google Drive
