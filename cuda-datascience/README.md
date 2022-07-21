# CUDA Generic Data Science Notebook

Custom notebook based on RHODS standard data science with:

* CUDA.

* [Streamlit](https://streamlit.io/) ([Github repo](https://github.com/streamlit/streamlit))

Streamlit turns data scripts into shareable web apps in minutes.
All in pure Python. No front‑end experience required.
See [details on Streamlit implementation](../streamlit-notebook/README.md).

Build example - Standard image:

```bash
podman build --build-arg=BASE_IMAGE=s2i-minimal-notebook-cuda116:0.0.1 -t s2i-cuda-datascience:0.0.1 .
```

Build example - Devel image:

```bash
podman build --build-arg=BASE_IMAGE=s2i-minimal-notebook-cuda116-devel:0.0.1 -t s2i-cuda-datascience-devel:0.0.1 .
```
