FROM bitnami/postgresql:14.4.0-debian-11-r12
USER root
RUN  apt-get update && \
     DEBIAN_FRONTEND=noninteractive apt-get upgrade -y && \
     DEBIAN_FRONTEND=noninteractive apt-get install curl -y apt-transport-https ca-certificates gnupg && \
     echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | tee -a /etc/apt/sources.list.d/google-cloud-sdk.list && \
     curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | tee /usr/share/keyrings/cloud.google.gpg && \
     DEBIAN_FRONTEND=noninteractive apt-get update && apt-get install -y google-cloud-cli && \
     rm -rf /var/lib/apt/lists/*
USER 1001
ENTRYPOINT [ "pg_dump" ]
