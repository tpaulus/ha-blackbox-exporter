# Home Assistant Blackbox Exporter

A minimal Home Assistant add-on running the upstream Prometheus Blackbox
Exporter on a host outside Kubernetes. No Supervisor API access, host networking
or privileged capabilities are required.

Add `https://github.com/tpaulus/ha-blackbox-exporter` to the Home Assistant
add-on store repositories. See [installation and configuration](blackbox_exporter/DOCS.md).
