---
title: "mysql notes"
date: 2017-10-11
lastmod: 2017-10-11
draft: false
tags: ["tech", "database", "mysql", "notes"]
categories: ["tech"]
description: "mysql使用笔记"
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: true
# reward: false
# mathjax: false
---

# read-only user
Adding the new MySQL user

Connect to your database as root, then add your new user like so:

CREATE USER 'lele'@'%' IDENTIFIED BY 'lele_lelelele';
The % here means the user 'tester' connecting from any host, you can place a network hostname here instead if you want to restrict access further. Naturally you will also want to substitute password with something a little stronger ;-)

Now run the following to grant the SELECT privilage to the new user on all databases:

GRANT SELECT ON *.* TO 'lele'@'%';
Or if you want to restrict access to only one database:

GRANT SELECT ON DATABASE.* TO 'lele'@'%';

# error

1. (1136, "Column count doesn't match value count at row 1"), 碰到这种错误, 排除正常的对齐错误, 100%都是你的字段名用了mysql的保留关键字.

# SQL
select a.id from stock_stock a left join stock_product b on a.jancode=b.jancode where b.jancode is null;

# Isolation
1. 'mysql> SELECT @@global.tx_isolation;'

# export & import

`mysqldump -uroot -p1222 -h 127.0.0.1 dbname tab1 tab2 tab3 > dump.sql`
`mysql -uroot -p1222 -h 127.0.0.1 dbname < dump.sql`


# mysql backup to google storage [^2]
描述怎么把lelewu mysql定时备份到google storage
## 安装percona-xtrabackup[^1]

```
yum install http://www.percona.com/downloads/percona-release/redhat/0.1-4/percona-release-0.1-4.noarch.rpm
yum install percona-xtrabackup-24
```

## 配置备份用户
需要创建数据库用户和系统用户

1. 创建数据库用户和授权
``` shell
CREATE USER 'tacy_lee'@'localhost' IDENTIFIED BY 'tacy_lee';
GRANT RELOAD, LOCK TABLES, REPLICATION CLIENT, CREATE TABLESPACE, PROCESS, SUPER, CREATE, INSERT, SELECT ON *.* TO 'tacy_lee'@'localhost';
FLUSH PRIVILEGES;

```

2. 创建系统用户

这里需要注意一下, 因为需要备份到google storage, 我用tacy_lee, 否则包oauth2认证错误

``` shell
sudo usermod -aG mysql tacy_lee
SELECT @@datadir; //获取mysql数据文件位置
sudo find /var/lib/mysql -type d -exec chmod 750 {} \;  //权限设置为750
```

## 创建Mysql备份配置参数

``` shell
$ sudo vi /etc/my.cnf.d/backup.conf
[client]
user=tacy_lee
password=tacy_lee

$ sudo chown tacy_lee /etc/my.cnf.d/backup.cnf
$ sudo chown 600 /etc/my.cnf.d/backup.cnf

```

## 备份脚本
### /usr/local/bin/backup-mysql.sh
``` shell
#!/bin/bash

export LC_ALL=C

days_of_backups=2  # Must be less than 7
backup_owner="tacy_lee"
parent_dir="/home/tacy_lee/backups"
defaults_file="/etc/my.cnf.d/backup.cnf"
todays_dir="${parent_dir}/$(date +%a)"
log_file="${todays_dir}/backup-progress.log"
now="$(date +%m-%d-%Y_%H-%M-%S)"
processors="$(nproc --all)"

# Use this to echo to standard error
error () {
    printf "%s: %s\n" "$(basename "${BASH_SOURCE}")" "${1}" >&2
    exit 1
}

trap 'error "An unexpected error occurred."' ERR

sanity_check () {
    # Check user running the script
    if [ "$USER" != "$backup_owner" ]; then
        error "Script can only be run as the \"$backup_owner\" user"
    fi
}

set_options () {
    # List the xtrabackup arguments
    xtrabackup_args=(
        "--defaults-file=${defaults_file}"
        "--backup"
        "--extra-lsndir=${todays_dir}"
        "--compress"
        "--stream=xbstream"
        "--parallel=${processors}"
        "--compress-threads=${processors}"
        "--slave-info"
    )

    backup_type="full"

    # Add option to read LSN (log sequence number) if a full backup has been
    # taken today.
    if grep -q -s "to_lsn" "${todays_dir}/xtrabackup_checkpoints"; then
        backup_type="incremental"
        lsn=$(awk '/to_lsn/ {print $3;}' "${todays_dir}/xtrabackup_checkpoints")
        xtrabackup_args+=( "--incremental-lsn=${lsn}" )
    fi
}

rotate_old () {
    # Remove the oldest backup in rotation
    day_dir_to_remove="${parent_dir}/$(date --date="${days_of_backups} days ago" +%a)"

    if [ -d "${day_dir_to_remove}" ]; then
        rm -rf "${day_dir_to_remove}"
    fi
}

take_backup () {
    # Make sure today's backup directory is available and take the actual backup
    mkdir -p "${todays_dir}"
    find "${todays_dir}" -type f -name "*.incomplete" -delete
    xtrabackup "${xtrabackup_args[@]}" --target-dir="${todays_dir}" > "${todays_dir}/${backup_type}-${now}.xbstream.incomplete" 2> "${log_file}"

    mv "${todays_dir}/${backup_type}-${now}.xbstream.incomplete" "${todays_dir}/${backup_type}-${now}.xbstream"

    # upload backup to google storage
	if [ ${backup_type} == 'full' ]; then
	  gsutil cp -r ${todays_dir} gs://lelewu_mysql_backup
	else
	  gsutil cp ${todays_dir}/${log_file} gs://lelewu_mysql_backup/$(date +%a)
	  gsutil cp ${todays_dir}/xtrabackup_info gs://lelewu_mysql_backup/$(date +%a)
	  gsutil cp ${todays_dir}/xtrabackup_checkpoints gs://lelewu_mysql_backup/$(date +%a)
	  gsutil cp ${todays_dir}/${backup_type}-${now}.xbstream gs://lelewu_mysql_backup/$(date +%a)
	fi
}

sanity_check && set_options && rotate_old && take_backup

# Check success and print message
if tail -1 "${log_file}" | grep -q "completed OK"; then
    printf "Backup successful!\n"
    printf "Backup created at %s/%s-%s.xbstream\n" "${todays_dir}" "${backup_type}" "${now}"
else
    error "Backup failure! Check ${log_file} for more information"
fi
```

### /usr/local/bin/extract-mysql.sh
解压缩脚本, 运行该脚本会在解压之前的备份到当前目录的restore目录下

``` shell
#!/bin/bash

export LC_ALL=C

backup_owner="tacy_lee"
log_file="extract-progress.log"
number_of_args="${#}"
processors="$(nproc --all)"

# Use this to echo to standard error
error () {
    printf "%s: %s\n" "$(basename "${BASH_SOURCE}")" "${1}" >&2
    exit 1
}

trap 'error "An unexpected error occurred.  Try checking the \"${log_file}\" file for more information."' ERR

sanity_check () {
    # Check user running the script
    if [ "${USER}" != "${backup_owner}" ]; then
        error "Script can only be run as the \"${backup_owner}\" user"
    fi

    # Check whether the qpress binary is installed
    if ! command -v qpress >/dev/null 2>&1; then
        error "Could not find the \"qpress\" command.  Please install it and try again."
    fi

    # Check whether any arguments were passed
    if [ "${number_of_args}" -lt 1 ]; then
        error "Script requires at least one \".xbstream\" file as an argument."
    fi
}

do_extraction () {
    for file in "${@}"; do
        base_filename="$(basename "${file%.xbstream}")"
        restore_dir="./restore/${base_filename}"

        printf "\n\nExtracting file %s\n\n" "${file}"

        # Extract the directory structure from the backup file
        mkdir --verbose -p "${restore_dir}"
        xbstream -x -C "${restore_dir}" < "${file}"

        xtrabackup_args=(
            "--parallel=${processors}"
            "--decompress"
        )

        xtrabackup "${xtrabackup_args[@]}" --target-dir="${restore_dir}"
        find "${restore_dir}" -name "*.qp" -exec rm {} \;

        printf "\n\nFinished work on %s\n\n" "${file}"

    done > "${log_file}" 2>&1
}

sanity_check && do_extraction "$@"

ok_count="$(grep -c 'completed OK' "${log_file}")"

# Check the number of reported completions.  For each file, there is an
# informational "completed OK".  If the processing was successful, an
# additional "completed OK" is printed. Together, this means there should be 2
# notices per backup file if the process was successful.
if (( $ok_count !=  $# )); then
    error "It looks like something went wrong. Please check the \"${log_file}\" file for additional information"
else
    printf "Extraction complete! Backup directories have been extracted to the \"restore\" directory.\n"
fi
```

### /usr/local/bin/prepare-mysql.sh
restore之前需要做prepare操作, 这是prepare脚本, 脚本会apply增量备份, 所以如果你需要移除一些增量备份, 你只需要移除restore里面的对应目录就行了
``` shell
#!/bin/bash

export LC_ALL=C

shopt -s nullglob
incremental_dirs=( ./incremental-*/ )
full_dirs=( ./full-*/ )
shopt -u nullglob

backup_owner="tacy_lee"
log_file="prepare-progress.log"
full_backup_dir="${full_dirs[0]}"

# Use this to echo to standard error
error() {
    printf "%s: %s\n" "$(basename "${BASH_SOURCE}")" "${1}" >&2
    exit 1
}

trap 'error "An unexpected error occurred.  Try checking the \"${log_file}\" file for more information."' ERR

sanity_check () {
    # Check user running the script
    if [ "${USER}" != "${backup_owner}" ]; then
        error "Script can only be run as the \"${backup_owner}\" user."
    fi

    # Check whether a single full backup directory are available
    if (( ${#full_dirs[@]} != 1 )); then
        error "Exactly one full backup directory is required."
    fi
}

do_backup () {
    # Apply the logs to each of the backups
    printf "Initial prep of full backup %s\n" "${full_backup_dir}"
    xtrabackup --prepare --apply-log-only --target-dir="${full_backup_dir}"

    for increment in "${incremental_dirs[@]}"; do
        printf "Applying incremental backup %s to %s\n" "${increment}" "${full_backup_dir}"
        xtrabackup --prepare --apply-log-only --incremental-dir="${increment}" --target-dir="${full_backup_dir}"
    done

    printf "Applying final logs to full backup %s\n" "${full_backup_dir}"
    xtrabackup --prepare --target-dir="${full_backup_dir}"
}

sanity_check && do_backup > "${log_file}" 2>&1

# Check the number of reported completions.  Each time a backup is processed,
# an informational "completed OK" and a real version is printed.  At the end of
# the process, a final full apply is performed, generating another 2 messages.
ok_count="$(grep -c 'completed OK' "${log_file}")"

if (( ${ok_count} == ${#full_dirs[@]} + ${#incremental_dirs[@]} + 1 )); then
    cat << EOF
Backup looks to be fully prepared.  Please check the "prepare-progress.log" file
to verify before continuing.

If everything looks correct, you can apply the restored files.

First, stop MySQL and move or remove the contents of the MySQL data directory:

        sudo systemctl stop mysql
        sudo mv /var/lib/mysql/ /tmp/

Then, recreate the data directory and  copy the backup files:

        sudo mkdir /var/lib/mysql
        sudo xtrabackup --copy-back --target-dir=${PWD}/$(basename "${full_backup_dir}")

Afterward the files are copied, adjust the permissions and restart the service:

        sudo chown -R mysql:mysql /var/lib/mysql
        sudo find /var/lib/mysql -type d -exec chmod 750 {} \\;
        sudo systemctl start mysql
EOF
else
    error "It looks like something went wrong.  Check the \"${log_file}\" file for more information."
fi
```

## 设置定时任务

``` shell
crontab -e (注意是设置tacy_lee的定时任务)
0 */3 * * * /usr/local/bin/backup-mysql.sh
```


# tips
## 导出csv

select * from sample INTO OUTFILE '/var/lib/mysql-files/orders.csv'

## 常用sql

select deptid,COUNT(*) as TotalCount from staff group by deptid having count(*) > 2


[^1]: [Installing Percona XtraBackup on Red Hat Enterprise Linux and CentOS](https://www.percona.com/doc/percona-xtrabackup/LATEST/installation/yum_repo.html)

[^2]: [How To Configure MySQL Backups with Percona XtraBackup on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-configure-mysql-backups-with-percona-xtrabackup-on-ubuntu-16-04)
