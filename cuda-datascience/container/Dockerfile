# ARGS found in the different build layers
ARG BASE_IMAGE

####################################
# Generic Data Science + Streamlit #
####################################

FROM ${BASE_IMAGE}

USER root

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
