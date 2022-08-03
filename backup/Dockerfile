FROM bitnami/postgresql:14.4.0-debian-11-r12
USER root
RUN  apt-get update && \
     DEBIAN_FRONTEND=noninteractive apt-get upgrade -y && \
     DEBIAN_FRONTEND=noninteractive apt-get install curl -y apt-transport-https ca-certificates gnupg locales && \
     echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" \
     > /etc/apt/sources.list.d/google-cloud-sdk.list && \
     curl https://packages.cloud.google.com/apt/doc/apt-key.gpg > /usr/share/keyrings/cloud.google.gpg && \
     DEBIAN_FRONTEND=noninteractive apt-get update && apt-get install -y google-cloud-cli && \
     rm -rf /var/lib/apt/lists/*
RUN sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen && locale-gen
RUN useradd -u 1001 -m postgres
USER postgres
ENV LANG en_US.UTF-8  
ENV LANGUAGE en_US:en  
ENV LC_ALL en_US.UTF-8    
CMD [ "sleep","99d" ]
