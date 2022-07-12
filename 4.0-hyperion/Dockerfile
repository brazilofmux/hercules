FROM debian:stable-slim as builder

WORKDIR /opt
COPY html.patch .

RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates wget cmake git gcc libc6-dev sed make patch \
 && rm -rf /var/lib/apt/lists/* \
 && wget -O regina-rexx-3.9.4.tar.gz https://sourceforge.net/projects/regina-rexx/files/regina-rexx/3.9.4/regina-rexx-3.9.4.tar.gz/download \
 && tar xzf regina-rexx-3.9.4.tar.gz \
 && rm regina-rexx-3.9.4.tar.gz \
 && (cd regina-rexx-3.9.4;./configure;make -j12;make install) \
 && rm -rf regina-rexx-3.9.4 \
 && git clone --depth 1 https://github.com/hercules-390/hyperion.git hyperion \
 && (cd hyperion;patch -p0 < ../html.patch;mkdir build;cd build;cmake ..;cmake --build .;ctest;cmake -P cmake_install.cmake) \
 && rm -rf hyperion

FROM debian:stable-slim

COPY --from=builder /usr/local/lib/ /usr/local/lib/
COPY --from=builder /usr/local/bin /usr/local/bin/
