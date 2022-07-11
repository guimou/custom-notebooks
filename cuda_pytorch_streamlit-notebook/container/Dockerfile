# ARGS found in the different builds
ARG OPENCV_VERSION=4.6.0
ARG CUDA_VERSION=11.6.2-cudnn8-devel-centosstream8
ARG CUDA_ARCH_BIN=6.1

FROM localhost/cuda:${CUDA_VERSION} as base

LABEL name="s2i-cuda-pytorch-streamlit-notebook:latest" \
      summary="Data Science Jupyter Notebook Source-to-Image for Python 3.9 applications." \
      description="Notebook image based on Source-to-Image.These images can be used in OpenDatahub JupterHub." \
      io.k8s.description="Notebook image based on Source-to-Image.These images can be used in OpenDatahub JupterHub." \
      io.k8s.display-name="Generic Data Science with Monai and Streamlit Notebook Python 3.9-ubi8 S2I" \
      io.openshift.expose-services="8888:http" \
      io.openshift.tags="python,python39" \
      authoritative-source-url="https://github.com/guimou/custom-notebooks" \
      io.openshift.s2i.build.commit.ref="main" \
      io.openshift.s2i.build.source-location="https://github.com/guimou/custom-notebooks/cuda-pytorch-streamlit-notebook" \
      io.openshift.s2i.build.image="https://quay.io/guimou/s2i-cuda-pytorch-streamlit-notebook"

############
# s2i core #
############

ENV \
    # DEPRECATED: Use above LABEL instead, because this will be removed in future versions.
    STI_SCRIPTS_URL=image:///usr/libexec/s2i \
    # Path to be used in other layers to place s2i scripts into
    STI_SCRIPTS_PATH=/usr/libexec/s2i \
    APP_ROOT=/opt/app-root \
    # The $HOME is not set by default, but some applications needs this variable
    HOME=/opt/app-root/src \
    PATH=/opt/app-root/src/bin:/opt/app-root/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:${PATH} \
    PLATFORM="el8"

USER root

# Copy just prepare-yum-repositories that is needed for packages install step,
# other files might be added later so changing them does not cause packages
# to be installed again, which takes long time

# This is the list of basic dependencies that all language container image can
# consume.
# Also setup the 'openshift' user that is used for the build execution and for the
# application runtime execution.
# TODO: Use better UID and GID values

RUN INSTALL_PKGS="bsdtar \
  findutils \
  groff-base \
  glibc-locale-source \
  glibc-langpack-en \
  gettext \
  rsync \
  scl-utils \
  tar \
  unzip \
  xz \
  yum" && \
  mkdir -p ${HOME}/.pki/nssdb && \
  chown -R 1001:0 ${HOME}/.pki && \
  yum -y update && \
  yum install -y --setopt=tsflags=nodocs $INSTALL_PKGS && \
  rpm -V $INSTALL_PKGS && \
  yum -y clean all --enablerepo='*' && \
  rm -rf /var/cache/dnf

# Copy extra files to the image.
COPY ./root/ /

# Directory with the sources is set as the working directory so all STI scripts
# can execute relative to this path.
WORKDIR ${HOME}


# Reset permissions of modified directories and add default user
RUN rpm-file-permissions && \
  useradd -u 1001 -r -g 0 -d ${HOME} -s /sbin/nologin \
      -c "Default Application User" default && \
  chown -R 1001:0 ${APP_ROOT}

############
# s2i base #
############

ENV NODEJS_VER=14

# This is the list of basic dependencies that all language container image can
# consume.
RUN yum -y module enable nodejs:$NODEJS_VER && \
  INSTALL_PKGS="autoconf \
  automake \
  bzip2 \
  gcc-c++ \
  gd-devel \
  gdb \
  git \
  libcurl-devel \
  libpq-devel \
  libxml2-devel \
  libxslt-devel \
  lsof \
  make \
  mariadb-connector-c-devel \
  openssl-devel \
  patch \
  procps-ng \
  npm \
  redhat-rpm-config \
  sqlite-devel \
  unzip \
  wget \
  which \
  zlib-devel" && \
  yum install -y --setopt=tsflags=nodocs $INSTALL_PKGS && \
  rpm -V $INSTALL_PKGS && \
  yum -y clean all --enablerepo='*' && \
  rm -rf /var/cache/dnf

############
# s2i py39 #
############

EXPOSE 8080

ENV PYTHON_VERSION=3.9 \
    PATH=$HOME/.local/bin/:$PATH \
    PYTHONUNBUFFERED=1 \
    PYTHONIOENCODING=UTF-8 \
    LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    CNB_STACK_ID=com.redhat.stacks.ubi8-python-39 \
    CNB_USER_ID=1001 \
    CNB_GROUP_ID=0 \
    PIP_NO_CACHE_DIR=off

# RHEL7 base images automatically set these envvars to run scl_enable. RHEl8
# images, however, don't as most images don't need SCLs any more. But we want
# to run it even on RHEL8, because we set the virtualenv environment as part of
# that script
ENV BASH_ENV=${APP_ROOT}/etc/scl_enable \
    ENV=${APP_ROOT}/etc/scl_enable \
    PROMPT_COMMAND=". ${APP_ROOT}/etc/scl_enable"

RUN INSTALL_PKGS="python39 python39-devel python39-setuptools python39-pip nss_wrapper \
        httpd httpd-devel mod_ssl mod_auth_gssapi mod_ldap \
        mod_session atlas-devel gcc-gfortran libffi-devel libtool-ltdl enchant" && \
    yum -y module enable python39:3.9 httpd:2.4 && \
    yum -y --setopt=tsflags=nodocs install $INSTALL_PKGS && \
    rpm -V $INSTALL_PKGS && \
    yum -y clean all --enablerepo='*' && \
    rm -rf /var/cache/dnf

# Copy the S2I scripts from the specific language image to $STI_SCRIPTS_PATH.
COPY 3.9/s2i/bin/ $STI_SCRIPTS_PATH

# Copy extra files to the image.
COPY 3.9/root/ /

# - Create a Python virtual environment for use by any application to avoid
#   potential conflicts with Python packages preinstalled in the main Python
#   installation.
# - In order to drop the root user, we have to make some directories world
#   writable as OpenShift default security model is to run the container
#   under random UID.
RUN \
    python3.9 -m venv ${APP_ROOT} && \
    # Python 3 only code, Python 2 installs pip from PyPI in the assemble script. \
    # We have to upgrade pip to a newer verison because: \
    # * pip < 9 does not support different packages' versions for Python 2/3 \
    # * pip < 19.3 does not support manylinux2014 wheels. Only manylinux2014 (and later) wheels \
    #   support platforms like ppc64le, aarch64 or armv7 \
    # We are newly using wheel from one of the latest stable Fedora releases (from RPM python-pip-wheel) \
    # because it's tested better then whatever version from PyPI and contains useful patches. \
    # We have to do it here (in the macro) so the permissions are correctly fixed and pip is able \
    # to reinstall itself in the next build phases in the assemble script if user wants the latest version \
    ${APP_ROOT}/bin/pip install /opt/wheels/pip-* && \
    rm -r /opt/wheels && \
    chown -R 1001:0 ${APP_ROOT} && \
    fix-permissions ${APP_ROOT} -P && \
    rpm-file-permissions


#############
# s2i thoth #
#############

ENV THAMOS_NO_PROGRESSBAR=1 \
    THAMOS_NO_EMOJI=1 \
    MICROPIPENV_NO_LOCKFILE_PRINT=0 \
    MICROPIPENV_NO_LOCKFILE_WRITE=0

COPY thoth/ .

COPY assemble .

RUN TMPFILE=$(mktemp) && \
    TMPFILE_ASSEMBLE=$(mktemp) && \
    /usr/bin/pip3 install -U "micropipenv" && \
    MICROPIPENV_NO_LOCKFILE_WRITE=1 MICROPIPENV_PIP_BIN=/usr/bin/pip3 micropipenv install && \
    cp assemble "${TMPFILE_ASSEMBLE}" && \
    cp "${STI_SCRIPTS_PATH}/assemble" "${TMPFILE}" && \
    head -n1 "${TMPFILE}" >"${STI_SCRIPTS_PATH}/assemble" && \
    cat "${TMPFILE_ASSEMBLE}" >>"${STI_SCRIPTS_PATH}/assemble" && \
    tail -n+2 "${TMPFILE}" >>"${STI_SCRIPTS_PATH}/assemble" && \
    rm "${TMPFILE}" "${TMPFILE_ASSEMBLE}" Pipfile Pipfile.lock && \
    sed -i '/  echo "---> Running application from .*/d' "${STI_SCRIPTS_PATH}/run" && \
    chown -R 1001:0 ${APP_ROOT} && \
    fix-permissions ${APP_ROOT} -P

####################
# Minimal notebook #
####################

ENV JUPYTER_ENABLE_LAB="true" \
    ENABLE_MICROPIPENV="1" \
    UPGRADE_PIP_TO_LATEST="1" \
    WEB_CONCURRENCY="1" \
    THOTH_ADVISE="0" \
    THOTH_ERROR_FALLBACK="1" \
    THOTH_DRY_RUN="1" \
    THAMOS_DEBUG="0" \
    THAMOS_VERBOSE="1" \
    THOTH_PROVENANCE_CHECK="0"

# Copying in override assemble/run scripts
COPY minimal/.s2i/bin /tmp/scripts
# Copying in source code
COPY minimal/. /tmp/src
# Change file ownership to the assemble user. Builder image must support chown command.
RUN chown -R 1001:0 /tmp/scripts /tmp/src

USER 1001
RUN /tmp/scripts/assemble


#######################
# Pytorch + Streamlit #
#######################

USER root

RUN yum install -y yum-utils && \
    yum-config-manager --enable powertools && \
    yum install -y https://download.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm && \
    yum install -y https://download1.rpmfusion.org/free/el/rpmfusion-free-release-8.noarch.rpm && \
    yum install -y https://download1.rpmfusion.org/nonfree/el/rpmfusion-nonfree-release-8.noarch.rpm && \
    INSTALL_PKGS="libsndfile \
    ffmpeg \
    ffmpeg-devel" && \
    yum install -y --setopt=tsflags=nodocs $INSTALL_PKGS && \
    rpm -V $INSTALL_PKGS && \
    yum -y clean all --enablerepo='*' && \
    rm -rf /var/cache/dnf

WORKDIR /tmp/

# Copying custom packages
COPY Pipfile Pipfile.lock /tmp/
RUN micropipenv install --deploy && \
    rm -rf /opt/app-root/lib/python3.9/site-packages/~*

# Streamlit extenstion installation
COPY streamlit-launcher.sh /opt/app-root/bin/
COPY jupyterlab_streamlit_menu-0.1.0-py3-none-any.whl /tmp/

# Install packages and cleanup upgrades
RUN pip install --no-cache-dir /tmp/jupyterlab_streamlit_menu-0.1.0-py3-none-any.whl

RUN jupyter lab build && \
    jupyter lab clean && \
    npm install -g yarn && \
    npm cache clean --force && \
    yarn cache clean && \
    fix-permissions /opt/app-root

WORKDIR /opt/app-root/src
USER 1001

CMD /opt/app-root/bin/start-singleuser.sh --ip=0.0.0.0 --port=8080


## Build OpenCV and contrib from source
FROM base as BUILDER
ARG OPENCV_VERSION=4.6.0
ARG CUDA_ARCH_BIN=6.1

USER 0

# Install needed packages to build
RUN INSTALL_PKGS="epel-release git gcc gcc-c++ cmake qt5-qtbase-devel \
    gtk2-devel libpng-devel jasper-devel OpenEXR-devel libwebp-devel \
    libjpeg-turbo-devel libtiff-devel tbb-devel libv4l-devel \
    eigen3-devel freeglut-devel mesa-libGL mesa-libGL-devel \
    boost boost-thread boost-devel gstreamer1-plugins-base \
    gstreamer1-plugins-base-devel hdf5 openblas openblas-devel lapack" && \
    yum install -y --setopt=tsflags=nodocs $INSTALL_PKGS && \
    rpm -V $INSTALL_PKGS && \
    yum -y clean all --enablerepo='*' && \
    rm -rf /var/cache/dnf

RUN mkdir -p /tmp/opencv_build && cd /tmp/opencv_build && \
    git clone --depth 1 --branch ${OPENCV_VERSION} https://github.com/opencv/opencv.git && \
    git clone --depth 1 --branch ${OPENCV_VERSION} https://github.com/opencv/opencv_contrib.git && \
    mkdir -p /tmp/opencv_build/opencv/build

WORKDIR /tmp/opencv_build/opencv/build

RUN cmake3 -D CMAKE_BUILD_TYPE=RELEASE \
    -D CMAKE_INSTALL_PREFIX=/opt/opencv \
    -D HAVE_opencv_python3=ON \
    -D INSTALL_C_EXAMPLES=OFF \
    -D INSTALL_PYTHON_EXAMPLES=OFF \
    -D OPENCV_GENERATE_PKGCONFIG=ON \
    -D WITH_CUDA=ON \
    -D WITH_CUDNN=ON \
    -D WITH_CUBLAS=ON \
    -D WITH_TBB=ON \
    -D OPENCV_DNN_CUDA=ON \
    -D OPENCV_ENABLE_NONFREE=ON \
    -D CUDA_ARCH_BIN=6.1 \
    -D OPENCV_EXTRA_MODULES_PATH=/tmp/opencv_build/opencv_contrib/modules \
    -D BUILD_EXAMPLES=OFF ..

#RUN make -j`nproc --ignore=2`

