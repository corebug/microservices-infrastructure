## Logstash

Logging role for deploying and managing [Logstash](http://logstash.net) 1.5 with Docker and systemd.

## Variables

You can use these variables to customize your Logstash installations:

| var | description | default |
|-----|-------------|---------|
|`logstash_input_log4j`|Read events over a TCP socket from a Log4j SocketAppender|yes|
|`logstash_input_docker`|Read Docker containers JSON type logs|yes|
|`logstash_input_mesos`|Read Mesos logs|yes|
|`logstash_input_syslog`|Read RFC3164 syslog messages|yes|
|`logstash_input_collectd`|Read events from the collectd binary protocol|yes|
|`logstash_output_webhdfs`|Store events via WebHDFS using gzip compression|yes|
|`logstash_output_logentries`|Forward logs to [logentries](https://logentries.com/)|no|
|`logstash_logentries_token`|Unique token provided by logentries|no|
