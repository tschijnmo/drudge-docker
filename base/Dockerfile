#
# The basic base images for drudge development and execution
# ==========================================================
#

# Base image for C++ library developments.
#
# Starting with GCC, which is generally the hardest to get up-to-date, tools
# like cmake and gdb are installed.  For convenience of development, utilities
# like vim/tmux are also included, and the apt cache lists are not cleared.
#
# This stage is designed to be used in the development of core C++ libraries
# like libcanon or libparenth.
FROM gcc:9.2.0 AS corebase

RUN set -ex; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
	    cmake \
	    gdb \
	    man \
	    tmux \
	    vim \
	    ;

COPY gdbinit /etc/gdb/gdbinit

WORKDIR /home/src
CMD bash


# Image for actual drudge development and execution.
#
# In addition to things in the core base, here the cutting edge Python and
# Spark are installed, as well as some other small Python runtime dependencies.
FROM corebase AS base

# Python
#
# The entire code is adapted from the official Python docker file.
RUN set -ex; \
	apt-get install -y --no-install-recommends \
	    tk-dev \
	    uuid-dev \
	    ;

# ensure local python is preferred over distribution python
ENV PATH /usr/local/bin:$PATH

# http://bugs.python.org/issue19846
# > At the moment, setting "LANG=C" on a Linux system *fundamentally breaks Python 3*, and that's not OK.
ENV LANG C.UTF-8

ENV GPG_KEY 0D96DF4D4110E5C43FBFB17F2D347EA6AA65421D
ENV PYTHON_VERSION 3.7.4

RUN set -ex \
	\
	&& wget -O python.tar.xz "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz" \
	&& wget -O python.tar.xz.asc "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz.asc" \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys "$GPG_KEY" \
	&& gpg --batch --verify python.tar.xz.asc python.tar.xz \
	&& { command -v gpgconf > /dev/null && gpgconf --kill all || :; } \
	&& rm -rf "$GNUPGHOME" python.tar.xz.asc \
	&& mkdir -p /usr/src/python \
	&& tar -xJC /usr/src/python --strip-components=1 -f python.tar.xz \
	&& rm python.tar.xz \
	\
	&& cd /usr/src/python \
	&& gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
	&& ./configure \
		--build="$gnuArch" \
		--enable-loadable-sqlite-extensions \
		--enable-optimizations \
		--enable-shared \
		--with-system-expat \
		--with-system-ffi \
		--without-ensurepip \
	&& make -j "$(nproc)" \
# setting PROFILE_TASK makes "--enable-optimizations" reasonable: https://bugs.python.org/issue36044 / https://github.com/docker-library/python/issues/160#issuecomment-509426916
		PROFILE_TASK='-m test.regrtest --pgo \
			test_array \
			test_base64 \
			test_binascii \
			test_binhex \
			test_binop \
			test_bytes \
			test_c_locale_coercion \
			test_class \
			test_cmath \
			test_codecs \
			test_compile \
			test_complex \
			test_csv \
			test_decimal \
			test_dict \
			test_float \
			test_fstring \
			test_hashlib \
			test_io \
			test_iter \
			test_json \
			test_long \
			test_math \
			test_memoryview \
			test_pickle \
			test_re \
			test_set \
			test_slice \
			test_struct \
			test_threading \
			test_time \
			test_traceback \
			test_unicode \
		' \
	&& make install \
	&& ldconfig \
	\
	&& find /usr/local -depth \
		\( \
			\( -type d -a \( -name test -o -name tests \) \) \
			-o \
			\( -type f -a \( -name '*.pyc' -o -name '*.pyo' \) \) \
		\) -exec rm -rf '{}' + \
	&& rm -rf /usr/src/python \
	\
	&& python3 --version

# make some useful symlinks that are expected to exist
RUN cd /usr/local/bin \
	&& ln -s idle3 idle \
	&& ln -s pydoc3 pydoc \
	&& ln -s python3 python \
	&& ln -s python3-config python-config

# if this is called "PIP_VERSION", pip explodes with "ValueError: invalid truth value '<VERSION>'"
ENV PYTHON_PIP_VERSION 19.2.3
# https://github.com/pypa/get-pip
ENV PYTHON_GET_PIP_URL https://github.com/pypa/get-pip/raw/309a56c5fd94bd1134053a541cb4657a4e47e09d/get-pip.py
ENV PYTHON_GET_PIP_SHA256 57e3643ff19f018f8a00dfaa6b7e4620e3c1a7a2171fd218425366ec006b3bfe

RUN set -ex; \
	\
	wget -O get-pip.py "$PYTHON_GET_PIP_URL"; \
	echo "$PYTHON_GET_PIP_SHA256 *get-pip.py" | sha256sum --check --strict -; \
	\
	python get-pip.py \
		--disable-pip-version-check \
		--no-cache-dir \
		"pip==$PYTHON_PIP_VERSION" \
	; \
	pip --version; \
	\
	find /usr/local -depth \
		\( \
			\( -type d -a \( -name test -o -name tests \) \) \
			-o \
			\( -type f -a \( -name '*.pyc' -o -name '*.pyo' \) \) \
		\) -exec rm -rf '{}' +; \
	rm -f get-pip.py

# Some drudge-specific Python dependencies.
RUN set -ex \
	&& pip3 install Jinja2 sympy numpy networkx ipython pytest

RUN set -ex \
	&& git clone https://github.com/tschijnmo/DummyRDD.git \
	&& cd DummyRDD \
	&& python3 setup.py install \
	&& cd .. && rm -rf DummyRDD


# Spark
#
# Sid is first added for Java 8, which is required for Spark.
RUN set -ex; \
	echo "deb http://ftp.us.debian.org/debian sid main" >> /etc/apt/sources.list; \
	apt-get update; \
	apt-get install -y --no-install-recommends  openjdk-8-jre;

# Install Spark runtime.
RUN set -ex \
    && wget -O spark.tar.gz "https://www-us.apache.org/dist/spark/spark-2.4.4/spark-2.4.4-bin-hadoop2.7.tgz" \
    && mkdir -p /usr/local/spark \
    && tar -xzC /usr/local/spark --strip-components=1 -f spark.tar.gz \
    && rm spark.tar.gz

# Set the environment variables up.
ENV JAVA_HOME /usr/lib/jvm/java-8-openjdk-amd64/jre
ENV SPARK_HOME /usr/local/spark
ENV PYSPARK_PYTHON /usr/local/bin/python3
ENV PYSPARK_DRIVER_PYTHON ${PYSPARK_PYTHON}
ENV PYTHONPATH ${PYTHONPATH}:${SPARK_HOME}/python:${SPARK_HOME}/python/lib/py4j-0.10.4-src.zip
ENV SPARK_LOCAL_IP 127.0.0.1
