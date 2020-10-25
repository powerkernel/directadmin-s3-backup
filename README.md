# directadmin-s3-backup

DirectAdmin S3 Backup

## Features

- Auto backup upload to AWS S3
- Auto Create S3 Bucket
- Auto delete old backups after X day(s)

## Installation

Download the [.zip](https://github.com/powerkernel/directadmin-s3-backup/archive/master.zip) file, and then extract it into your DirectAdmin server at `/home/admin/tools/directadmin-s3-backup/`, then update `config.php` with your AWS access keys.

Go to extracted location and run `composer update` to download the AWS PHP-SDK.

Then download the custom s3 upload script by running those commands:

```bash
mkdri -p /usr/local/directadmin/scripts/custom
wget -O /usr/local/directadmin/scripts/custom/ftp_upload.php https://raw.githubusercontent.com/powerkernel/directadmin-s3-backup/master/upload-script.sh
chmod +x /usr/local/directadmin/scripts/custom/ftp_upload.php
```

Finally, go to `DirectAdmin \ Admin Backup/Transfer` to create Cron Schedule backup, select FTP for the backup location.

NOTE: The FTP user/pass is your DirectAdmin admin account & password, FTP IP is `127.0.0.1`.
