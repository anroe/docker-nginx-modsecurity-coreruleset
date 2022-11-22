ARG NGINX_VERSION=1.22.1

FROM nginx:$NGINX_VERSION

ARG NGINX_VERSION=1.22.1
ARG MODSECURITY_VERSION=v3.0.8
ARG MODSECURITY_CONNECTOR_VERSION=v1.0.3
ARG CRS_VERSION=3.3.4

RUN apt-get update \
    && apt-get install -y \
        apt-utils \
        autoconf \
        automake \
        build-essential \
        libcurl4-gnutls-dev \
        libgeoip-dev \
        libpcre++-dev \
        libtool \
        libxml2-dev \
        liblmdb0 \
        libyajl-dev \
        pkgconf \
        zlib1g-dev \
        wget \
        git

RUN cd /opt \
    && git clone --depth 1 -b $MODSECURITY_VERSION --single-branch https://github.com/SpiderLabs/ModSecurity \
    && cd ModSecurity \
    && git submodule update --init \
    && ./build.sh \
    && ./configure \
    && make \
    && make install 

RUN cd /opt \
    && git clone --depth 1 -b $MODSECURITY_CONNECTOR_VERSION --single-branch https://github.com/SpiderLabs/ModSecurity-nginx.git \
    && wget -O - http://nginx.org/download/nginx-$NGINX_VERSION.tar.gz | tar zxfv - \
    && mv /opt/nginx-$NGINX_VERSION /opt/nginx \
    && cd /opt/nginx \
    && ./configure --with-compat --add-dynamic-module=/opt/ModSecurity-nginx \
    && make modules 

RUN cd /opt \
    && wget -O - https://github.com/coreruleset/coreruleset/archive/v$CRS_VERSION.tar.gz | tar zxfv - \
    && ls -la /opt \
    && mv /opt/coreruleset-$CRS_VERSION /opt/coreruleset

FROM nginx:$NGINX_VERSION

COPY --from=0 /opt/nginx/objs/ngx_http_modsecurity_module.so /usr/lib/nginx/modules
COPY --from=0 /opt/ModSecurity/modsecurity.conf-recommended /etc/nginx/modsecurity/modsecurity.conf
COPY --from=0 /opt/coreruleset/crs-setup.conf.example /etc/nginx/modsecurity/crs-setup.conf
COPY --from=0 /opt/coreruleset/rules/ /etc/nginx/modsecurity/rules/
COPY --from=0 /usr/local/modsecurity/ /usr/local/modsecurity/
#https://github.com/SpiderLabs/ModSecurity/issues/1941
COPY --from=0 /opt/ModSecurity/unicode.mapping /etc/nginx/modsecurity/unicode.mapping

RUN apt-get update \
    && apt-get install --no-install-recommends --no-install-suggests -y \
        libyajl2 \
        liblmdb0 \
        libcurl4-gnutls-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && chmod -R 644 \
        /usr/lib/nginx/modules/ngx_http_modsecurity_module.so \
        /etc/nginx/modsecurity/crs-setup.conf \
        /etc/nginx/modsecurity/rules/* \
    && chmod 755 \
        /etc/nginx/modsecurity/ \
        /etc/nginx/modsecurity/rules \
    && mv /etc/nginx/modsecurity/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf.example /etc/nginx/modsecurity/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf \
    && mv /etc/nginx/modsecurity/rules/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf.example /etc/nginx/modsecurity/rules/RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf \
    && sed -i '1iload_module \/usr\/lib\/nginx\/modules\/ngx_http_modsecurity_module.so;' /etc/nginx/nginx.conf \
    && sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' /etc/nginx/modsecurity/modsecurity.conf \
    && sed -i 's/SecAuditEngine RelevantOnly/SecAuditEngine Off/' /etc/nginx/modsecurity/modsecurity.conf \
    && echo 'include "/etc/nginx/modsecurity/modsecurity.conf"' >> /etc/nginx/modsecurity/main.conf \
    && echo 'include "/etc/nginx/modsecurity/crs-setup.conf"' >> /etc/nginx/modsecurity/main.conf \
    && echo 'include "/etc/nginx/modsecurity/rules/*.conf"' >> /etc/nginx/modsecurity/main.conf \
    && sed -i '/^http {/a \ \ \ \ modsecurity on;\n\ \ \ \ modsecurity_rules_file /etc/nginx/modsecurity/main.conf;\n' /etc/nginx/nginx.conf