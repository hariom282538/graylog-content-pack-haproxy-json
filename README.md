# HAProxy Content Pack for Graylog3

> Graylog2 Content Pack - Refer **graylog2_contentPack** branch

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

### Setting Up Centralized Logging with Graylog3
Centralized logging is an important component of any production-grade infrastructure.Analyzing log data can help in debugging issues with your deployed applications and services, such as determining the reason for service termination or application crash.

Compatible/Tested with following versions:
- HA-Proxy version 1.8.8-1ubuntu0.7 2019/11/04
- Graylog 3.X
- Ubuntu 18.04
- Digitalocean/AWS/GCP/Azure

### Requirements
> Enabling custom HAProxy logging in JSON format
#### $ vi /etc/haproxy/haproxy.cfg
```
# Only Mentioning changes for full configuration refer haproxy.cfg

global
	log 127.0.0.1 len 8096 local2
  log-send-hostname

defaults
	log-format {\"haproxy_clientIP\":\"%ci\",\"haproxy_clientPort\":\"%cp\",\"haproxy_dateTime\":\"%t\",\"haproxy_frontendNameTransport\":\"%ft\",\"haproxy_backend\":\"%b\",\"haproxy_serverName\":\"%s\",\"haproxy_Tw\":\"%Tw\",\"haproxy_Tc\":\"%Tc\",\"haproxy_Tt\":\"%Tt\",\"haproxy_bytesRead\":\"%B\",\"haproxy_terminationState\":\"%ts\",\"haproxy_actconn\":%ac,\"haproxy_FrontendCurrentConn\":%fc,\"haproxy_backendCurrentConn\":%bc,\"haproxy_serverConcurrentConn\":%sc,\"haproxy_retries\":%rc,\"haproxy_srvQueue\":%sq,\"haproxy_backendQueue\":%bq,\"haproxy_backendSourceIP\":\"%bi\",\"haproxy_backendSourcePort\":\"%bp\",\"haproxy_statusCode\":\"%ST\",\"haproxy_serverIP\":\"%si\",\"haproxy_serverPort\":\"%sp\",\"haproxy_frontendIP\":\"%fi\",\"haproxy_frontendPort\":\"%fp\",\"haproxy_capturedRequestHeaders\":\"%hr\",\"haproxy_httpRequest\":\"%r\"}
 

frontend Local_Server
    capture request header Host len 30
    capture request header User-Agent len 200
    capture request header Referer len 800
    capture request header X-Forwarded-For len 20
    bind 0.0.0.0:80
    mode http
    default_backend My_Web_Servers

backend My_Web_Servers
    mode http
    balance roundrobin
    option forwardfor
    http-request set-header X-Forwarded-Port %[dst_port]
    http-request add-header X-Forwarded-Proto https if { ssl_fc }
    option httpchk HEAD / HTTP/1.1rnHost:localhost
    server app1  127.0.0.1:8081
    server app2 127.0.0.1:8082

listen stats
    bind 0.0.0.0:1936
    stats enable
    stats hide-version
    stats refresh 30s
    stats show-node
    stats auth admin:admin
    stats uri  /stats

```

> rsyslogd 8.32.0 
> Enabling rsyslog to receive logs on 127.0.0.1 -> UDP port 514
#### $ vi /etc/rsyslog.conf [Add at the end of Module section - Refer rsyslog.conf] 
```
$ModLoad imudp
$UDPServerAddress 127.0.0.1
$UDPServerRun 514
```

#### $ vi /etc/rsyslog.d/49-haproxy.conf [Refer 49-haproxy.conf]
```
$template GRAYLOGRFC5424,"<%PRI%>%PROTOCOL-VERSION% %TIMESTAMP:::date-rfc3339% %HOSTNAME% %APP-NAME% %PROCID% %MSGID% %STRUCTURED-DATA% %msg%\$
local2.=info -/var/log/haproxy/haproxy.log;GRAYLOGRFC5424
local2.=info @172.31.10.107:12211;GRAYLOGRFC5424
& stop
local2.=notice -/var/log/haproxy/haproxy-status.log;GRAYLOGRFC5424
& stop
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
$ curl -X PUT -d @'graylog-custom-mapping.json' -H 'Content-Type: application/json' 'http://localhost:9200/_template/graylog-custom-mapping?pretty'
{
  "acknowledged" : true
}
```
- Rotate indices manually
    - GUI : System>Indices> | Select "Default index set" Maintenace>Rotate Active write index
    - verify : ``` $ curl -X GET 'http://localhost:9200/graylog_deflector/_mapping?pretty' ```

## Screenshots

![Screenshot](/4.png?raw=true "Dashboard Screenshot")
![Screenshot](/5.png?raw=true "Dashboard Screenshot")
![Screenshot](/6.png?raw=true "Dashboard Screenshot")
![Screenshot](/7.png?raw=true "Dashboard Screenshot")
![Screenshot](/8.png?raw=true "Dashboard Screenshot")
----
Want to contribute? Great!
 - [Connect ->  Hariom Vashisth](mailto:hariom.devops@gmail.com)

support developers/maintainers you depend on, too!
 - [Paypal ->  Hariom Vashisth](https://www.paypal.me/dreamalarm)


