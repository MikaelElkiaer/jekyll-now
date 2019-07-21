---
layout: post
title: ELK stack for Docker services
---

### For running elasticsearch on Unraid OS
```
#/etc/sysctl.conf
vm.max_map_count = 262144
```

`> sysctl --system`

Create new file and add
```
#/boot/config/sysctl.config
vm.max_map_count = 262144
```

Enable loading of new file on boot:
```
#/boot/config/go
/sbin/sysctl -p /boot/config/sysctl.conf
```

### Add Cloudflare IPs when using CDN
```
#/etc/nginx/conf.d/realip.conf
#Cloudflare IPs
set_real_ip_from 173.245.48.0/20;
set_real_ip_from 103.21.244.0/22;
set_real_ip_from 103.22.200.0/22;
set_real_ip_from 103.31.4.0/22;
set_real_ip_from 141.101.64.0/18;
set_real_ip_from 108.162.192.0/18;
set_real_ip_from 190.93.240.0/20;
set_real_ip_from 188.114.96.0/20;
set_real_ip_from 197.234.240.0/22;
set_real_ip_from 198.41.128.0/17;
set_real_ip_from 162.158.0.0/15;
set_real_ip_from 104.16.0.0/12;
set_real_ip_from 172.64.0.0/13;
set_real_ip_from 131.0.72.0/22;

real_ip_header x-forwarded-for;
```

### Add request host to elasticsearch logs
```
#/usr/share/filebeat/module/nginx/access/ingest/default.json
\"?%{IPORHOST:http.request.host} %{IP_LIST:nginx.access.remote_ip_list} - %{DATA:user.name} \[%{HTTPDATE:nginx.access.time}\] \"%{GREEDYDATA:nginx.access.info}\" %{NUMBER:http.response.status_code:long} %{NUMBER:http.response.body.bytes:long} \"%{DATA:http.request.referrer}\" \"%{DATA:user_agent.original}\"
```

In case pipeline has already been created before above changes:

`DELETE _ingest/pipeline/filebeat-7.2.0-nginx-access-default`
