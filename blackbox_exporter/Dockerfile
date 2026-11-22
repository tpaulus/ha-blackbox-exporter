FROM quay.io/prometheus/blackbox-exporter:v0.29.1

USER nobody
CMD ["--config.file=/config/blackbox.yml", "--web.listen-address=:9115"]
