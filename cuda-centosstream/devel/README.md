# Build instructions

### Build from a local base image (e.g. localhost/cuda:11.6.2-base-centosstream8):

```bash
podman build --build-arg=IMAGE_NAME=localhost/cuda -t cuda:11.6.2-devel-centosstream8 .
```

### Build CUDNN version from a local runtime image (e.g. localhost/cuda:11.6.2-runtime-centosstream8).

From the cudnn8 folder:

```bash
podman build --build-arg=IMAGE_NAME=localhost/cuda -t cuda:11.6.2-cudnn8-devel-centosstream8 .
```
