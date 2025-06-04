FROM grafana/loki:latest

# Copy configuration file
COPY loki-config.yml /etc/loki/local-config.yaml

# Define command to run Loki
ENTRYPOINT ["/usr/bin/loki", "-config.file=/etc/loki/local-config.yaml"]