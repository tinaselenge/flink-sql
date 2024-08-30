FROM registry.access.redhat.com/ubi9/ubi:latest

# Set environment variables
ENV JAVA_VERSION=17
ENV JAVA_HOME=/usr/lib/jvm/java-$JAVA_VERSION-openjdk

# Install Java and dependencies
RUN dnf install -y \
    java-$JAVA_VERSION-openjdk \
    java-$JAVA_VERSION-openjdk-devel \
    wget \
    && dnf clean all

ENV PATH="$JAVA_HOME/bin:$PATH"

# Grab gosu for easy step-down from root
ENV GOSU_VERSION 1.11
RUN set -ex && \
  wget -nv -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(uname -m | sed -e 's/x86_64/amd64/' -e 's/aarch64/arm64/' -e 's/ppc64le/ppc64el/')" && \
  wget -nv -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(uname -m | sed -e 's/x86_64/amd64/' -e 's/aarch64/arm64/' -e 's/ppc64le/ppc64el/').asc" && \
  export GNUPGHOME="$(mktemp -d)" && \
#  NOTE: None of these key servers worked
#  for server in ha.pool.sks-keyservers.net $(shuf -e \
#                          hkp://p80.pool.sks-keyservers.net:80 \
#                          keyserver.ubuntu.com \
#                          hkp://keyserver.ubuntu.com:80 \
#                          pgp.mit.edu) ; do \
#      gpg --batch --keyserver "$server" --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 && break || : ; \
#  done && \
  gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 && \
  gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu && \
  gpgconf --kill all && \
  rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc && \
  chmod +x /usr/local/bin/gosu && \
  gosu nobody true

# Configure Flink version
ENV FLINK_VERSION=1.19
ENV FLINK_FULL_VERSION=1.19.1
ENV SCALA_VERSION=2.12
ENV FLINK_KAFKA_CONNECTOR_VERSION=3.2.0-1.19
ENV AVRO_CONFLUENT_REGISTRY_VERSION=1.19.1
ENV FLINK_TGZ_URL=https://dlcdn.apache.org/flink/flink-$FLINK_FULL_VERSION/flink-$FLINK_FULL_VERSION-bin-scala_2.12.tgz \
    FLINK_ASC_URL=https://downloads.apache.org/flink/flink-$FLINK_FULL_VERSION/flink-$FLINK_FULL_VERSION-bin-scala_2.12.tgz.asc \
    GPG_KEY=6378E37EB3AAEA188B9CB0D396C2914BB78A5EA1 \
    CHECK_GPG=true

# Prepare environment
ENV FLINK_HOME=/opt/flink
ENV FLINK_LIB_DIR=$FLINK_HOME/lib
ENV PATH=$FLINK_HOME/bin:$PATH
RUN groupadd --system --gid=9999 flink && \
    useradd --system --home-dir $FLINK_HOME --uid=9999 --gid=flink flink
WORKDIR $FLINK_HOME

# Install Flink
RUN set -ex && \
  wget -nv -O flink.tgz "$FLINK_TGZ_URL" && \
  if [ "$CHECK_GPG" = "true" ]; then \
    wget -nv -O flink.tgz.asc "$FLINK_ASC_URL"; \
    export GNUPGHOME="$(mktemp -d)"; \
    gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys "$GPG_KEY"; \
#  NOTE: None of these key servers worked
#    for server in ha.pool.sks-keyservers.net $(shuf -e \
#                            hkp://p80.pool.sks-keyservers.net:80 \
#                            keyserver.ubuntu.com \
#                            hkp://keyserver.ubuntu.com:80 \
#                            pgp.mit.edu) ; do \
#        gpg --batch --keyserver "$server" --recv-keys "$GPG_KEY" && break || : ; \
#    done && \
    gpg --batch --verify flink.tgz.asc flink.tgz; \
    gpgconf --kill all; \
    rm -rf "$GNUPGHOME" flink.tgz.asc; \
  fi && \
  tar -xzf flink.tgz -C $FLINK_HOME --strip-components=1 && \
  rm -rf flink.tgz && \
  chown -R flink:flink . && \
  # Replace default REST/RPC endpoint bind address to use the container's network interface \
  CONF_FILE="$FLINK_HOME/conf/flink-conf.yaml" && \
  if [ ! -e "$FLINK_HOME/conf/flink-conf.yaml" ]; then \
    CONF_FILE="${FLINK_HOME}/conf/config.yaml"; \
    /bin/bash "$FLINK_HOME/bin/config-parser-utils.sh" "${FLINK_HOME}/conf" "${FLINK_HOME}/bin" "${FLINK_HOME}/lib" \
        "-repKV" "rest.address,localhost,0.0.0.0" \
        "-repKV" "rest.bind-address,localhost,0.0.0.0" \
        "-repKV" "jobmanager.bind-host,localhost,0.0.0.0" \
        "-repKV" "taskmanager.bind-host,localhost,0.0.0.0" \
        "-rmKV" "taskmanager.host=localhost"; \
  else \
    sed -i 's/rest.address: localhost/rest.address: 0.0.0.0/g' "$CONF_FILE"; \
    sed -i 's/rest.bind-address: localhost/rest.bind-address: 0.0.0.0/g' "$CONF_FILE"; \
    sed -i 's/jobmanager.bind-host: localhost/jobmanager.bind-host: 0.0.0.0/g' "$CONF_FILE"; \
    sed -i 's/taskmanager.bind-host: localhost/taskmanager.bind-host: 0.0.0.0/g' "$CONF_FILE"; \
    sed -i '/taskmanager.host: localhost/d' "$CONF_FILE"; \
  fi

# Download the Flink Kafka SQL connector fat JAR
RUN wget -P $FLINK_LIB_DIR https://repo1.maven.org/maven2/org/apache/flink/flink-sql-connector-kafka/$FLINK_KAFKA_CONNECTOR_VERSION/flink-sql-connector-kafka-$FLINK_KAFKA_CONNECTOR_VERSION.jar
## Download the Flink Avro Confluent registry JAR
RUN wget -P $FLINK_LIB_DIR https://repo1.maven.org/maven2/org/apache/flink/flink-avro-confluent-registry/$AVRO_CONFLUENT_REGISTRY_VERSION/flink-avro-confluent-registry-$AVRO_CONFLUENT_REGISTRY_VERSION.jar

## Set the classpath for Flink to include the downloaded JARs
ENV CLASSPATH=$FLINK_LIB_DIR/*

## Copy flink-sql-runner
RUN mkdir $FLINK_HOME/usrlib
ADD target/flink-sql-runner-*.jar $FLINK_HOME/usrlib/flink-sql-runner.jar

# Download the script for docker entrypoint from the GitHub repository and make it executable
RUN wget -O docker-entrypoint.sh https://raw.githubusercontent.com/apache/flink-docker/master/$FLINK_VERSION/scala_$SCALA_VERSION-java$JAVA_VERSION-ubuntu/docker-entrypoint.sh
RUN chmod +x docker-entrypoint.sh

# Configure container
ENTRYPOINT ["/docker-entrypoint.sh"]
#EXPOSE 6123 8081
EXPOSE 6123
CMD ["help"]
