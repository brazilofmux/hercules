FROM debian:stable-slim AS builder

RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates git wget gcc g++ make cmake autoconf automake flex gawk m4 zlib1g-dev libbz2-dev libtool libtool-bin libltdl-dev \
 && wget -O regina-rexx-3.9.7.tar.gz https://sourceforge.net/projects/regina-rexx/files/regina-rexx/3.9.7/regina-rexx-3.9.7.tar.gz/download \
 && tar xzf regina-rexx-3.9.7.tar.gz \
 && rm regina-rexx-3.9.7.tar.gz \
 && (cd regina-rexx-3.9.7;./configure;make -j$(nproc);make install) \
 && rm -rf regina-rexx-3.9.7

# Build extpkgs (crypto/decNumber/SoftFloat/telnet) from source.
# Extpkg CMake parses the build dir name as <name><bits>.<type>,
# e.g. crypto64.Release. Must be out-of-source.
# SDL configure looks for libs in lib/<arch> on non-x86; symlink to lib/.
RUN mkdir -p /tmp/extpkgs /tmp/extpkgs-work \
 && for pkg in crypto decNumber SoftFloat telnet; do \
        git clone --depth 1 https://github.com/SDL-Hercules-390/${pkg}.git /tmp/extpkgs-work/${pkg} \
     && mkdir -p /tmp/extpkgs-work/${pkg}64.Release \
     && (cd /tmp/extpkgs-work/${pkg}64.Release \
         && cmake /tmp/extpkgs-work/${pkg} -DINSTALL_PREFIX=/tmp/extpkgs \
         && make -j$(nproc) \
         && make install); \
    done \
 && MACHINE=$(uname -m) \
 && if [ "$MACHINE" != "x86_64" ]; then \
        mkdir -p /tmp/extpkgs/lib/$MACHINE \
     && cp /tmp/extpkgs/lib/*.a /tmp/extpkgs/lib/$MACHINE/; \
    fi

RUN git clone --depth 1 https://github.com/SDL-Hercules-390/hyperion.git sdl \
 && cd sdl \
 && MACHINE=$(uname -m) \
 && if [ "$MACHINE" = "x86_64" ]; then \
        ARCH_FLAGS='CFLAGS=-march=x86-64-v2 CXXFLAGS=-march=x86-64-v2'; \
    else \
        ARCH_FLAGS=''; \
    fi \
 && ./configure --enable-extpkgs=/tmp/extpkgs --enable-regina-rexx --enable-cckd-bzip2 --enable-het-bzip2 --enable-optimization=yes $ARCH_FLAGS \
 && make -j$(nproc) \
 && make install \
 && rm -rf /sdl /tmp/extpkgs /tmp/extpkgs-work \
 && apt-get purge -y git wget gcc g++ make cmake autoconf automake flex m4 \
 && apt-get autoremove -y \
 && rm -rf /var/lib/apt/lists/* \
 && ldconfig

FROM debian:stable-slim

COPY --from=builder /usr/local/lib/ /usr/local/lib/
COPY --from=builder /usr/local/bin /usr/local/bin/
