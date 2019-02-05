# directadmin-s3-backup
DirectAdmin S3 Backup

Features
--------
- Auto backup upload to AWS S3
- Auto Create S3 Bucket
- Auto delete old backups after X day(s)

Installation
------------

Download the [.zip](https://github.com/powerkernel/directadmin-s3-backup/archive/master.zip) file, and then extract it into your DirectAdmin at a location you choose. Then update `config.php` with your AWS access keys.

We assume that the .zip is extracted at `/home/admin/tools/directadmin-s3-backup`, run `composer update` to download the AWS PHP-SDK.

Create `ftp_upload.php` file in `/usr/local/directadmin/scripts/custom` with the following content:
```
#!/bin/sh
HOSTNAME=hostname -f;
if [ ${ftp_ip} == $HOSTNAME ] && [ ${ftp_username} == "admin" ]; then
	RET=0;
	php /home/admin/tools/directadmin-s3-backup/ftp_upload_s3.php $ftp_local_file $ftp_remote_file 2>&1
	RET=$?	
else
	FTPPUT=/usr/bin/ncftpput
	CURL=/usr/local/bin/curl
	OS=`uname`;
	DU=/usr/bin/du
	BC=/usr/bin/bc
	EXPR=/usr/bin/expr
	TOUCH=/bin/touch
	PORT=${ftp_port}
	FTPS=0
	MD5=${ftp_md5}

	if [ "${ftp_secure}" = "ftps" ]; then
		FTPS=1
	fi

	#######################################################
	# SETUP

	if [ ! -e $TOUCH ] && [ -e /usr/bin/touch ]; then
		TOUCH=/usr/bin/touch
	fi
	if [ ! -x ${EXPR} ] && [ -x /bin/expr ]; then
		EXPR=/bin/expr
	fi

	if [ ! -e "${ftp_local_file}" ]; then
		echo "Cannot find backup file ${ftp_local_file} to upload";

		/bin/ls -la ${ftp_local_path}

		/bin/df -h

		exit 11;
	fi

	get_md5() {
		MF=$1

		if [ ${OS} = "FreeBSD" ]; then
			MD5SUM=/sbin/md5
		else
			MD5SUM=/usr/bin/md5sum
		fi
		if [ ! -x ${MD5SUM} ]; then
			return
		fi

		if [ ! -e ${MF} ]; then
			return
		fi

		if [ ${OS} = "FreeBSD" ]; then
			FMD5=`$MD5SUM -q $MF`
		else
			FMD5=`$MD5SUM $MF | cut -d\  -f1`
		fi

		echo "${FMD5}"
	}

	#######################################################

	CFG=${ftp_local_file}.cfg
	/bin/rm -f $CFG
	$TOUCH $CFG
	/bin/chmod 600 $CFG

	RET=0;


	#######################################################
	TIMEOUT=120

	#dynamic timeout for nctpput.
	#Curl kicks the control connection with keep-alive pings by default.
	SIZE_GIG=0
	SECONDS_PER_GIG=120
	if [ -x ${DU} ]; then
		if [ "${OS}" = "FreeBSD" ]; then
			SIZE_GIG=`BLOCKSIZE=G ${DU} -A ${ftp_local_file} | cut -f1`
		else
			SIZE_GIG=`${DU} --apparent-size --block-size=1G ${ftp_local_file} | cut -f1`
		fi

		if [ "${SIZE_GIG}" -gt 1 ]; then
			NEW_TIMEOUT=$TIMEOUT

			if [ -x ${BC} ]; then
				NEW_TIMEOUT=`echo "${SIZE_GIG} * ${SECONDS_PER_GIG}" | ${BC}`
			elif [ -x ${EXPR} ]; then
				NEW_TIMEOUT=`${EXPR} ${SIZE_GIG} \* ${SECONDS_PER_GIG}`
			else
				echo "Cannot find ${BC} nor ${EXPR} for ftp upload timeout change on large file: ${SIZE_GIG} Gig.";
			fi

			#make sure it's a useful number
			if [ "${NEW_TIMEOUT}" -gt "${TIMEOUT}" ]; then
				TIMEOUT=${NEW_TIMEOUT};
			fi
		fi
	fi

	#######################################################
	# FTP
	upload_file()
	{
		if [ ! -e $FTPPUT ]; then
			echo "";
			echo "*** Backup not uploaded ***";
			echo "Please install $FTPPUT by running:";
			echo "";
			echo "cd /usr/local/directadmin/scripts";
			echo "./ncftp.sh";
			echo "";
			exit 10;
		fi

		/bin/echo "host $ftp_ip" >> $CFG
		/bin/echo "user $ftp_username" >> $CFG
		/bin/echo "pass $ftp_password" >> $CFG

		if [ ! -s ${CFG} ]; then
			echo "${CFG} is empty. ncftpput is not going to be happy about it.";
			ls -la ${CFG}
			ls -la ${ftp_local_file}
			df -h
		fi

		$FTPPUT -f $CFG -V -t ${TIMEOUT} -P $PORT -m "$ftp_path" "$ftp_local_file" 2>&1
		RET=$?

		if [ "${RET}" -ne 0 ]; then
			echo "ncftpput return code: $RET";
		fi
	}

	#######################################################
	# FTPS
	upload_file_ftps()
	{
		if [ ! -e ${CURL} ]; then
			CURL=/usr/bin/curl
		fi

		if [ ! -e ${CURL} ]; then
			echo "";
			echo "*** Backup not uploaded ***";
			echo "Please install curl by running:";
			echo "";
			echo "cd /usr/local/directadmin/custombuild";
			echo "./build curl";
			echo "";
			exit 10;
		fi

		/bin/echo "user =  \"$ftp_username:$ftp_password\"" >> $CFG

		if [ ! -s ${CFG} ]; then
			echo "${CFG} is empty. curl is not going to be happy about it.";
			ls -la ${CFG}
			ls -la ${ftp_local_file}
			df -h
		fi

		#ensure ftp_path ends with /
		ENDS_WITH_SLASH=`echo "$ftp_path" | grep -c '/$'`
		if [ "${ENDS_WITH_SLASH}" -eq 0 ]; then
			ftp_path=${ftp_path}/
		fi

		${CURL} --config ${CFG} --ftp-ssl -k --silent --show-error --ftp-create-dirs --upload-file $ftp_local_file  ftp://$ftp_ip:${PORT}/$ftp_path$ftp_remote_file 2>&1
		RET=$?

		if [ "${RET}" -ne 0 ]; then
			echo "curl return code: $RET";
		fi
	}

	#######################################################
	# Start

	if [ "${FTPS}" = "1" ]; then
		upload_file_ftps
	else
		upload_file
	fi

	if [ "${RET}" = "0" ] && [ "${MD5}" = "1" ]; then
		MD5_FILE=${ftp_local_file}.md5
		M=`get_md5 ${ftp_local_file}`
		if [ "${M}" != "" ]; then
			echo "${M}" > ${MD5_FILE}

			ftp_local_file=${MD5_FILE}
			ftp_remote_file=${ftp_remote_file}.md5

			if [ "${FTPS}" = "1" ]; then
				upload_file_ftps
			else
				upload_file
			fi
		fi
	fi

	/bin/rm -f $CFG

	exit $RET
fi
```

Remember to replace `HOSTNAME` with your server hostname. 

Finally, go to `DirectAdmin \ Admin Backup/Transfer` to create Cron Schedule backup, select FTP for the backup location.

NOTE: The FTP user/pass is your DirectAdmin admin account & password, FTP IP is server hostname (ser1.domain.com).
