
#FROM forgerock/downloader 
FROM gcr.io/engineering-devops/downloader as downloader


ARG VERSION="6.5.0"
RUN download -v $VERSION opendj

FROM openjdk:11-jdk as dsbuilder
COPY --from=downloader /opendj.zip /var/tmp/opendj.zip

WORKDIR /var/tmp/bootstrap

COPY bootstrap/ /var/tmp/bootstrap/

# This speeds up docker cloudbuild considerably
ENV OPENDJ_JAVA_ARGS "-Djava.security.egd=file:/dev/./urandom"

RUN ./setup.sh

FROM openjdk:11-jdk as jlink

RUN jlink --compress=2 \
   --add-modules java.base,java.compiler,java.desktop,java.instrument,java.management.rmi,java.prefs,java.scripting,java.security.jgss,java.security.sasl,java.sql,jdk.security.auth,jdk.unsupported \
   --output /var/tmp/jdk \
   && apt-get update && apt-get install -y binutils \ 
   && strip -p --strip-unneeded /var/tmp/jdk/lib/server/libjvm.so


#####################  Runtime container ################
FROM debian:sid-slim


RUN useradd -d /opt/opendj -G root -m -u 11111 forgerock \
    && apt-get update \
    && apt-get install -y --no-install-recommends vim \
    && rm -rf /var/lib/apt/lists/* 

COPY --from=dsbuilder --chown=forgerock:root /var/tmp/bootstrap/run/dsrs1 /opt/opendj/
COPY --from=jlink --chown=forgerock:root  /var/tmp/jdk /opt/opendj/jdk/

   
WORKDIR /opt/opendj 


# This is the default location where secrets should be mounted.
ENV SECRET_PATH /var/run/secrets/opendj
# Path to secret file that contains the cn=Directory Manager password
# Used to configure the image
ENV DIR_MANAGER_PW_FILE /var/run/secrets/opendj/dirmanager.pw

# Path to the file that contains the uid=monitor user password for Prometheus metrics.
ENV MONITOR_PW_FILE /var/run/secrets/opendj/monitor.pw
# Paths to the keystore and the keystore pin.
ENV KEYSTORE_FILE /var/run/secrets/opendj/keystore.pkcs12
ENV KEYSTORE_PIN_FILE /var/run/secrets/opendj/keystore.pin
# Default Java arguments - may be overridden when deployed
ENV OPENDJ_JAVA_ARGS "-Xmx1024m"
ENV OPENDJ_JAVA_HOME "/opt/opendj/jdk" 

COPY --chown=forgerock:root *.sh /opt/opendj/
COPY --chown=forgerock:root bootstrap/secrets/* /var/run/secrets/opendj/
COPY --chown=forgerock:root scripts/ /opt/opendj/scripts/ 

#RUN chown -R forgerock:root /opt/opendj

USER 11111

ENV PATH $PATH:/opt/opendj/scripts:/opt/opendj/bin


ENTRYPOINT  ["/opt/opendj/docker-entrypoint.sh"]

CMD ["start"]
