############## custom jupyter image ###############
ARG JUPYTER_BASE_IMAGE
ARG JUPYTER_BASE_VERSION
FROM jupyter/${JUPYTER_BASE_IMAGE}:${JUPYTER_BASE_VERSION} AS jupyter

ARG MLFLOW_ARTIFACT_STORE
ARG MLFLOW_VERSION
ARG JUPYTER_USERNAME
ENV HOME_DIR=/home/$JUPYTER_USERNAME
ENV WORK_DIR=$HOME_DIR/work

# Install MLflow, psycopg2 and other dependencies via conda
RUN conda install --quiet --yes \
        'mlflow='$MLFLOW_VERSION \
        'psycopg2=2.8.*' \
        'conda-build' \
        'tqdm' \
        'jupyter_contrib_nbextensions' \
        'jupyter_nbextensions_configurator' && \
    conda build purge-all && \
    fix-permissions $CONDA_DIR

USER root
# Enable Jupyter extensions and create artifact store directory
RUN ldconfig && \
    jupyter nbextension enable toc2/main --sys-prefix && \
    jupyter nbextension enable collapsible_headings/main --sys-prefix && \
    mkdir -p $MLFLOW_ARTIFACT_STORE

USER $NB_UID

# Install Python dependencies from requirements.txt
WORKDIR $WORK_DIR
COPY ./requirements.txt ./
RUN pip install --upgrade pip && \
    pip install -r ./requirements.txt && \
    # Pin protobuf to avoid MLflow protobuf descriptor errors
    pip install "protobuf<3.20.0" && \
    fix-permissions $HOME_DIR

# Install experiment code in editable mode
COPY ./setup.py ./
COPY ./src/ ./src/

USER root
RUN chown -R $NB_UID:$NB_GID ./src/
USER $NB_UID

# Install the package in editable mode
RUN pip install -e .

################## test image ###################
FROM jupyter AS test

ARG TEST_DIR=/tests

# Install test requirements
COPY ./docker/jupyter/requirements.test.txt ${TEST_DIR}/
RUN pip install -r ${TEST_DIR}/requirements.test.txt

# Copy and prepare test scripts
COPY ./tests/docker ${TEST_DIR}/docker
COPY ./tests/run_docker_tests.sh ${TEST_DIR}/
COPY ./docker/jupyter/scripts/wait-for-it.sh ${TEST_DIR}/
USER root
RUN chmod +x ${TEST_DIR}/wait-for-it.sh
USER $NB_UID

WORKDIR ${TEST_DIR}
