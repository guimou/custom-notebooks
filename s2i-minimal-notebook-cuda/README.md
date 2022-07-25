# Base notebook

Minimal notebook build in one pass, from any base image, for example a custom-build CUDA image.

Build example - Standard image:

```bash
podman build --build-arg=BASE_IMAGE=localhost/cuda:11.6.2-cudnn8-runtime-centosstream9 -t s2i-minimal-notebook-cuda116:0.0.1 .
```

Build example - Devel image:

```bash
podman build --build-arg=BASE_IMAGE=localhost/cuda:11.6.2-cudnn8-devel-centosstream9 -t s2i-minimal-notebook-cuda116-devel:0.0.1 .
```
