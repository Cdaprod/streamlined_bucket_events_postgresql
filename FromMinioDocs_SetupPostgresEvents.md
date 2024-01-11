Publish MinIO events via PostgreSQL

NOTE: Until release RELEASE.2020-04-10T03-34-42Z PostgreSQL notification used to support following options:

host                (hostname)           Postgres server hostname (used only if `connection_string` is empty)
port                (port)               Postgres server port, defaults to `5432` (used only if `connection_string` is empty)
username            (string)             database username (used only if `connection_string` is empty)
password            (string)             database password (used only if `connection_string` is empty)
database            (string)             database name (used only if `connection_string` is empty)
These are now deprecated, if you plan to upgrade to any releases after RELEASE.2020-04-10T03-34-42Z make sure to migrate to only using connection_string option. To migrate, once you have upgraded all the servers use the following command to update the existing notification targets.

mc admin config set myminio/ notify_postgres[:name] connection_string="host=hostname port=2832 username=psqluser password=psqlpass database=bucketevents"
Please make sure this step is carried out, without this step PostgreSQL notification targets will not work, an error message will be shown on the console upon server upgrade/restart, make sure to follow the above instructions appropriately. For further questions please join our https://slack.min.io
Install PostgreSQL database server. For illustrative purposes, we have set the "postgres" user password as password and created a database called minio_events to store the events.

This notification target supports two formats: namespace and access.

When the namespace format is used, MinIO synchronizes objects in the bucket with rows in the table. It creates rows with two columns: key and value. The key is the bucket and object name of an object that exists in MinIO. The value is JSON encoded event data about the operation that created/replaced the object in MinIO. When objects are updated or deleted, the corresponding row from this table is updated or deleted respectively.

When the access format is used, MinIO appends events to a table. It creates rows with two columns: event_time and event_data. The event_time is the time at which the event occurred in the MinIO server. The event_data is the JSON encoded event data about the operation on an object. No rows are deleted or modified in this format.

The steps below show how to use this notification target in namespace format. The other format is very similar and is omitted for brevity.

Step 1: Ensure PostgresSQL minimum requirements are met

MinIO requires PostgreSQL version 9.5 or above. MinIO uses the INSERT ON CONFLICT (aka UPSERT) feature, introduced in version 9.5 and the JSONB data-type introduced in version 9.4.

Step 2: Add PostgreSQL endpoint to MinIO

The PostgreSQL configuration is located in the notify_postgresql key. Create a configuration key-value pair here for your PostgreSQL instance. The key is a name for your PostgreSQL endpoint, and the value is a collection of key-value parameters described in the table below.

KEY:
notify_postgres[:name]  publish bucket notifications to Postgres databases

ARGS:
connection_string*   (string)             Postgres server connection-string e.g. "host=localhost port=5432 dbname=minio_events user=postgres password=password sslmode=disable"
table*               (string)             DB table name to store/update events, table is auto-created
format*              (namespace*|access)  'namespace' reflects current bucket/object list and 'access' reflects a journal of object operations, defaults to 'namespace'
queue_dir            (path)               staging dir for undelivered messages e.g. '/home/events'
queue_limit          (number)             maximum limit for undelivered messages, defaults to '100000'
max_open_connections (number)             maximum number of open connections to the database, defaults to '2'
comment              (sentence)           optionally add a comment to this setting
or environment variables

KEY:
notify_postgres[:name]  publish bucket notifications to Postgres databases

ARGS:
MINIO_NOTIFY_POSTGRES_ENABLE*              (on|off)             enable notify_postgres target, default is 'off'
MINIO_NOTIFY_POSTGRES_CONNECTION_STRING*   (string)             Postgres server connection-string e.g. "host=localhost port=5432 dbname=minio_events user=postgres password=password sslmode=disable"
MINIO_NOTIFY_POSTGRES_TABLE*               (string)             DB table name to store/update events, table is auto-created
MINIO_NOTIFY_POSTGRES_FORMAT*              (namespace*|access)  'namespace' reflects current bucket/object list and 'access' reflects a journal of object operations, defaults to 'namespace'
MINIO_NOTIFY_POSTGRES_QUEUE_DIR            (path)               staging dir for undelivered messages e.g. '/home/events'
MINIO_NOTIFY_POSTGRES_QUEUE_LIMIT          (number)             maximum limit for undelivered messages, defaults to '100000'
MINIO_NOTIFY_POSTGRES_COMMENT              (sentence)           optionally add a comment to this setting
MINIO_NOTIFY_POSTGRES_MAX_OPEN_CONNECTIONS (number)             maximum number of open connections to the database, defaults to '2'
NOTE: If the max_open_connections key or the environment variable MINIO_NOTIFY_POSTGRES_MAX_OPEN_CONNECTIONS is set to 0, There will be no limit set on the number of open connections to the database. This setting is generally NOT recommended as the behavior may be inconsistent during recursive deletes in namespace format.
MinIO supports persistent event store. The persistent store will backup events when the PostgreSQL connection goes offline and replays it when the broker comes back online. The event store can be configured by setting the directory path in queue_dir field and the maximum limit of events in the queue_dir in queue_limit field. For eg, the queue_dir can be /home/events and queue_limit can be 1000. By default, the queue_limit is set to 100000.

Note that for illustration here, we have disabled SSL. In the interest of security, for production this is not recommended. To update the configuration, use mc admin config get command to get the current configuration.

$ mc admin config get myminio notify_postgres
notify_postgres:1 queue_dir="" connection_string="" queue_limit="0"  table="" format="namespace"
Use mc admin config set command to update the configuration for the deployment. Restart the MinIO server to put the changes into effect. The server will print a line like SQS ARNs: arn:minio:sqs::1:postgresql at start-up if there were no errors.

mc admin config set myminio notify_postgres:1 connection_string="host=localhost port=5432 dbname=minio_events user=postgres password=password sslmode=disable" table="bucketevents" format="namespace"
Note that, you can add as many PostgreSQL server endpoint configurations as needed by providing an identifier (like "1" in the example above) for the PostgreSQL instance and an object of per-server configuration parameters.

Step 3: Enable PostgreSQL bucket notification using MinIO client

We will now enable bucket event notifications on a bucket named images. Whenever a JPEG image is created/overwritten, a new row is added or an existing row is updated in the PostgreSQL configured above. When an existing object is deleted, the corresponding row is deleted from the PostgreSQL table. Thus, the rows in the PostgreSQL table, reflect the .jpg objects in the images bucket.

To configure this bucket notification, we need the ARN printed by MinIO in the previous step. Additional information about ARN is available here.

With the mc tool, the configuration is very simple to add. Let us say that the MinIO server is aliased as myminio in our mc configuration. Execute the following:

# Create bucket named `images` in myminio
mc mb myminio/images
# Add notification configuration on the `images` bucket using the MySQL ARN. The --suffix argument filters events.
mc event add myminio/images arn:minio:sqs::1:postgresql --suffix .jpg
# Print out the notification configuration on the `images` bucket.
mc event list myminio/images
mc event list myminio/images
arn:minio:sqs::1:postgresql s3:ObjectCreated:*,s3:ObjectRemoved:* Filter: suffix=".jpg"
Step 4: Test on PostgreSQL

Open another terminal and upload a JPEG image into images bucket.

mc cp myphoto.jpg myminio/images
Open PostgreSQL terminal to list the rows in the bucketevents table.

$ psql -h 127.0.0.1 -U postgres -d minio_events
minio_events=# select * from bucketevents;

key                 |                      value
--------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 images/myphoto.jpg | {"Records": [{"s3": {"bucket": {"arn": "arn:aws:s3:::images", "name": "images", "ownerIdentity": {"principalId": "minio"}}, "object": {"key": "myphoto.jpg", "eTag": "1d97bf45ecb37f7a7b699418070df08f", "size": 56060, "sequencer": "147CE57C70B31931"}, "configurationId": "Config", "s3SchemaVersion": "1.0"}, "awsRegion": "", "eventName": "s3:ObjectCreated:Put", "eventTime": "2016-10-12T21:18:20Z", "eventSource": "aws:s3", "eventVersion": "2.0", "userIdentity": {"principalId": "minio"}, "responseElements": {}, "requestParameters": {"sourceIPAddress": "[::1]:39706"}}]}
(1 row)