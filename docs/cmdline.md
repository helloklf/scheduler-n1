## 命令行
- scene-daemon本身是个可执行文件，主要功能是代替GUI执行需要ROOT权限的操作
- 它集成的的一些依赖，被暴露可以在shell里使用


#### zip 压缩/解压

```sh
alias scene-daemon="/data/data/com.omarea.vtools/files/scene-daemon"

# 压缩
# scene-daemon zip <srcFolder> <out.zip>
scene-daemon zip /cache/test1 test.zip

# 解压
# scene-daemon unzip <src.zip> <outFolder>
scene-daemon unzip test.zip /cache/test2
```


#### sql

```sh
alias scene-daemon="/data/data/com.omarea.vtools/files/scene-daemon"

# SELECT 之类带返回内容的操作
# scene-daemon sql <sql>
DB=/data/data/com.oplus.cosa/databases/db_game_database
scene-daemon sql $DB "select package_name, sdk_config, game_zone from PackageConfigBean where package_name='com.tencent.tmgp.sgame'"
# com.tencent.tmgp.sgame|{"recognize_interval":"1000","wake_up_times":"10","key_thread_ux":"2","white_list":["tmgp.sgame","UnityChoreograp","UnityMain"],"bind_core":"1","pipeline":{"3":"Job.worker 1","4":"CoreThread","5":"UnityGfxDeviceW","7":"UnityMain"},"bind_list":{"UnityMain":"g_11","UnityGfxDeviceW":"g_10","Thread-":"g_10","NativeThread":"g_10","AudioTrack":"g_10","Loading.AsyncRe":"g_10","CoreThread":"g_10","Job.worker 1":"g_10","GVoiceUtil":"g_10","sf_app_wakee":"g_11"},"uad_ux":"1","preempt":["cent.tmgp.sgame","Thread-","UnityMain","cent.tmgp.sgame","__render__","Job.worker 1","CoreThread"]}

# 等同于
# sqlite3 $DB "select package_name, sdk_config, game_zone from PackageConfigBean where package_name='com.tencent.tmgp.sgame'"


# 替代sqlite3进行数据库raw查询的示例 ——这个案例演示的是，在ColorOS中清除所有游戏的官调线程分配策略
alias sqlite3="/data/data/com.omarea.vtools/files/scene-daemon sql"
DB=/data/data/com.oplus.cosa/databases/db_game_database
umount "$DB" 2>/dev/null
am force-stop com.oplus.cosa
sqlite3 "$DB" "PRAGMA wal_checkpoint(TRUNCATE);" > /dev/null
cp -f "$DB" "$DB.copy"
chown $(stat -c %u:%g $DB) $DB.copy
chmod $(stat -c %a $DB) $DB.copy
chcon $(ls -Z $DB | cut -d' ' -f1) $DB.copy
# sqlite3 "$DB.copy" "UPDATE PackageConfigBean SET game_zone=replace(game_zone,'\"bind_core\":\"1\"','\"bind_core\":\"0\"') WHERE package_name='com.tencent.tmgp.sgame'; PRAGMA wal_checkpoint(TRUNCATE);" > /dev/null
sqlite3 "$DB.copy" "DELETE FROM PackageConfigBean; PRAGMA wal_checkpoint(TRUNCATE);" > /dev/null
mount --bind "$DB.copy" "$DB"

```

