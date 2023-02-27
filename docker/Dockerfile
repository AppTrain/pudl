FROM condaforge/mambaforge:4.12.0-0

# Install curl and js
# awscli requires unzip, less, groff and mandoc
# hadolint ignore=DL3008
RUN apt-get update && apt-get install --no-install-recommends -y curl jq unzip less groff mandoc \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Configure gsutil authentication
# hadolint ignore=DL3059
RUN printf '[GoogleCompute]\nservice_account = default' > /etc/boto.cfg

# Install awscli2
RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && unzip awscliv2.zip && ./aws/install

# Create a non-root user inside the container
# hadolint ignore=DL3059
RUN useradd -Ums /bin/bash catalyst

ENV CONTAINER_HOME=/home/catalyst
ENV CONDA_PREFIX=${CONTAINER_HOME}/env
ENV CONDA_RUN="conda run --no-capture-output --prefix ${CONDA_PREFIX}"
ENV PYTHON_VERSION="3.10"
ENV PUDL_REPO=${CONTAINER_HOME}/pudl
ENV CONTAINER_PUDL_IN=${CONTAINER_HOME}/pudl_in
ENV CONTAINER_PUDL_OUT=${CONTAINER_HOME}/pudl_out
ENV CONTAINER_DAGSTER_HOME=${CONTAINER_HOME}/dagster_home

USER catalyst
WORKDIR ${CONTAINER_HOME}
RUN mkdir test

# copy environment file to home directory first - so we can cache the environment
COPY ./test/test-environment.yml ${CONTAINER_HOME}/test/test-environment.yml
COPY ./setup.py ${CONTAINER_HOME}/setup.py

# mamba create can't read environment.yml, and mamba env isn't installed yet,
# so we create a bare py3.10 env first:
# hadolint ignore=SC2035
RUN mamba create --copy --prefix ${CONDA_PREFIX} --yes python=${PYTHON_VERSION} && \
    mamba env update --prefix ${CONDA_PREFIX} --file test/test-environment.yml && \
    conda clean -afy && \
    ${CONDA_RUN} python setup.py egg_info && \
    ${CONDA_RUN} pip install `grep -v '^\[' ./*.egg-info/requires.txt` && \
    rm -rf *.egg-info/

# Copy the cloned pudl repository into the user's home directory
COPY . ${CONTAINER_HOME}

# Install PUDL
RUN --mount=type=bind,source=.git,target=${PUDL_REPO}/.git \
    ${CONDA_RUN} pip install --no-cache-dir -e './[dev,doc,test,datasette]' && \
    mkdir ${CONTAINER_PUDL_IN} ${CONTAINER_PUDL_OUT} ${CONTAINER_DAGSTER_HOME} && \
    ${CONDA_RUN} pudl_setup --pudl_in ${CONTAINER_PUDL_IN} --pudl_out ${CONTAINER_PUDL_OUT}

# Run the unit tests:
CMD ["conda", "run", "--no-capture-output", "--prefix", "${CONDA_PREFIX}", "pytest", "test/unit"]
