## Cadvisor+collectd

Monitoring role for deploying and managing [Cadvisor](https://github.com/google/cadvisor) and [collectd](https://collectd.org) bundle in Docker containers, based on [maier/cadvisor-collectd](https://github.com/maier/cadvisor-collectd).
Sends collected metrics to host's UDP port 25826, those can be easily analyzed via Logstash.

## User-customizable variables

You can use these variables to customize your cadvisor+collectd installations.

| var | description | default |
|-----|-------------|---------|
|`cadvisor_image`|Cadvisor Docker registry image| [google/cadvisor](https://registry.hub.docker.com/u/google/cadvisor/)
|`collectd_image`|collectd Docker registry image| [corebug/cadvisor-collectd](https://registry.hub.docker.com/u/corebug/cadvisor-collectd)
|`cadvisor_ports`|ports to publish for Cadvisor web interface|-p 8085:8080

