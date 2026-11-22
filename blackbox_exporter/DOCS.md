# Installation

1. Add `https://github.com/tpaulus/ha-blackbox-exporter` to the add-on store
   repositories and install **Blackbox Exporter**. Home Assistant builds the
   small image locally using the pinned upstream multi-architecture image.
2. Copy the example `blackbox.yml` from this repository to
   `/share/blackbox_exporter/blackbox.yml` on Home Assistant. Create the
   directory first. The file must be readable by the container's `nobody` user.
   The add-on intentionally fails to start if configuration is missing or invalid.
3. Add any deployment-specific modules to that file, then start the add-on.
   Enable **Start on boot** and **Watchdog**. After configuration changes,
   restart the add-on. The watchdog checks the exporter, not target health.
4. Verify `http://HOME_ASSISTANT_IP:9115/-/healthy` and `/metrics` from the
   Prometheus host. Use the HA host's actual LAN IP, not a cluster ingress.

The add-on exposes an **unauthenticated** probe API on TCP 9115.
Restrict access to trusted monitoring clients using your network firewall.
Do not expose it to the Internet or untrusted VLANs: callers can ask it to
connect to arbitrary destinations and read probe diagnostics. The read-only
share mount is for configuration; do not put credentials in this repository.

# Prometheus

The exporter probes on demand. Prometheus must scrape `/probe` for each
target; scraping `/metrics` alone does not test a service.

```yaml
scrape_configs:
  - job_name: external-tcp
    scrape_interval: 15s
    scrape_timeout: 10s
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets: ['SERVICE_IP:443']
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: HOME_ASSISTANT_IP:9115
  - job_name: external-blackbox-exporter
    static_configs:
      - targets: ['HOME_ASSISTANT_IP:9115']
```

Alert on `probe_success == 0`, separately from scrape failures (`up == 0`)
or missing probe metrics. `up == 1` means the exporter answered, not that
the target worked.

For HTTPS to a numeric VIP, configure both `http.headers.Host` and
`http.tls_config.server_name` in a dedicated module. Leave certificate
verification enabled and disable redirects so requests cannot escape the VIP.
Compare with direct NodePort probes to distinguish routing from application
failures. Do not use ICMP to test Kubernetes service IPs.

# Limits

TCP probes check connection establishment, not application correctness or
client-certificate authentication. A single external host tests one source
path and cannot guarantee coverage of every router ECMP next hop.
