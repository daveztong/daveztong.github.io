---
title: 日志清理脚本
date: 2016-12-06 11:21:32
tags: shell
---



日志过多，crontab to the rescue!

清理X天前的日志:

```shell
logs_dir="/path/to/logs"

# default 7+1 day ago
days_ago=1

if [ ! -z "$1" ]; then
    echo "delete logs $1 days ago"
    days_ago=$1
else
   echo "delete logs $days_ago days ago"
fi
# 日志文件名格式:datacenter.2016-12-02.25.log
# 取出日期
recent_date=$(ls -t $logs_dir | head -10 | grep  -Eo "[[:digit:]]{4}-[[:digit:]]+-[[:digit:]]+" | head -1)
# delete logs older than 7 days
recent_date=$(date -d "$recent_date -7days" +%F)
for (( d=1; d<=$days_ago; d++))
do
    xdaysago=$(date -d "$recent_date -$d days" +%F)
    file_pattern="$logs_dir/datacenter.$xdaysago.*"
    echo "rm $file_pattern"
    $(rm $file_pattern)
done
echo "done!"
```

Then add a cron job:

```shell
crontab -e
@weekly sh /opt/ruoshui/bin/clear_logs.sh 8
```



