---
title: grafana持久化用户操作日志
date: 2020-05-20 22:04:10
tags:
  - grafana
categories:
  - tools
---
> Grafana v6.6.2 (3fa63cfc34)

+ [log]
```
# Either "console", "file", "syslog". Default is console and file
# Use space to separate multiple modes, e.g. "console file"
mode = console file
```
+ [paths]
```
# Directory where grafana can store logs
logs = data/log
```
+ [server]
```
# Path to where grafana can store temp files, sessions, and the sqlite3 db (if that is used)
data = data
# Log web requests
router_logging = false
```

+ docker启动
```
sudo docker run -d --name grafana-log   \
-p 3000:3000 \
-e GF_LOG_MODE="console file" \
-e GF_PATHS_LOGS='/var/log/grafana'   \
-e GF_SERVER_ROUTER_LOGGING=true \
-v /Users/blank/grafana:/var/lib/grafana   \
-v /Users/blank/grafana/log:/var/log/grafana   \
grafana
```
> 如果错误提示：【Failed to start grafana. error: failed to initialize file handler: open /var/log/grafana/grafana.log: permission denied】请配置目录/Users/blank/grafana权限为777