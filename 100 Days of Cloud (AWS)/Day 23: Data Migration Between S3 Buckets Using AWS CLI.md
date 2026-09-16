# Data Migration Between S3 Buckets Using AWS CLI

## Problem Statement

As part of a data migration project, the team lead has tasked the team with migrating data from an existing S3 bucket to a new S3 bucket. The existing bucket contains a substantial amount of data that must be accurately transferred to the new bucket. The team is responsible for creating the new S3 bucket and ensuring that all data from the existing bucket is copied or synced to the new bucket completely and accurately. It is imperative to perform thorough verification steps to confirm that all data has been successfully transferred to the new bucket without any loss or corruption.

As a member of the Nautilus DevOps Team, your task is to perform the following:

- Create a new private S3 bucket named `datacenter-sync-940266880`.
- Migrate the entire data from the existing `datacenter-s3-940266880` bucket to the new bucket.
- Ensure both buckets contain the same data.
- Use AWS CLI to create the bucket and complete the migration.

## Solution

### 1. Create a New Private S3 Bucket

Create the new bucket. If your source bucket is in a different region, change `us-east-1` accordingly.

```bash
aws s3api create-bucket --bucket datacenter-sync-940266880 --region us-east-1 --acl private
```

Example output:

```json
{
    "Location": "/datacenter-sync-940266880",
    "BucketArn": "arn:aws:s3:::datacenter-sync-940266880"
}
```

### 2. Turn on the Public Access Block to Keep It Strictly Private

This step prevents the bucket from being made public accidentally or through misconfigured policies.

```bash
aws s3api put-public-access-block \
  --bucket datacenter-sync-940266880 \
  --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

### 3. Migrate the Data

Next, copy the data from the source bucket to the destination bucket. The `sync` command compares both buckets and copies only the files that are missing or different.

```bash
aws s3 sync s3://datacenter-s3-940266880 s3://datacenter-sync-940266880
```

### 4. Verify 100% Data Consistency

Count the files in the old bucket:

```bash
aws s3 ls s3://datacenter-s3-940266880 --recursive --human-readable --summarize
```

Count the files in the new bucket:

```bash
aws s3 ls s3://datacenter-sync-940266880 --recursive --human-readable --summarize
```

Review the `Total Objects` and `Total Size` at the bottom of both outputs. If the numbers match exactly, the migration is complete and fully consistent.

## Command Explanations

### `aws s3api create-bucket`

This command creates a new Amazon S3 bucket using the AWS CLI.

- `aws`: starts the AWS CLI.
- `s3api`: uses the S3 API commands, which are useful for bucket-level operations such as creating buckets and managing access settings.
- `create-bucket`: tells AWS to create a new bucket.
- `--bucket datacenter-sync-940266880`: sets the bucket name. This must be globally unique across AWS.
- `--region us-east-1`: tells AWS to create the bucket in the `us-east-1` region. This should match your environment or the source bucket's region when needed.
- `--acl private`: sets the bucket access control to private. `ACL` stands for Access Control List. It is a way of granting or restricting permissions to users or groups. Using `private` prevents public access by default and keeps the bucket secure.

This step is required because the destination bucket must exist before data can be copied into it.

### `aws s3api put-public-access-block`

This command turns on S3 public access protections for the bucket.

- `put-public-access-block`: configures the public access settings for the bucket.
- `--bucket datacenter-sync-940266880`: selects the bucket to protect.
- `--public-access-block-configuration`: defines the access rules that should be enforced.

The configuration values are:

- `BlockPublicAcls=true`: stops anyone from setting public ACLs on the bucket or objects.
- `IgnorePublicAcls=true`: ignores any public ACLs that already exist.
- `BlockPublicPolicy=true`: stops public policies from making the bucket public.
- `RestrictPublicBuckets=true`: restricts bucket policies that would allow public access.

This step is important because even if a bucket is created as private, misconfigured policies or ACLs can still expose it publicly. The public access block adds a strong security layer and helps meet compliance and best-practice requirements.

### `aws s3 sync`

This command copies data from one S3 location to another while keeping the destination synchronized with the source.

```bash
aws s3 sync s3://datacenter-s3-940266880 s3://datacenter-sync-940266880
```

- `sync`: compares the source and destination and copies files that are missing, changed, or different.
- `s3://datacenter-s3-940266880`: the source bucket path, where the original data is stored.
- `s3://datacenter-sync-940266880`: the destination bucket path, where the migrated data should be placed.

This is the main migration step. It is efficient because it transfers only the necessary data and helps ensure that the new bucket mirrors the original bucket.

### `aws s3 ls --recursive --human-readable --summarize`

This command lists the contents of an S3 bucket and summarizes the total object count and size.

```bash
aws s3 ls s3://datacenter-s3-940266880 --recursive --human-readable --summarize
```

```bash
aws s3 ls s3://datacenter-sync-940266880 --recursive --human-readable --summarize
```

- `ls`: lists bucket contents.
- `s3://...`: points to the bucket being inspected.
- `--recursive`: includes all files and folders inside the bucket, not just top-level entries.
- `--human-readable`: shows file sizes in a format easier for humans to read, such as `KB`, `MB`, or `GB`.
- `--summarize`: adds the total object count and total size for the bucket.

This verification step is necessary to confirm that the data was copied completely and without loss. If the totals match between the source and destination buckets, it indicates that the migration was successful and consistent.

## Final Note

The workflow follows a secure and reliable pattern:

1. Create a private destination bucket.
2. Enforce public access restrictions.
3. Copy all source data into the new bucket.
4. Compare the object count and size to confirm the migration is complete.

This ensures both security and data integrity throughout the migration process.



