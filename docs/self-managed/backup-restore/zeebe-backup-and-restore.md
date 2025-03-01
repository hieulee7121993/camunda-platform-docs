---
id: zeebe-backup-and-restore
title: "Backup and restore Zeebe data"
description: "Create a backup of a running Zeebe cluster comprised of a consistent snapshot of all partitions."
keywords: ["backup", "backups"]
---

A backup of a Zeebe cluster is comprised of a consistent snapshot of all partitions. The backup is taken asynchronously in the background while Zeebe is processing. Thus, the backups can be taken with minimal impact on normal processing. The backups can be used to restore a cluster in case of failures that lead to full data loss or data corruption.

Zeebe provides a REST API to create backups, query, and manage existing backups.
The backup management API is a custom endpoint `backups`, available via [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/2.7.x/reference/htmlsingle/#actuator.endpoints). This is accessible via the management port of the gateway. The API documentation is also available as [OpenApi specification](https://github.com/camunda/zeebe/blob/main/dist/src/main/resources/api/backup-management-api.yaml).

The backups are stored to an external data storage. [S3](https://aws.amazon.com/s3/) or any S3-compatible storage is supported as the backup storage.

## Prerequisites

To use the backup feature in Zeebe, the following configurations must be provided:

- Enable backups by setting the flag `ZEEBE_BROKER_EXPERIMENTAL_FEATURES_ENABLEBACKUP` to `true`.
- Ensure the configuration `MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE` includes the endpoint "backups". Set `MANAGEMENT_ENDPOINTS_BACKUPS_ENABLED` to `true`.
  For additional configurations such as security, refer to the [Spring Boot documentation](https://docs.spring.io/spring-boot/docs/2.7.x/reference/htmlsingle/#actuator.endpoints).
- An external S3-compatible storage must be configured as the backup store.

### Configure S3 backup store

Configure backup store in the configuration file as follows:

```
zeebe:
  broker:
    data:
      backup:
        store: S3
        s3:
          bucketName:
          basePath:
          region:
          endpoint:
          accessKey:
          secretKey:
```

Alternatively, you can configure backup store using environment variables:

- `ZEEBE_BROKER_DATA_BACKUP_STORE` - Specify which storage to use as the backup storage. Currently, only S3 is supported. You can use any S3-compatible storage.
- `ZEEBE_BROKER_DATA_BACKUP_S3_BUCKETNAME` - The backup is stored in this bucket. **The bucket must already exist**.
- `ZEEBE_BROKER_DATA_BACKUP_S3_BASEPATH` - If the bucket is shared with other Zeebe clusters, a unique basePath must be configured.
- `ZEEBE_BROKER_DATA_BACKUP_S3_ENDPOINT` - If no endpoint is provided, it is determined based on the configured region.
- `ZEEBE_BROKER_DATA_BACKUP_S3_REGION` - If no region is provided, it is determined [from the environment](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/region-selection.html#automatically-determine-the-aws-region-from-the-environment).
- `ZEEBE_BROKER_DATA_BACKUP_S3_ACCESSKEY` - If either `accessKey` or `secretKey` is not provided, the credentials are determined [from the environment](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/credentials.html#credentials-chain).
- `ZEEBE_BROKER_DATA_BACKUP_S3_SECRETKEY` - Specify the secret key.

The same configuration must be provided to all brokers in a cluster.

#### Backup Encryption

Zeebe does not support backup encryption natively, but it _can_ use encrypted S3 buckets. For AWS S3, this means [enabling default bucket encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/default-bucket-encryption.html).

Using default bucket encryption gives you control over the encryption keys and algorithms while being completely transparent with Zeebe.

Combined with TLS between Zeebe and the S3 API, backups are fully encrypted in transit and at rest. Other S3 compatible services might have similar features that should work as well.

#### Backup compression

Backups can be large depending on your usage of Zeebe. To reduce S3 storage costs and upload times, you can enable backup compression.

Zeebe compresses backup data immediately before uploading to S3 and buffers the compressed files in a temporary directory. Compression and buffering of compressed files can have a negative effect if Zeebe is heavily resource constrained.

You can enable compression by specifying a compression algorithm to use. We recommend using [zstd] as it provides a good trade off between compression ratio and resource usage.

More compression algorithms are available; check [commons-compress] for a full list.

```yaml
zeebe:
  broker:
    data:
      backup:
        store: s3
        s3:
          compression: zstd
```

Alternatively, you can configure backup compression using an environment variable: `ZEEBE_BROKER_DATA_BACKUP_S3_COMPRESSION`.

[zstd]: https://github.com/facebook/zstd
[commons-compress]: https://commons.apache.org/proper/commons-compress/

## Create backup API

The following request can be used to start a backup.

### Request

```
POST actuator/backups
{
  "backupId": <backupId>
}
```

A `backupId` is an integer and must be greater than the id of previous backups.
Zeebe does not take two backups with the same ids. If a backup fails, a new `backupId` must be provided to trigger a new backup.

<details>
  <summary>Example request</summary>

```
curl --request POST 'http://localhost:9600/actuator/backups' \
-H 'Content-Type: application/json' \
-d '{ "backupId": "100" }'
```

</details>

### Response

| Code             | Description                                                                                                              |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------ |
| 202 Accepted     | A Backup has been successfully scheduled. To determine if the backup process was completed, refer to the GET API.        |
| 400 Bad Request  | Indicates issues with the request, for example when the `backupId` is not valid or backup is not enabled on the cluster. |
| 409 Conflict     | Indicates a backup with the same `backupId` or a higher id already exists.                                               |
| 500 Server Error | All other errors. Refer to the returned error message for more details.                                                  |
| 502 Bad Gateway  | Zeebe has encountered issues while communicating to different brokers.                                                   |
| 504 Timeout      | Zeebe failed to process the request within a pre-determined timeout.                                                     |

<details>
  <summary>Example response body with 202 Accepted</summary>

```json
{
  "message": "A backup with id 100 has been scheduled. Use GET actuator/backups/100 to monitor the status."
}
```

</details>

## Get backup info API

Information about a specific backup can be retrieved using the following request:

### Request

```
GET actuator/backups/{backupId}
```

<details>
  <summary>Example request</summary>

```
curl --request GET 'http://localhost:9600/actuator/backups/100'
```

</details>

### Response

| Code             | Description                                                                                |
| ---------------- | ------------------------------------------------------------------------------------------ |
| 200 OK           | Backup state could be determined and is returned in the response body (see example below). |
| 400 Bad Request  | There is an issue with the request. Refer to the returned error message for details.       |
| 404 Not Found    | A backup with that ID does not exist.                                                      |
| 500 Server Error | All other errors. Refer to the returned error message for more details.                    |
| 502 Bad Gateway  | Zeebe has encountered issues while communicating to different brokers.                     |
| 504 Timeout      | Zeebe failed to process the request within a pre-determined timeout.                       |

When the response is 200 OK, the response body consists of a JSON object describing the state of the backup.

- `backupId`: Id in the request.
- `state`: Gives the overall status of the backup. The state can be one of the following:
  - `COMPLETED` if all partitions have completed the backup.
  - `FAILED` if at least one partition has failed. In this case, `failureReason` contains a string describing the reason for failure.
  - `INCOMPLETE` if at least one partition's backup does not exist.
  - `IN_PROGRESS` if at least one partition's backup is in progress.
- `details`: Gives the state of each partition's backup.
- `failureReason`: The reason for failure if the state is `FAILED`.

<details>
  <summary>Example response body with 200 OK</summary>

```json
{
  "backupId": 100,
  "details": [
    {
      "brokerVersion": "8.2.0-SNAPSHOT",
      "checkpointPosition": 5,
      "createdAt": "2022-12-08T13:00:55.344276672Z",
      "lastUpdatedAt": "2022-12-08T13:00:55.805351556Z",
      "partitionId": 1,
      "snapshotId": "2-1-3-2",
      "state": "COMPLETED"
    },
    {
      "brokerVersion": "8.2.0-SNAPSHOT",
      "checkpointPosition": 7,
      "createdAt": "2022-12-08T13:00:55.370965069Z",
      "lastUpdatedAt": "2022-12-08T13:00:55.84756566Z",
      "partitionId": 2,
      "snapshotId": "3-1-5-3",
      "state": "COMPLETED"
    }
  ],
  "state": "COMPLETED"
}
```

</details>

## List backups API

Information about all backups can be retrieved using the following request:

### Request

```
GET actuator/backups
```

<details>
  <summary>Example request</summary>

```
curl --request GET 'http://localhost:9600/actuator/backups'
```

</details>

### Response

| Code             | Description                                                                                |
| ---------------- | ------------------------------------------------------------------------------------------ |
| 200 OK           | Backup state could be determined and is returned in the response body (see example below). |
| 400 Bad Request  | There is an issue with the request. Refer to returned error message for details.           |
| 500 Server Error | All other errors. Refer to the returned error message for more details.                    |
| 502 Bad Gateway  | Zeebe has encountered issues while communicating to different brokers.                     |
| 504 Timeout      | Zeebe failed to process the request with in a pre-determined timeout.                      |

When the response is 200 OK, the response body consists of a JSON object with a list of backup info.
See [get backup info API response](#response-1) for the description of each field.

<details>
  <summary>Example response body with 200 OK</summary>

```json
[
  {
    "backupId": 100,
    "details": [
      {
        "brokerVersion": "8.2.0-SNAPSHOT",
        "createdAt": "2022-12-08T13:00:55.344276672Z",
        "partitionId": 1,
        "state": "COMPLETED"
      },
      {
        "brokerVersion": "8.2.0-SNAPSHOT",
        "createdAt": "2022-12-08T13:00:55.370965069Z",
        "partitionId": 2,
        "state": "COMPLETED"
      }
    ],
    "state": "COMPLETED"
  },
  {
    "backupId": 200,
    "details": [
      {
        "brokerVersion": "8.2.0-SNAPSHOT",
        "createdAt": "2022-12-08T13:01:15.27750375Z",
        "partitionId": 1,
        "state": "COMPLETED"
      },
      {
        "brokerVersion": "8.2.0-SNAPSHOT",
        "createdAt": "2022-12-08T13:01:15.279995106Z",
        "partitionId": 2,
        "state": "COMPLETED"
      }
    ],
    "state": "COMPLETED"
  }
]
```

</details>

## Delete backup API

A backup can be deleted using the following request:

### Request

```
DELETE actuator/backups/{backupId}
```

<details>
  <summary>Example request</summary>

```
curl --request DELETE 'http://localhost:9600/actuator/backups/100'
```

</details>

### Response

| Code             | Description                                                                      |
| ---------------- | -------------------------------------------------------------------------------- |
| 204 No Content   | The backup has been deleted.                                                     |
| 400 Bad Request  | There is an issue with the request. Refer to returned error message for details. |
| 500 Server Error | All other errors. Refer to the returned error message for more details.          |
| 502 Bad Gateway  | Zeebe has encountered issues while communicating to different brokers.           |
| 504 Timeout      | Zeebe failed to process the request with in a pre-determined timeout.            |

## Restore

A new Zeebe cluster can be created from a specific backup. Camunda provides a standalone app which must be run on each node where a Zeebe broker will be running. This is a Spring Boot application similar to the broker and can run using the binary provided as part of the distribution. The app can be configured the same way a broker is configured - via environment variables or using the configuration file located in `config/application.yaml`.

To restore a Zeebe cluster, run the following in each node where the broker will be running:

```
tar -xzf zeebe-distribution-X.Y.Z.tar.gz -C zeebe/
./bin/restore --backupId=<backupId>
```

If restore was successful, the app exits with a log message of `Successfully restored broker from backup`.

Restore fails if:

- There is no valid backup with the given backupId.
- Backup store is not configured correctly.
- The configured data directory is not empty.
- Any other unexpected errors.

If the restore fails, you can re-run the application after fixing the root cause.

:::note
When restoring, provide the same configuration (node id, data directory, cluster size, and replication count) as the broker that will be running in this node. The partition count must be same as in the backup.
:::
