#
# This file is for creating the build environment docker image
# for frugen.
#
ARG ostype=ubuntu
ARG osver=noble
FROM $ostype:$osver
ARG ostype
ARG osver
ENV osver=$osver
ENV ostype=$ostype
ENV DEBIAN_FRONTEND=noninteractive
RUN case "$ostype" in \
      fedora) \
        echo Building for Fedora $osver; \
        yum install -y \
          make \
          automake \
          gcc \
          glibc-static \
          cmake \
          json-c-devel \
          doxygen \
          graphviz \
          git && \
        echo "Preparing a static version of json-c for Fedora" ;\
        JSONC_VER=$(pkgconf --modversion json-c)&& \
        cd / && \
        git clone -b json-c-${JSONC_VER} https://github.com/json-c/json-c.git && \
        cd /json-c && \
        mkdir build && \
        cd build && \
        cmake .. && \
        make && \
        make install \
        ;; \
      ubuntu) \
        echo Building for Ubuntu $osver; \
        apt-get update && \
        apt-get -y install \
          git \
          build-essential \
          mingw-w64 \
          pkg-config \
          cmake \
          libjson-c-dev \
          doxygen \
          graphviz \
        ;; \
      *) \
        echo "Aborted: OS Name is NOT defined!"; \
        exit 1 \
        ;; \
    esac
