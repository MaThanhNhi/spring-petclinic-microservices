FROM prom/prometheus:latest
ADD prometheus.yml /etc/prometheus/
ADD http-errors-alert.yml /etc/prometheus/

ENTRYPOINT [ "/bin/prometheus" ]
CMD [ "--config.file=/etc/prometheus/prometheus.yml", \
    "--enable-feature=exemplar-storage", \
    "--storage.tsdb.path=/prometheus" ]