# Two setup processes, one for MinIO and one for Postgres

# Docker

```bash

```

# Postgres

```bash
docker exec -it postgres /bin/sh

psql -U myuser -d postgres -c "CREATE TABLE IF NOT EXISTS events (id SERIAL PRIMARY KEY, event_name VARCHAR(255), bucket_name VARCHAR(255), object_key VARCHAR(255), size INT, eTag VARCHAR(255), sequencer VARCHAR(255), data JSONB);"

exit
```

# Minio
First insure mc is installed on your host, then run:

```bash
# Set up MinIO alias on the host
mc alias set myminio http://localhost:9000 minio minio123

# Create a bucket
mc mb myminio/mybucket

# Set up notification configuration
mc admin config set myminio notify_postgres:1 connection_string="user=myuser password=mypassword host=host.docker.internal dbname=postgres port=5432 sslmode=disable" table="events" format="namespace"

# Restart MinIO server
mc admin service restart myminio

# Add event notification
mc event add myminio/mybucket arn:minio:sqs::1:postgresql --event put,get,delete

# Create a test file and copy it to the bucket
echo "Hello MinIO" > testfile.txt
mc cp testfile.txt myminio/mybucket/
```

# To Test & Validate


```bash
# Check events in PostgreSQL
docker exec -it postgres psql -U myuser -d postgres -c "SELECT * FROM events;"
```