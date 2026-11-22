# Installation

1. Add `https://github.com/tpaulus/ha-blackbox-exporter` to the add-on store
   repositories and install **Blackbox Exporter**. Home Assistant builds the
   small image locally using the pinned upstream multi-architecture image.
2. Create `app_configs/local_blackbox_exporter/blackbox.yml` using this
   repository's example, then add any deployment-specific modules. Access the
   app configuration folder through File Editor, SSH, or Samba. The folder is included in app backups.
3. Start the add-on. It reads the file at `/config/blackbox.yml`.
   Enable **Start on boot** and **Watchdog**. After configuration changes,
   restart the add-on. The watchdog checks the exporter, not target health.
4. Verify `http://HOME_ASSISTANT_IP:9115/-/healthy` and `/metrics` from the
   Prometheus host. Use the HA host's actual LAN IP, not a cluster ingress.

The add-on exposes an **unauthenticated** probe API on TCP 9115.
Restrict access to trusted monitoring clients using your network firewall.
Do not expose it to the Internet or untrusted VLANs: callers can ask it to
connect to arbitrary destinations and read probe diagnostics. The app
configuration mount is read-only.

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
