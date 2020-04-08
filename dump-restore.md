mongodump reads data from a MongoDB database and creates high fidelity BSON files which the mongorestore tool can use to populate a MongoDB database. mongodump and mongorestore are simple and efficient tools for backing up and restoring small MongoDB deployments, but are not ideal for capturing backups of larger systems."

https://docs.mongodb.com/manual/core/backups/


```shell
#!/bin/sh

# Make sure to:
# 1) Name this file `backup.sh` and place it in /home/ubuntu
# 2) Run sudo apt-get install awscli to install the AWSCLI
# 3) Run aws configure (enter s3-authorized IAM user and specify region)
# 4) Fill in DB host + name
# 5) Create S3 bucket for the backups and fill it in below (set a lifecycle rule to expire files older than X days in the bucket)
# 6) Run chmod +x backup.sh
# 7) Test it out via ./backup.sh
# 8) Set up a daily backup at midnight via `crontab -e`:
#    0 0 * * * /home/ubuntu/backup.sh > /home/ubuntu/backup.log

# DB host (secondary preferred as to avoid impacting primary performance)
HOST=db.example.com

# DB name
DBNAME=my-db

# S3 bucket name
BUCKET=s3-bucket-name

# Linux user account
USER=ubuntu

# Current time
TIME=`/bin/date +%d-%m-%Y-%T`

# Backup directory
DEST=/home/$USER/tmp

# Tar file of backup directory
TAR=$DEST/../$TIME.tar

# Create backup dir (-p to avoid warning if already exists)
/bin/mkdir -p $DEST

# Log
echo "Backing up $HOST/$DBNAME to s3://$BUCKET/ on $TIME";

# Dump from mongodb host into backup directory
/usr/bin/mongodump -h $HOST -d $DBNAME -o $DEST

# Create tar of backup directory
/bin/tar cvf $TAR -C $DEST .

# Upload tar to s3
/usr/bin/aws s3 cp $TAR s3://$BUCKET/

# Remove tar file locally
/bin/rm -f $TAR

# Remove backup directory
/bin/rm -rf $DEST

# All done
echo "Backup available at https://s3.amazonaws.com/$BUCKET/$TIME.tar"
```



**Restore from Backup**

Download the .tar backup to the server from the S3 bucket via wget or curl:

`wget -O backup.tar https://s3.amazonaws.com/my-bucket/xx-xx-xxxx-xx:xx:xx.tar`
Alternatively, use the awscli to download it securely.

Then, extract the tar archive:

`tar xvf backup.tar`
Finally, import the backup into a MongoDB host:

`mongorestore --host {db.example.com} --db {my-db} {db-name}/`



**Simple script to backup MongoDB to S3, without waste diskspace for temp files. And a way to restore from the latest snapshot.**

```shell
#!/bin/sh
set -e
HOST=localhost
DB=test-entd-products
COL=asimproducts

S3PATH="s3://mongodb-backups-test1-entd/$DB/$COL/"
S3BACKUP=$S3PATH`date +"%Y%m%d_%H%M%S"`.dump.gz
S3LATEST=$S3PATH"latest".dump.gz
/usr/bin/aws s3 mb $S3PATH
/usr/bin/mongodump -h $HOST -d $DB -c $COL -o - | gzip -9 | aws s3 cp - $S3BACKUP
aws s3 cp $S3BACKUP $S3LATEST

# Restore
echo -n "Restore: "
echo -n "aws s3 cp $S3LATEST - |gzip -d  | mongorestore --host $HOST --db $DB -c $COL - "
```

* simpler way to do archieve `mongodump --archive --gzip | aws s3 cp - s3://my-bucket/some-file`

Small mods to make this work with array of db names

```shell
#!/bin/bash

# Make sure to:
# 1) Name this file `backup.sh` and place it in /home/ubuntu
# 2) Run sudo apt-get install awscli to install the AWSCLI
# 3) Run aws configure (enter s3-authorized IAM user and specify region)
# 4) Fill in DB host + name
# 5) Create S3 bucket for the backups and fill it in below (set a lifecycle rule to expire files older than X days in the bucket)
# 6) Run chmod +x backup.sh
# 7) Test it out via ./backup.sh
# 8) Set up a daily backup at midnight via `crontab -e`:
#    0 0 * * * /home/ubuntu/backup.sh > /home/ubuntu/backup.log

# DB host (secondary preferred as to avoid impacting primary performance)
HOST=localhost

# DB name
DBNAMES=("db1" "db2" "db3")

# S3 bucket name
BUCKET=bucket

# Linux user account
USER=ubuntu

# Current time
TIME=`/bin/date +%d-%m-%Y-%T`

# Backup directory
DEST=/home/$USER/tmp

# Tar file of backup directory
TAR=$DEST/../$TIME.tar

# Create backup dir (-p to avoid warning if already exists)
/bin/mkdir -p $DEST

# Log
echo "Backing up $HOST/$DBNAME to s3://$BUCKET/ on $TIME";

# Dump from mongodb host into backup directory
for DBNAME in "${DBNAMES[@]}"
do
   /usr/bin/mongodump -h $HOST -d $DBNAME -o $DEST
done

# Create tar of backup directory
/bin/tar cvf $TAR -C $DEST .

# Upload tar to s3
/usr/bin/aws s3 cp $TAR s3://$BUCKET/

# Remove tar file locally
/bin/rm -f $TAR

# Remove backup directory
/bin/rm -rf $DEST

# All done
echo "Backup available at https://s3.amazonaws.com/$BUCKET/$TIME.tar"
```


Ref: https://gist.github.com/eladnava/96bd9771cd2e01fb4427230563991c8d
