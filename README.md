# HAProxy Content Pack for Graylog

This content pack includes following configurations for one click setup:

-  JSON Logging 
    -   HTTP Access/Request/Captured Log

- Inputs 
    - HaProxy log - Syslog UDP

- Extractors
    - Extract JSON fields
    - Empty JSON field 
    - Reduced message to path
    - HTTP Method from haproxy_httpRequest
    - HTTP URI from haproxy_httpRequest
    - HTTP Request Protocol version from haproxy_httpRequest
    - Empty haproxy_httpRequest Field 
    - Removing parenthesis from haproxy_capturedRequestHeaders
    - Host Extraction from Captured HTTP Request
    - User Agent Extraction from Captured HTTP Request
    - HTTP Referer Extraction from Captured HTTP Request
    - HTTP XForwardedFor Extraction from Captured HTTP Request
    - Browser Extraction from haproxy_capturedHttpRequestUserAgent

- Streams
    - HAProxy
    - HAProxy HTTP 4XX
    - HTTP HTTP 5XXs

- Dashboards
    - Requests last 24h (Count)
    - Requests last 24h (Histogram)
    - HTTP 4XXs last 24h(Count)
    - HTTP 4XXs last 24h (Histogram)
    - HTTP 5XXs last 24h (Count)
    - HTTP 5XXs last 24h (Histogram)
    - Map of requests last 24h (World Map)
    - Top 5 countries with Most Requests last 24h (pie with table)
    - Frontend Connection Graph: Last 7 days (Field Graph)
    - Requests per HTTP Methods: last 24h (pie with table)
    - Response codes last 24h (pie with table)
    - Top 10 Most Requested Domains : 24h (pie with table)
    - Top 10 URLs with most requests : last 24h (pie with table)
    - Top 10 IPs with Most Requests last 24h (pie with table)
    - Top Hourly backends(pie with table)
    - Top 10 Browsers with most requests : 24h (pie with table)
    - Average Request Time (in ms) last 24 h (count)
    - Average Request Size (in bytes) last 24 h (count)
    - Time/Size last 24h (combined graph)
    - Response size (bytes) last 24h (Line Graph)
    - Response Time (ms) last 24h (Line Graph)

### Setting Up Centralized Logging with Graylog
Centralized logging is an important component of any production-grade infrastructure.Analyzing log data can help in debugging issues with your deployed applications and services, such as determining the reason for service termination or application crash.

Compatible/Tested with following versions:
- HA-Proxy version 1.5.19 2016/12/25
- Graylog 2.4.5
- Ubuntu 16.04
- Digitalocean/AWS/GCP/Azure

### Requirements
> Enabling custom HAProxy logging in JSON format
#### $ vi /etc/haproxy/haproxy.cfg 
```
global
	log 127.0.0.1 len 8096 local2
    log-send-hostname
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# expriment
	# Default ciphers to use on SSL-enabled listening sockets.
	# For more information, see ciphers(1SSL). This list is from:
	#  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
	ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS
	ssl-default-bind-options no-sslv3

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
	log-format {"haproxy_clientIP":"%ci","haproxy_clientPort":"%cp","haproxy_dateTime":"%t","haproxy_frontendNameTransport":"%ft","haproxy_backend":"%b","haproxy_serverName":"%s","haproxy_Tw":"%Tw","haproxy_Tc":"%Tc","haproxy_Tt":"%Tt","haproxy_bytesRead":"%B","haproxy_terminationState":"%ts","haproxy_actconn":%ac,"haproxy_FrontendCurrentConn":%fc,"haproxy_backendCurrentConn":%bc,"haproxy_serverConcurrentConn":%sc,"haproxy_retries":%rc,"haproxy_srvQueue":%sq,"haproxy_backendQueue":%bq,"haproxy_backendSourceIP":"%bi","haproxy_backendSourcePort":"%bp","haproxy_statusCode":"%ST","haproxy_serverIP":"%si","haproxy_serverPort":"%sp","haproxy_frontendIP":"%fi","haproxy_frontendPort":"%fp","haproxy_capturedRequestHeaders":"%hr","haproxy_httpRequest":"%r"}
	timeout connect 5000
        timeout client  50000
        timeout server  50000
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http

frontend localnodes
    capture request header Host len 30
    capture request header User-Agent len 200
    capture request header Referer len 800
    capture request header X-Forwarded-For len 20
    bind *:9290
    default_backend nodes

backend nodes
    balance roundrobin
    option forwardfor
    http-request set-header X-Forwarded-Port %[dst_port]
    http-request add-header X-Forwarded-Proto https if { ssl_fc }
    option httpchk HEAD / HTTP/1.1\r\nHost:localhost
    server web01 127.0.0.1:9200 check

```

> rsyslogd 8.16.0 
> Enabling rsyslog to receive logs on 127.0.0.1 -> UDP port 514
#### $ vi /etc/rsyslog.conf [Add at the end of Module section]
```
$ModLoad imudp
$UDPServerAddress 127.0.0.1
$UDPServerRun 514
```
#### $ vi /etc/rsyslog.d/49-haproxy.conf
```
# $MaxMessageSize 64k
# $ModLoad imudp
# $UDPServerRun 514
$template GRAYLOGRFC5424,"<%PRI%>%PROTOCOL-VERSION% %TIMESTAMP:::date-rfc3339% %HOSTNAME% %APP-NAME% %PROCID% %MSGID% %STRUCTURED-DATA% %msg%\$
local2.=info -/var/log/haproxy/haproxy.log;GRAYLOGRFC5424
local2.notice -/var/log/haproxy/haproxy-status.log;GRAYLOGRFC5424
# check your local log file location
### keep logs in localhost ##
#local2.* ~
if $syslogtag contains 'haproxy' and $msg contains 'stats' then ~
if $syslogtag contains 'haproxy' then @XXX.XX.XXX.XXX:12211;GRAYLOGRFC5424
:syslogtag, contains, "haproxy" ~
```

#### Extractors of HAProxy log input [import it if not created by content pack ~ haproxy-graylog-extractors.json]
```
# check sort order
Extract JSON fields
Empty JSON field
Reduced message to path 
HTTP Method from haproxy_httpRequest
HTTP URI from haproxy_httpRequest 
HTTP Request Protocol version from haproxy_httpRequest 
Empty haproxy_httpRequest Field
Removing parenthesis from String
Host Extraction from Captured HTTP Request
User Agent Extraction from Captured HTTP Request
HTTP Referer Extraction from Captured HTTP Request
```

#### graylog-custom-mapping.json - custom index mappings
why we need custom mapping?
haproxy_Tc,haproxy_Tt and haproxy_bytesRead fields are saved as a string.
Sometimes it’s useful to not rely on Elasticsearch’s dynamic mapping but to define a stricter schema for messages.
In order to extend the default mapping of Elasticsearch and Graylog, you can create one or more custom index mappings and add them as index templates to Elasticsearch.
- Creating a new index template
Save the following index template for the custom index mapping into a file named graylog-custom-mapping.json:
```
{
  "template": "graylog_*",
  "mappings" : {
    "message" : {
      "properties" : {
       "haproxy_Tc" : {
            "type" : "long"
          },
          "haproxy_Tt" : {
            "type" : "long"
          },
          "haproxy_Tw" : {
            "type" : "long"
          },
        "haproxy_bytesRead" : {
          "type" : "long"
        }
      }
    }
  }
}
```
Finally, load the index mapping into Elasticsearch with the following command:
```
$ curl -X PUT -d @'graylog-custom-mapping.json' 'http://localhost:9200/_template/graylog-custom-mapping?pretty'
{
  "acknowledged" : true
}
```
- Rotate graylog deflactor
    - GUI : System>Maintenance>Rotate Active write index
    - verify : ``` $ curl -X GET 'http://localhost:9200/graylog_deflector/_mapping?pretty' ```

## Screenshots
![Screenshot](/screenshot0.png?raw=true "Dashboard Screenshot")
![Screenshot](/screenshot1.png?raw=true "Dashboard Screenshot")
![Screenshot](/screenshot2.png?raw=true "Dashboard Screenshot")
----
Want to contribute? Great!
 - [Connect ->  Hariom Vashisth](mailto:vashisth.hariom7@gmail.com)

