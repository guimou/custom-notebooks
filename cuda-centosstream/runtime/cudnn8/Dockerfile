ARG IMAGE_NAME
FROM ${IMAGE_NAME}:11.6.2-runtime-centosstream8

ENV NV_CUDNN_VERSION 8.4.1.50-1
ENV NV_CUDNN_PACKAGE libcudnn8-${NV_CUDNN_VERSION}.cuda11.6

LABEL com.nvidia.cudnn.version="${NV_CUDNN_VERSION}"

RUN yum install -y \
    ${NV_CUDNN_PACKAGE} \
    && yum clean all \
    && rm -rf /var/cache/yum/*