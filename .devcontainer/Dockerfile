ARG BASE_IMAGE=mcr.microsoft.com/vscode/devcontainers/base:ubuntu-22.04
ARG DC_NAME
ARG DC_VERSION

FROM ${BASE_IMAGE} AS common-env

# Base toolchain packages and linker alternatives (shared by all compilers and versions)
RUN set -ex && \
    apt-get update && \
    apt-get install -y --no-install-recommends \
        curl ca-certificates gpg xz-utils \
        gcc libc6-dev lld \
        gdb && \
    update-alternatives --install "/usr/bin/ld" "ld" "/usr/bin/ld.lld" 20 && \
    update-alternatives --install "/usr/bin/ld" "ld" "/usr/bin/ld.gold" 10 && \
    update-alternatives --install "/usr/bin/ld" "ld" "/usr/bin/ld.bfd" 10 && \
    apt-get clean -y && rm -rf /var/lib/apt/lists/* /tmp/library-scripts

ONBUILD ARG DBASE=/dlang

FROM common-env AS dmd-env
ONBUILD ARG DBASE=${DBASE}
ONBUILD ARG DC_NAME=${DC_NAME}
ONBUILD ARG DC_VERSION=${DC_VERSION}
ONBUILD ARG D_PATH=${DBASE}/dc/linux/bin64
ONBUILD ARG D_LIBRARY_PATH=${DBASE}/dc/linux/lib64
ONBUILD ARG D_LD_LIBRARY_PATH=${DBASE}/dc/linux/lib64
ONBUILD ARG DMD_PATH=${D_PATH}
ONBUILD ENV DMD=dmd
ONBUILD ENV DC=dmd

FROM common-env AS ldc-env
ONBUILD ARG DBASE=${DBASE}
ONBUILD ARG DC_NAME=${DC_NAME}
ONBUILD ARG DC_VERSION=${DC_VERSION}
ONBUILD ARG D_PATH=${DBASE}/dc/bin
ONBUILD ARG D_LIBRARY_PATH=${DBASE}/dc/lib
ONBUILD ARG D_LD_LIBRARY_PATH=${DBASE}/dc/lib
ONBUILD ARG DMD_PATH=${D_PATH}
ONBUILD ENV DMD=ldmd2
ONBUILD ENV DC=ldc2

# libxml2 is a runtime dependency for ldc (version-independent)
ARG EXTRA_PKG=libxml2
RUN apt-get update && \
    apt-get install -y --no-install-recommends ${EXTRA_PKG} && \
    apt-get clean -y && rm -rf /var/lib/apt/lists/*

FROM common-env AS gdc-env
ONBUILD ARG DBASE=${DBASE}
ONBUILD ARG DC_NAME=${DC_NAME}
ONBUILD ARG DC_VERSION=${DC_VERSION}
ONBUILD ARG D_PATH=${DBASE}/dub:${DBASE}/dc/bin
ONBUILD ARG D_LIBRARY_PATH=${DBASE}/dc/lib64
ONBUILD ARG D_LD_LIBRARY_PATH=${DBASE}/dc/lib64
ONBUILD ARG DMD_PATH=${DBASE}/dc/bin
ONBUILD ENV DMD=gdmd
ONBUILD ENV DC=gdc

# perl-modules is a runtime dependency for gdc (version-independent)
ARG EXTRA_PKG=perl-modules
RUN apt-get update && \
    apt-get install -y --no-install-recommends ${EXTRA_PKG} && \
    apt-get clean -y && rm -rf /var/lib/apt/lists/*

FROM ${DC_NAME}-env

ENV DEBIAN_FRONTEND=noninteractive

# D toolchain for the requested version
RUN set -ex && \
    case "${DC_NAME}:${DC_VERSION}" in \
        dmd:latest|ldc:latest|gdc:latest) DC_PKG="${DC_NAME}" ;; \
        dmd:master) DC_PKG="dmd-nightly" ;; \
        *) DC_PKG="${DC_NAME}-${DC_VERSION}" ;; \
    esac && \
    curl -fsS https://dlang.org/install.sh | bash -s "${DC_PKG}" -p "${DBASE}" && \
    DC_DIR="$(find "${DBASE}" -mindepth 1 -maxdepth 1 -type d -name "${DC_NAME}[-.]*" | sort | tail -n1)" && \
    test -n "${DC_DIR}" && \
    ln -s "${DC_DIR}" "${DBASE}/dc" && \
    chmod 755 -R "${DBASE}" && \
    if getent passwd vscode >/dev/null; then chown -R vscode:vscode "${DBASE}"; fi

# Prevents conflicts with system libs
RUN if [ "${DC_NAME}" = "gdc" ]; then \
        rm ${D_LIBRARY_PATH}/libstdc++.so.6*; \
    fi

# Tweaks for code-d settings
RUN if [ "${DC_NAME}" != "dmd" ]; then ln -s ${DMD_PATH}/${DMD} ${DMD_PATH}/dmd; fi
RUN if [ "${DC_NAME}" = "gdc" ]; then \
        ln -s ${DBASE}/dc/lib/gcc/x86_64-unknown-linux-gnu/*/include/d /dlang/dc/import || true; \
    fi

ENV PATH=${D_PATH}:${PATH}
ENV LIBRARY_PATH=${D_LIBRARY_PATH}:${LIBRARY_PATH}
ENV LD_LIBRARY_PATH=${D_LD_LIBRARY_PATH}:${LD_LIBRARY_PATH}
