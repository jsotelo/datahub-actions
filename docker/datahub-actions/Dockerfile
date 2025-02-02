# Copyright 2021 Acryl Data, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Defining environment
ARG APP_ENV=prod

FROM python:3.9.9 as base
ENV DOCKERIZE_VERSION v0.6.1
RUN apt-get update \
&& if [ $(arch) = "aarch64" ]; then \
       DOCKERIZE_ARCH='aarch64';\
        elif [ $(arch) = "x86_64" ]; then \
       DOCKERIZE_ARCH='amd64'; \
        else \
        echo >&2 "Unsupported architecture $(arch)" ; exit 1; \
        fi \
        && apt-get install -y \
        jq \
        librdkafka-dev \
        python3-ldap \
        libldap2-dev \
        libsasl2-dev \
        libsasl2-modules \
        libaio1 \
        wget \
        zip \
        unzip \
        ldap-utils \
    && curl -L https://github.com/treff7es/dockerize/releases/download/$DOCKERIZE_VERSION/dockerize-linux-${DOCKERIZE_ARCH}-$DOCKERIZE_VERSION.tar.gz | tar -C /usr/local/bin -xzv \
    && python -m pip install --upgrade pip wheel setuptools==57.5.0

RUN if [ $(arch) = "x86_64" ]; then \
        mkdir /opt/oracle && \
        cd /opt/oracle && \
        wget --no-verbose -c https://download.oracle.com/otn_software/linux/instantclient/216000/instantclient-basic-linux.x64-21.6.0.0.0dbru.zip && \
        unzip instantclient-basic-linux.x64-21.6.0.0.0dbru.zip && \
        rm instantclient-basic-linux.x64-21.6.0.0.0dbru.zip && \
        sh -c "echo /opt/oracle/instantclient_21_6 > /etc/ld.so.conf.d/oracle-instantclient.conf" && \
        ldconfig; \
    else \
        mkdir /opt/oracle && \
        cd /opt/oracle && \
        wget --no-verbose -c https://download.oracle.com/otn_software/linux/instantclient/191000/instantclient-basic-linux.arm64-19.10.0.0.0dbru.zip && \
        unzip instantclient-basic-linux.arm64-19.10.0.0.0dbru.zip && \
        rm instantclient-basic-linux.arm64-19.10.0.0.0dbru.zip && \
        sh -c "echo /opt/oracle/instantclient_19_10 > /etc/ld.so.conf.d/oracle-instantclient.conf" && \
        ldconfig; \
    fi;

FROM base as prod-install
COPY datahub-actions /actions-src
RUN mkdir -p /etc/datahub/actions && mkdir -p /tmp/datahub/logs/actions/system
RUN cd /actions-src && \
    pip install "." && \
    pip install '.[executor]'

COPY ./docker/datahub-actions/start.sh /start_datahub_actions.sh
RUN chmod a+x /start_datahub_actions.sh

# Add other default configurations into this!
RUN mkdir -p /etc/datahub/actions/conf && mkdir -p /etc/datahub/actions/system/conf
COPY ./docker/config/executor.yaml /etc/datahub/actions/system/conf

RUN addgroup --system datahub && adduser --system datahub --ingroup datahub \
    && chown datahub /etc/datahub \
    && chown -R datahub /tmp/datahub

# By transferring the root user's pip cache directory to the datahub
# user, we can avoid the need for some redundant dependency downloads.
RUN mkdir -p /home/datahub/.cache \
    && mv /root/.cache/pip /home/datahub/.cache/pip \
    && chown -R datahub /home/datahub/.cache/pip



FROM ${APP_ENV}-install as final
USER datahub
RUN curl -s "https://get.sdkman.io" | bash
RUN /bin/bash -c "source /$HOME/.sdkman/bin/sdkman-init.sh; sdk version; sdk install java 8.0.332-zulu"
CMD dockerize -wait http://$DATAHUB_GMS_HOST:$DATAHUB_GMS_PORT/health -timeout 240s /start_datahub_actions.sh
