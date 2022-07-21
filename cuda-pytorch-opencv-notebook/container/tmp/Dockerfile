ARG BASE_IMAGE
FROM ${BASE_IMAGE} as BUILDER

ARG OPENCV_VERSION=4.6.0
ARG CUDA_ARCH_BIN=6.0,6.1,7.0,7.5,8.0,8.6

USER 0

##########################################
# Install various dependencies for build #
##########################################

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

# Install needed packages to build
RUN INSTALL_PKGS="epel-release git gcc gcc-c++ cmake qt5-qtbase-devel \
    gtk2-devel libpng-devel jasper-devel OpenEXR-devel libwebp-devel \
    libjpeg-turbo-devel libtiff-devel tbb-devel libv4l-devel \
    eigen3-devel freeglut-devel mesa-libGL mesa-libGL-devel \
    boost boost-thread boost-devel gstreamer1-plugins-base \
    gstreamer1-plugins-base-devel hdf5  hdf5-devel openblas openblas-devel \
    lapack lapack-devel lmdb-devel" && \
    yum install -y --setopt=tsflags=nodocs $INSTALL_PKGS && \
    rpm -V $INSTALL_PKGS && \
    yum -y clean all --enablerepo='*' && \
    rm -rf /var/cache/dnf

#########################################
## Build OpenCV and Contrib from source #
#########################################

# Build OpenCV
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
    -D OPENCV_ENABLE_NONFREE=OFF \
    -D CUDA_ARCH_BIN=${CUDA_ARCH_BIN} \
    -D OPENCV_EXTRA_MODULES_PATH=/tmp/opencv_build/opencv_contrib/modules \
    -D BUILD_EXAMPLES=OFF ..

RUN make -j`nproc --ignore=2`

RUN make install

##############################
## Build PyTorch from source #
##############################

# PyTorch Dependencies

WORKDIR /tmp

COPY make.inc link.sh /tmp/

# Install Intel MKL and fix symlinks in packages
#  yum install intel-oneapi-mkl-2022.1.0  intel-oneapi-mkl-common-2022.1.0 intel-oneapi-mkl-common-devel-2022.1.0 intel-oneapi-mkl-devel intel-oneapi-mkl-devel-2022.1.0
RUN pip install astunparse ninja future mkl mkl-include mkl-devel && \
    cp /tmp/link.sh /opt/app-root/lib/ && cd /opt/app-root/lib/ && \
    /opt/app-root/lib/link.sh

# Compile Magma
RUN mkdir -p magma_build && cd magma_build && \
    wget http://icl.utk.edu/projectsfiles/magma/downloads/magma-2.6.2.tar.gz && \
    tar -xvzf magma-2.6.2.tar.gz && cd /tmp/magma_build/magma-2.6.2 && \
    cp /tmp/make.inc /tmp/magma_build/magma-2.6.2 && \
    # Don't compile Fortran modules
    sed -i -e 's/FORT\ *?=\ gfortran/#FORT/' Makefile && \
    make lib && \
    make sparse-lib && \
    # Remove copy of Fortan modules
    sed -i -e '/cp\ include\/\*\.mod/d' Makefile && \
    make install prefix=/opt/magma && \
    ln -s /opt/magma/lib/pkgconfig/magma.pc /usr/share/pkgconfig/ && \
    ldconfig

# Install PyTorch from source
ENV USE_MKL=1 \
    USE_MKLDNN=1 \
    MKL_ROOT=/opt/app-root \
    MKL_INCLUDE_DIR=/opt/app-root/include \
    MKL_LIBRARIES=/opt/app-root/lib \
    CMAKE_INCLUDE_PATH=$MKL_INCLUDE_DIR:$CMAKE_INCLUDE_PATH \
    CMAKE_LIBRARY_PATH=$MKL_LIBRARIES:$CMAKE_LIBRARY_PATH \
    BLAS=MKL \
    USE_TBB=1 \
    MKL_TBB=1 \
    MKL_THREADING=TBB \
    MKLDNN_THREADING=TBB \
    USE_CUDA=1 \
    TORCH_CUDA_ARCH_LIST="6.0;6.1;7.0;7.5;8.0;8.6" \
    USE_OPENCV=1 \
    USE_FFMPEG=1 \
    USE_LMDB=1 \
    LD_PRELOAD=/opt/app-root/lib/libiomp5.so \
    BUILD_CAFFE2=1 \
    USE_NUMA=OFF \
    USE_MPI=OFF

RUN mkdir -p /tmp/pytorch_build && cd /tmp/pytorch_build && \
    git clone --recurse-submodules -j 8 --depth 1 --branch v1.12.0 https://github.com/pytorch/pytorch && \
    cd /tmp/pytorch_build/pytorch && \
    # Git repo has pre-release numbering, which prevents pip list to get the right version
    echo "1.12.0" > version.txt && \
    # Remove git directory to not pollute version numbers
    rm -rf .git && \
    MAXJOBS=`nproc --ignore=2` python setup.py install --prefix=/opt/app-root/torch && \
    ln -s /opt/app-root/torch/lib64/python3.9/site-packages/torch /opt/app-root/lib/python3.9/site-packages/torch && \
    ln -s /opt/app-root/torch/lib64/python3.9/site-packages/caffe2 /opt/app-root/lib/python3.9/site-packages/caffe2 && \
    ln -s /opt/app-root/torch/lib64/python3.9/site-packages/torch-1.12.0-py3.9.egg-info /opt/app-root/lib/python3.9/site-packages/torch-1.12.0-py3.9.egg-info && \
    ln -s /opt/app-root/torch/lib64/python3.9/site-packages/torchgen /opt/app-root/lib/python3.9/site-packages/torchgen

# Install Torchvision from source

ENV FORCE_CUDA=1

RUN mkdir -p /tmp/torchvision_build && cd /tmp/torchvision_build && \
    git clone --recurse-submodules -j 8 --depth 1 --branch v0.13.0 https://github.com/pytorch/vision && \
    cd /tmp/torchvision_build/vision && \
    # Git repo has pre-release numbering, which prevents pip list to get the right version
    echo "0.13.0" > version.txt && \
    # Remove git directory to not pollute version numbers
    rm -rf .git && \
    MAXJOBS=`nproc --ignore=2` python setup.py install --prefix=/opt/app-root/torchvision && \
    ln -s /opt/app-root/torchvision/lib64/python3.9/site-packages/torchvision-0.13.0-py3.9-linux-x86_64.egg/torchvision /opt/app-root/lib/python3.9/site-packages/torchvision && \
    ln -s /opt/app-root/torchvision/lib64/python3.9/site-packages/torchvision-0.13.0-py3.9-linux-x86_64.egg/EGG-INFO /opt/app-root/lib/python3.9/site-packages/torchvision-0.13.0-py3.9.egg-info

# TorchAudio

ENV BUILD_SOX=ON \
    BUILD_KALDI=ON \
    BUILD_RNNT=ON \
    BUILD_CTC_DECODER=ON \
    BUILD_TORCHAUDIO_PYTHON_EXTENSION=ON \
    USE_FFMPEG=ON \
    USE_CUDA=ON \
    USE_ROCM=OFF \
    USE_OPENMP=OFF

RUN mkdir -p /tmp/torchaudio_build && cd /tmp/torchaudio_build && \
    git clone --recurse-submodules -j 8 --depth 1 --branch v0.12.0 https://github.com/pytorch/audio.git && \
    cd /tmp/torchaudio_build/audio && \
    # Git repo has pre-release numbering, which prevents pip list to get the right version
    echo "0.12.0" > version.txt && \
    MAXJOBS=`nproc --ignore=2` python setup.py install --prefix=/opt/app-root/torchaudio && \

# TorchText

