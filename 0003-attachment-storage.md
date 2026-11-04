# ADR-0003: Store TimeCapsule attachments in Amazon S3

- **Status:** Accepted
- **Date:** 2026-11-04
- **Project:** TimeCapsule
- **Scope:** Storage of user-uploaded capsule attachments

## Context

Before Lab 4, the PHP OOP TimeCapsule application stored capsule attachments in `storage/uploads/` on the `timecapsule-dev-app` EC2 instance. The application used a `Storage` class to save, delete, and stream local files. A replacement instance would not automatically have those files; running multiple application instances would introduce a file-sharing problem.

The database remains PostgreSQL on `timecapsule-dev-db`, and the existing network topology (VPC, public/private subnets, bastion, routes) remains unchanged. The decision concerns **attachments**, not database records or PHP session persistence.

## Decision

Use the private Amazon S3 bucket `timecapsule-dev-uploads-yakovlev` in `eu-central-1`. Store attachment objects under `uploads/<random-file-name>`.

- `Storage::save()` uses AWS SDK for PHP `PutObject`, sets the object's content type, and returns the generated file name.
- `Storage::delete()` uses `DeleteObject`; with versioning enabled, ordinary deletion creates a delete marker rather than destroying all object versions.
- `Storage::stream()` signs a `GetObject` request for five minutes and replies to the browser with HTTP `302 Location`. The browser retrieves the file directly from S3. Response headers preserve the MIME type and original download filename.
- Application code running on EC2 obtains temporary AWS credentials through its instance IAM role `timecapsule-dev-app-role`; no static AWS access keys are stored in `.env`.
- Authorization to view a capsule is enforced by TimeCapsule **before** generating the URL; possession of the URL then grants temporary access to the signed object.

The final IAM policy allows only `s3:GetObject`, `s3:PutObject`, and `s3:DeleteObject` on `arn:aws:s3:::timecapsule-dev-uploads-yakovlev/uploads/*`. The initial `s3:ListBucket` permission required for migration was subsequently removed. No `s3:DeleteObjectVersion` permission is granted to the application.

## Alternatives considered

| Option | Advantages | Disadvantages for this project |
|---|---|---|
| **EC2 EBS disk** | Existing file I/O works unchanged; simple for one server | Attachment data depends on instance/volume management; shared access and server replacement are more complex |
| **Amazon EFS** | Shared POSIX-style filesystem; usable by multiple EC2 instances | Requires mounting, network filesystem access, and further infrastructure; no native S3 presigned object URLs |
| **Amazon S3 — chosen** | Object storage independent of EC2; versioning and lifecycle; IAM integration; direct browser downloads | Requires SDK integration and handling object-storage semantics rather than local paths |

## Consequences

### File access and link validity

An authenticated/authorized user requests `/capsules/{id}/file`. TimeCapsule checks access and creates a signed S3 `GetObject` URL. It returns an HTTP 302 redirect, after which the browser downloads from S3 without the application server proxying file bytes. Each generated URL has an `X-Amz-Expires=300` lifetime (five minutes); it can also stop working earlier if the underlying temporary signing credentials expire or permission is revoked. After expiry, the client must request a new URL from the application, which performs authorization again. The application does not make the bucket public.

### Deletion protection and storage costs

Bucket versioning retains previous object versions across overwrite and ordinary `DeleteObject` operations. The application's role cannot invoke `DeleteObjectVersion`. This protects historical versions against permanent deletion using the application's credentials, although privileged administrators still can remove versions.

The lifecycle rule `timecapsule-dev-noncurrent` applies only to the `uploads/` prefix. It schedules transition of eligible noncurrent versions to Standard-IA after 30 days, permanent expiration after 90 days while retaining the two newest noncurrent versions, and abort of incomplete multipart uploads after seven days. Very small objects may be excluded from transition by S3's minimum-size behavior. `backups/` is outside the rule's scope. The bucket policy denies non-HTTPS requests, and Block Public Access remains enabled.

### Scaling and operational consequences

Any future application instance behind a load balancer can use the same S3 bucket and an appropriate instance role. No shared attachment directory or inter-server file synchronization is necessary. Application sessions remain a **separate** scaling concern: the current PHP session directory is local, so horizontal scaling still requires a shared session solution or appropriately designed stateless authentication. The PostgreSQL database also remains an external dependency.

Regional resilience is not provided by a single-region S3 bucket alone: if the design must withstand an entire AWS Region outage, consider cross-region replication, recovery of application/database infrastructure, and corresponding access/endpoint changes.

## Evidence / implementation

- [Lab 4 report](../lab4/README.md)
- [Updated architecture diagram](../lab4/architecture.drawio)
- [Implementation commit supplied for Lab 4](https://github.com/my-php-deploying-labs-organization/timecapsule/commit/98144757197c00e1794cb3ef1d18fd5843ccf5b0)
