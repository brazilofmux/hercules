FROM debian:stable-slim as builder

RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates git wget gcc g++ make autoconf automake flex gawk m4 zlib1g-dev libbz2-dev \
 && wget -O regina-rexx-3.9.3.tar.gz https://sourceforge.net/projects/regina-rexx/files/regina-rexx/3.9.3/regina-rexx-3.9.3.tar.gz/download \
 && tar xzf regina-rexx-3.9.3.tar.gz \
 && rm regina-rexx-3.9.3.tar.gz \
 && (cd regina-rexx-3.9.3;./configure;make -j12;make install) \
 && rm -rf regina-rexx-3.9.3 \
 && git clone --depth 1 https://github.com/SDL-Hercules-390/hyperion.git sdl \
 && (cd sdl;./configure --enable-regina-rexx --enable-cckd-bzip2 --enable-het-bzip2 --enable-optimization=yes;make -j12;make install) \
 && rm -rf sdl \
 && apt-get purge -y git wget gcc g++ make autoconf automake flex m4 \
 && apt-get autoremove -y \
 && rm -rf /var/lib/apt/lists/* \
 && ldconfig

FROM debian:stable-slim

COPY --from=builder /usr/local/lib/ /usr/local/lib/
COPY --from=builder /usr/local/bin /usr/local/bin/
