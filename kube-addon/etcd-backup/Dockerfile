ARG VERSION=v3.5.6
FROM quay.io/coreos/etcd:${VERSION}
COPY --chmod=0766 mc /usr/bin/
RUN apt-get update \
    && apt-get install -y ca-certificates \
    && update-ca-certificates
COPY --chmod=0766 entrypoint.sh /opt/entrypoint.sh
ENTRYPOINT [ "bash", "/opt/entrypoint.sh" ]