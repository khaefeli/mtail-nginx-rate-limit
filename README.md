# mtail-nginx-rate-limit
Mtail parser for Nginx error logs to create (Prometheus) metrics about delayed / limited requests.

## Tools
 * Nginx
 * mtail exporter
 * Prometheus / Grafana
 
## Nginx rate limiting

### Why
Rate limit your nginx site to avoid clients to:
 * Bruce force (e.g. `/login/`)
 * DOS your expensive calls (e.g. `/search/`)
 * Slow down your site due to hard crawling
 * etc etc

### Why not
Limiting the req/s for `binary_remote_addr` will not really help, if a botnet is (D)dosing your site.

### How
There are several good guides how to setup a rate limit for Nginx. 
I recommend the official [blog post](https://www.nginx.com/blog/rate-limiting-nginx/).

### Example
To apply new req. limits to prod. application, it might be better to learn first more about your application and the current traffic.
The following example only logs requests and doesn't block them (yet).

Create shared memory zones and set a request limit per zone.

```
 # create available zones / request limits
limit_req_zone $binary_remote_addr zone=small:10m rate=5r/s;
limit_req_zone $binary_remote_addr zone=medium:10m rate=15r/s;
limit_req_zone $binary_remote_addr zone=large:10m rate=50r/s;
```
With a zone size of 10mb shared memory, your Nginx could analyse up to 160'000 IPs at the same time.

Use the `$binary_remote_addr` (aka IP as hash) as key to define the request characteristic. 
You can also go further and use e.g. http headers or application UUIDs.

Afterwards the zone `small` is definied for the whole `server{}` block:
`limit_req zone=small burst=10000;`

The burst of 10000 req/s makes sure, no requests are getting a 503 error (as long as you're in the learning mode)
It only logs clients with more than 5req/s to your nginx error log and delay them.

After adjusting the req/s and maybe setting stricter limits to more detailed paths (e.g. `location ~ ^/login` with only 1req/s)
You might also want to apply the `nodelay` parameter to not slow down requests and set better bursts (e.g. 5 for the login path)

```
2018/04/10 09:11:02 [error] 42525#42525: *552003110 limiting requests, excess: 15.935 by zone "small", client: 8.8.8.8, server: example.com, request: "GET /login HTTP/1.0", host: "example.com"
```


## Monitoring

### Logs

Collect the error logs and put them into your ELK stack. 
The following config is for the old-school td-agent which logs into elastic / kibana.

```
<match td.logstash.nginx.errorlog>
  @type elasticsearch
  host efk.example.com
  port 9200
  logstash_format true
  logstash_prefix nginx-errorlog
  flush_interval 10s # for testing
</match>
```

```
<source>
  type tail
  path /var/log/nginx/error.log
  pos_file /var/log/td-agent/error.pos
  tag td.logstash.nginx.errorlog
  format /^(?<time>[^ ]+ [^ ]+) \[(?<log_level>.*)\] (?<pid>[0-9]*).(?<tid>[^:]*): (?<message>.*), client:(?<remote_addr>.*), server:(?<server>.*), request:(?<request>.*), host:(?<host>.*)/
</source>
```

### Metrics
After storing your logs for deeper analysis, you should collect timeseries and setup some alerting and dashboards.
The mtail config in this repo listens to your error log and exports the **delayed req /s** metrics.

`/usr/bin/mtail -logs /var/log/nginx/error.log -progs /etc/mtail/nginx_error.mtail -port 9449 -logtostderr`

Note: you need the mtail binary.

Ingest the metrics to your prometheus and create some fancy grafana graphs.

![Ratelimit](/ratelimit.png)
