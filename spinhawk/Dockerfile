FROM debian:stable-slim AS builder

RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates git wget gcc g++ make autoconf automake gawk m4 zlib1g-dev libbz2-dev \
 && wget -O regina-rexx-3.9.4.tar.gz https://sourceforge.net/projects/regina-rexx/files/regina-rexx/3.9.4/regina-rexx-3.9.4.tar.gz/download \
 && tar xzf regina-rexx-3.9.4.tar.gz \
 && rm regina-rexx-3.9.4.tar.gz \
 && (cd regina-rexx-3.9.4;./configure;make -j12;make install) \
 && rm -rf regina-rexx-3.9.4 \
 && git clone --depth 1 https://github.com/rbowler/spinhawk.git spinhawk \
 && (cd spinhawk;chmod u+x autogen.sh;./autogen.sh;./configure --enable-ltdl-install --enable-cckd-bzip2 --enable-het-bzip2 --enable-optimization=yes --enable-multi-cpu=yes --enable-interlocked-access-facility-2=yes;make -j12;make check;make install) \
 && rm -rf spinhawk \
 && apt-get purge -y git wget gcc g++ make autoconf automake gawk m4 zlib1g-dev libbz2-dev \
 && apt-get autoremove -y \
 && rm -rf /var/lib/apt/lists/* \
 && ldconfig

FROM debian:stable-slim

COPY --from=builder /usr/local/lib/ /usr/local/lib/
COPY --from=builder /usr/local/bin /usr/local/bin/
