### 
ARG REGISTRY=quay.io
ARG OWNER=jupyter
ARG PYTHON_VERSION=3.11
ARG BASE_IMAGE=${REGISTRY}/${OWNER}/pyspark-notebook:python-${MLRUN_PYTHON_VERSION}


# 使用quay.io/jupyter/pyspark-notebook:latest作为基础镜像
# FROM quay.io/jupyter/pyspark-notebook::${PYTHON_VERSION}
# Base image info: https://jupyter-docker-stacks.readthedocs.io/en/latest/using/selecting.html
# Base image tags: https://quay.io/repository/jupyter/minimal-notebook
# Customization is based on: https://github.com/jupyter-incubator/sparkmagic/blob/master/Dockerfile.jupyter
FROM ${BASE_IMAGE}


###
LABLE


### 
USER root



### 设置工作目录为/app
WORKDIR /app

# 将requirements.txt文件复制到容器中
COPY requirements.txt /app/requirements.txt

# 使用pip安装requirements.txt中列出的Python库
RUN pip install --no-cache-dir -r requirements.txt

# 可选：如果你需要将其他文件或目录复制到容器中，可以使用COPY或ADD命令
# COPY . /app 或者 ADD . /app

# 可选：暴露端口，如果需要的话（例如，如果你的应用程序运行在特定的端口上）
EXPOSE 8888

# 可选：设置容器启动时执行的命令，例如启动Jupyter Notebook
# CMD ["jupyter", "notebook", "--ip=0.0.0.0", "--no-browser", "--allow-root"]


# ---------

FROM quay.io/jupyter/minimal-notebook:2024-02-13
ARG TARGETPLATFORM
ARG KAMU_VERSION
ARG dev_mode=false


#########################################################################################
USER root

# Podman
# Source: https://github.com/containers/podman/blob/056f492f59c333d521ebbbe186abde0278e815db/contrib/podmanimage/stable/Dockerfile
RUN apt update && \
    apt -y install ca-certificates curl wget gnupg unzip jq && \
    . /etc/os-release && \
    echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/ /" | tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list && \
    curl -L "https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/Release.key" | apt-key add - && \
    apt update && \
    apt -y install podman fuse-overlayfs && \
    apt-get clean && rm -rf /var/lib/apt/lists /var/cache/apt/archives

COPY podman/containers.conf /etc/containers/containers.conf
COPY podman/storage.conf /etc/containers/storage.conf
COPY podman/containers-user.conf /home/$NB_USER/.config/containers/containers.conf
COPY podman/storage-user.conf /home/$NB_USER/.config/containers/storage.conf

# Create empty storage not to get errors when it's not mounted 
# See: https://www.redhat.com/sysadmin/image-stores-podman
RUN mkdir -p \
    /var/lib/containers/shared/overlay-images \ 
    /var/lib/containers/shared/overlay-layers \
    /var/lib/containers/shared/vfs-images \
    /var/lib/containers/shared/vfs-layers && \
    touch /var/lib/containers/shared/overlay-images/images.lock && \
    touch /var/lib/containers/shared/overlay-layers/layers.lock && \
    touch /var/lib/containers/shared/vfs-images/images.lock && \
    touch /var/lib/containers/shared/vfs-layers/layers.lock


# AWS CLI
RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && \
    unzip awscliv2.zip && \
    ./aws/install && \
    rm -rf aws*


# Sparkmagic and tools
COPY jupyter/requirements/$TARGETPLATFORM/requirements.txt requirements.txt

# TODO: Semi-permanent hack for `mapboxgl` package being broken in conda-forge
# See: https://github.com/kamu-data/kamu-cli/issues/533
RUN mamba install -y --file requirements.txt && \
    mamba uninstall mapboxgl && pip install --no-cache-dir mapboxgl && \
    mamba clean --all -f -y &&  \
    rm requirements.txt &&  \
    fix-permissions "${CONDA_DIR}" &&  \
    fix-permissions "/home/${NB_USER}"


# Kamu
RUN curl -s "https://get.kamu.dev" | KAMU_VERSION=$KAMU_VERSION KAMU_INSTALL_PATH=/usr/local/bin/kamu sh && \
    echo "source <(kamu completions bash)" >> /home/$NB_USER/.bashrc

COPY jupyter/kamu-start-hook.sh /usr/local/bin/before-notebook.d/kamu-start-hook.sh
COPY jupyter/.kamuconfig /.kamuconfig


#########################################################################################
USER $NB_USER

COPY jupyter/kamu.py /opt/conda/lib/python3.11/site-packages/kamu.py
COPY jupyter/sparkmagic.json /home/$NB_USER/.sparkmagic/config.json

RUN jupyter nbextension enable --py --sys-prefix widgetsnbextension
#RUN jupyter-kernelspec install --user $(pip show sparkmagic | grep Location | cut -d" " -f2)/sparkmagic/kernels/sparkkernel
RUN jupyter-kernelspec install --user $(pip show sparkmagic | grep Location | cut -d" " -f2)/sparkmagic/kernels/pysparkkernel
#RUN jupyter-kernelspec install --user $(pip show sparkmagic | grep Location | cut -d" " -f2)/sparkmagic/kernels/sparkrkernel
RUN jupyter serverextension enable --py sparkmagic


#########################################################################################
USER root

RUN rm -r /home/$NB_USER/work
COPY user-home/ /home/$NB_USER/
RUN fix-permissions "/home/${NB_USER}"


#########################################################################################
USER $NB_USER

# TODO: Remove show_banner option after Sparkmagic supports novebook >= 7.0.0
# See: https://github.com/jupyter-incubator/sparkmagic/issues/885
CMD ["jupyter", "notebook", "--ip", "0.0.0.0", "--port", "8080", "--NotebookApp.iopub_data_rate_limit=1e10", "--NotebookApp.show_banner=False"]





FROM quay.io/thoth-station/s2i-thoth-ubi8-py38:v0.29.0

LABEL name="OpenVINO(TM) Notebooks" \
  maintainer="helena.kloosterman@intel.com" \
  vendor="Intel Corporation" \
  version="0.2.0" \
  release="2021.4" \
  summary="OpenVINO(TM) Developer Tools and Jupyter Notebooks" \
  description="OpenVINO(TM) Notebooks Container"

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

USER root

# Upgrade NodeJS > 12.0
# Install dos2unix for line end conversion on Windows
RUN dnf --disableplugin=subscription-manager remove -y nodejs && \
  dnf --disableplugin=subscription-manager module -y reset nodejs && \
  dnf --disableplugin=subscription-manager module -y enable nodejs:20 && \
  dnf --disableplugin=subscription-manager install -y nodejs mesa-libGL dos2unix libsndfile python38-tkinter && \
  dnf --disableplugin=subscription-manager -y update-minimal --security --sec-severity=Important --sec-severity=Critical --sec-severity=Moderate

# GPU drivers
RUN dnf --disableplugin=subscription-manager install -y 'dnf-command(config-manager)' && \
  dnf --disableplugin=subscription-manager config-manager --add-repo  https://repositories.intel.com/graphics/rhel/8.6/intel-graphics.repo

RUN rpm -ivh https://vault.centos.org/centos/8/AppStream/x86_64/os/Packages/mesa-filesystem-21.1.5-1.el8.x86_64.rpm && \
  dnf --disableplugin=subscription-manager install --refresh -y \
  intel-opencl intel-media intel-mediasdk libmfxgen1 libvpl2 \
  level-zero intel-level-zero-gpu \
  intel-metrics-library intel-igc-core intel-igc-cm \
  libva libva-utils  intel-gmmlib && \
  rpm -ivh http://vault.centos.org/centos/8-stream/AppStream/x86_64/os/Packages/ocl-icd-2.2.12-1.el8.x86_64.rpm && \
  rpm -ivh https://dl.fedoraproject.org/pub/epel/8/Everything/x86_64/Packages/c/clinfo-3.0.21.02.21-4.el8.x86_64.rpm

# Copying in override assemble/run scripts
COPY .docker/.s2i/bin /tmp/scripts
# Copying in source code
COPY .docker /tmp/src
COPY .ci/patch_notebooks.py /tmp/scripts
COPY .ci/validate_notebooks.py /tmp/scripts
COPY .ci/validation_config.py /tmp/scripts
COPY .ci/ignore_treon_docker.txt /tmp/scripts
# workaround for coping file if it does not exists
COPY .ci/test_notebooks.* /tmp/scripts

# Git on Windows may convert line endings. Run dos2unix to enable
# building the image when the scripts have CRLF line endings.
RUN dos2unix /tmp/scripts/*
RUN dos2unix /tmp/src/builder/*

# Change file ownership to the assemble user. Builder image must support chown command.
RUN chown -R 1001:0 /tmp/scripts /tmp/src
USER 1001
RUN mkdir /opt/app-root/notebooks
COPY notebooks/ /opt/app-root/notebooks
RUN /tmp/scripts/assemble
RUN pip check
USER root
RUN dos2unix /opt/app-root/bin/*sh
RUN yum remove -y dos2unix
RUN chown -R 1001:0 .
RUN chown -R 1001:0 /opt/app-root/notebooks
USER 1001
# RUN jupyter lab build
CMD /tmp/scripts/run


-----
FROM        jupyter/all-spark-notebook:latest

USER        root

RUN         pip install jupyter_contrib_nbextensions \
                && jupyter contrib nbextension install --system \
                && pip install jupyter_nbextensions_configurator \
                && jupyter nbextensions_configurator enable --system \
                && pip install yapf # for code pretty

USER        $NB_UID



### Use jupyter/all-spark-notebooks with an existing Spark/YARN cluster

```{warning}
This recipe is not tested and might be broken.
```

```dockerfile
FROM quay.io/jupyter/all-spark-notebook

# Set env vars for pydoop
ENV HADOOP_HOME /usr/local/hadoop-2.7.3
ENV JAVA_HOME /usr/lib/jvm/java-8-openjdk-amd64
ENV HADOOP_CONF_HOME /usr/local/hadoop-2.7.3/etc/hadoop
ENV HADOOP_CONF_DIR /usr/local/hadoop-2.7.3/etc/hadoop

USER root
# Add proper open-jdk-8 not the jre only, needed for pydoop
RUN echo 'deb https://cdn-fastly.deb.debian.org/debian jessie-backports main' > /etc/apt/sources.list.d/jessie-backports.list && \
    apt-get update --yes && \
    apt-get install --yes --no-install-recommends -t jessie-backports openjdk-8-jdk && \
    rm /etc/apt/sources.list.d/jessie-backports.list && \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
# Add Hadoop binaries
    wget --progress=dot:giga https://mirrors.ukfast.co.uk/sites/ftp.apache.org/hadoop/common/hadoop-2.7.3/hadoop-2.7.3.tar.gz && \
    tar -xvf hadoop-2.7.3.tar.gz -C /usr/local && \
    chown -R "${NB_USER}:users" /usr/local/hadoop-2.7.3 && \
    rm -f hadoop-2.7.3.tar.gz && \
# Install os dependencies required for pydoop, pyhive
    apt-get update --yes && \
    apt-get install --yes --no-install-recommends build-essential python-dev libsasl2-dev && \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
# Remove the example hadoop configs and replace
# with those for our cluster.
# Alternatively, this could be mounted as a volume
    rm -f /usr/local/hadoop-2.7.3/etc/hadoop/*

# Download this from ambari/cloudera manager and copy it here
COPY example-hadoop-conf/ /usr/local/hadoop-2.7.3/etc/hadoop/

# Spark-Submit doesn't work unless I set the following
RUN echo "spark.driver.extraJavaOptions -Dhdp.version=2.5.3.0-37" >> /usr/local/spark/conf/spark-defaults.conf && \
    echo "spark.yarn.am.extraJavaOptions -Dhdp.version=2.5.3.0-37" >> /usr/local/spark/conf/spark-defaults.conf && \
    echo "spark.master=yarn" >>  /usr/local/spark/conf/spark-defaults.conf && \
    echo "spark.hadoop.yarn.timeline-service.enabled=false" >> /usr/local/spark/conf/spark-defaults.conf && \
    chown -R "${NB_USER}:users" /usr/local/spark/conf/spark-defaults.conf && \
    # Create an alternative HADOOP_CONF_HOME so we can mount as a volume and repoint
    # using ENV var if needed
    mkdir -p /etc/hadoop/conf/ && \
    chown "${NB_USER}":users /etc/hadoop/conf/

USER ${NB_UID}

# Install useful jupyter extensions and python libraries like :
# - Dashboards
# - PyDoop
# - PyHive
RUN pip install --no-cache-dir 'jupyter_dashboards' 'faker' && \
    jupyter dashboards quick-setup --sys-prefix && \
    pip2 install --no-cache-dir 'pyhive' 'pydoop' 'thrift' 'sasl' 'thrift_sasl' 'faker' && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

USER root
# Ensure we overwrite the kernel config so that toree connects to cluster
RUN jupyter toree install --sys-prefix --spark_opts="\
    --master yarn \
    --deploy-mode client \
    --driver-memory 512m \
    --executor-memory 512m \
    --executor-cores 1 \
    --driver-java-options \
    -Dhdp.version=2.5.3.0-37 \
    --conf spark.hadoop.yarn.timeline-service.enabled=false \
"
USER ${NB_UID}
```



ARG BASE_IMAGE=quay.io/modh/runtime-images:runtime-datascience-ubi9-python-3.9-2024a-20240523-70df141
FROM ${BASE_IMAGE} as builder

ARG UBI_VERSION
ARG PYTHON_VERSION
ARG PYTHON_VERSION_LONG
ARG RELEASE
ARG DATE
ARG CUDA

# Build options
ARG SPARK_VERSION=3.3.1
ARG HADOOP_VERSION=3.3.4
ARG JMX_PROMETHEUS_JAVAAGENT_VERSION=0.17.0
# Spark's Guava version to match with Hadoop's
ARG GUAVA_VERSION=27.0-jre

LABEL name="workbench-images:${CUDA}jupyter-spark-${UBI_VERSION}-${PYTHON_VERSION}_${RELEASE}_${DATE}" \
    summary="Datascience + Spark JupyterLab notebook with ${PYTHON_VERSION_LONG} based on ${UBI_VERSION}" \
    description="Datascience + Spark JupyterLab notebook with ${PYTHON_VERSION_LONG} based on ${UBI_VERSION}" \
    io.k8s.description="Datascience + Spark JupyterLab notebook with ${PYTHON_VERSION_LONG} based on ${UBI_VERSION} for ODH or RHODS" \
    io.k8s.display-name="Datascience + Spark JupyterLab notebook with ${PYTHON_VERSION_LONG} based on ${UBI_VERSION}" \
    authoritative-source-url="https://github.com/opendatahub-io-contrib/workbench-images" \
    io.openshift.build.commit.ref="${RELEASE}" \
    io.openshift.build.source-location="https://github.com/opendatahub-io-contrib/workbench-images/jupyter/spark" \
    io.openshift.build.image="https://quay.io/opendatahub-contrib/workbench-images:${CUDA}jupyter-spark-${UBI_VERSION}-${PYTHON_VERSION}_${RELEASE}_${DATE}"

USER 0

WORKDIR /

# Install gzip to extract archives
RUN dnf install -y gzip && \
    dnf clean all

# Download Spark
ADD https://archive.apache.org/dist/spark/spark-${SPARK_VERSION}/spark-${SPARK_VERSION}-bin-without-hadoop.tgz .
# Unzip Spark
RUN tar -xvzf spark-${SPARK_VERSION}-bin-without-hadoop.tgz --no-same-owner
RUN mv spark-${SPARK_VERSION}-bin-without-hadoop spark

# Download Hadoop
ADD https://archive.apache.org/dist/hadoop/common/hadoop-${HADOOP_VERSION}/hadoop-${HADOOP_VERSION}.tar.gz .
# Unzip Hadoop
RUN tar -xvzf hadoop-${HADOOP_VERSION}.tar.gz --no-same-owner
RUN mv hadoop-${HADOOP_VERSION} hadoop
# Delete unnecessary hadoop documentation
RUN rm -rf hadoop/share/doc

# Download JMX Prometheus javaagent jar
ADD https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/${JMX_PROMETHEUS_JAVAAGENT_VERSION}/jmx_prometheus_javaagent-${JMX_PROMETHEUS_JAVAAGENT_VERSION}.jar /prometheus/
RUN chmod 0644 prometheus/jmx_prometheus_javaagent*.jar

# Add updated Guava
WORKDIR /spark/jars
RUN rm -f guava-*.jar
ADD https://repo1.maven.org/maven2/com/google/guava/guava/${GUAVA_VERSION}/guava-${GUAVA_VERSION}.jar .

# Add Spark Hadoop Cloud to interact with cloud infrastructures
ADD https://repo1.maven.org/maven2/org/apache/spark/spark-hadoop-cloud_2.12/${SPARK_VERSION}/spark-hadoop-cloud_2.12-${SPARK_VERSION}.jar .

### Build final image
FROM ${BASE_IMAGE}

ARG UBI_VERSION
ARG PYTHON_VERSION
ARG PYTHON_VERSION_LONG
ARG JAVA_VERSION=1.8.0
ARG PKG_ROOT=/opt/app-root
ARG SPARK_VERSION=3.3.1
ARG HADOOP_VERSION=3.3.4
ARG JMX_PROMETHEUS_JAVAAGENT_VERSION=0.17.0

LABEL name="workbench-images:${CUDA}jupyter-spark-${UBI_VERSION}-${PYTHON_VERSION}_${RELEASE}_${DATE}" \
    summary="Datascience + Spark JupyterLab notebook with ${PYTHON_VERSION_LONG} based on ${UBI_VERSION}" \
    description="Datascience + Spark JupyterLab notebook with ${PYTHON_VERSION_LONG} based on ${UBI_VERSION}" \
    io.k8s.description="Datascience + Spark JupyterLab notebook with ${PYTHON_VERSION_LONG} based on ${UBI_VERSION} for ODH or RHODS" \
    io.k8s.display-name="Datascience + Spark JupyterLab notebook with ${PYTHON_VERSION_LONG} based on ${UBI_VERSION}" \
    authoritative-source-url="https://github.com/opendatahub-io-contrib/workbench-images" \
    io.openshift.build.commit.ref="${RELEASE}" \
    io.openshift.build.source-location="https://github.com/opendatahub-io-contrib/workbench-images/jupyter/spark" \
    io.openshift.build.image="https://quay.io/opendatahub-contrib/workbench-images:${CUDA}jupyter-spark-${UBI_VERSION}-${PYTHON_VERSION}_${RELEASE}_${DATE}"

USER 0

WORKDIR ${PKG_ROOT}/${SPARK_VERSION}

####################
# OpenJDK          #
####################

RUN yum -y install java-$JAVA_VERSION-openjdk maven &&\
    yum clean all

####################
# Spark            #
####################

# Copy Spark from builder stage
COPY --from=builder --chown=default:root /spark ${PKG_ROOT}/spark-${SPARK_VERSION}
COPY --from=builder --chown=default:root /spark/kubernetes/dockerfiles/spark/entrypoint.sh /opt/app-root/bin

# Copy Hadoop from builder stage
COPY --from=builder --chown=default:root /hadoop ${PKG_ROOT}/hadoop-${HADOOP_VERSION}

# Copy Prometheus jars from builder stage
COPY --from=builder --chown=default:root /prometheus ${PKG_ROOT}/prometheus

# Setup required env vars for spark and hadoop
ENV JAVA_HOME=/usr/lib/jvm/jre
ENV SPARK_HOME=${PKG_ROOT}/spark-${SPARK_VERSION}
ENV HADOOP_HOME ${PKG_ROOT}/hadoop-${HADOOP_VERSION}

ENV SPARK_DIST_CLASSPATH="$HADOOP_HOME/etc/hadoop:$HADOOP_HOME/share/hadoop/common/lib/*:$HADOOP_HOME/share/hadoop/common/*:$HADOOP_HOME/share/hadoop/hdfs:$HADOOP_HOME/share/hadoop/hdfs/lib/*:$HADOOP_HOME/share/hadoop/hdfs/*:$HADOOP_HOME/share/hadoop/yarn:$HADOOP_HOME/share/hadoop/yarn/lib/*:$HADOOP_HOME/share/hadoop/yarn/*:$HADOOP_HOME/share/hadoop/mapreduce/lib/*:$HADOOP_HOME/share/hadoop/mapreduce/*:/contrib/capacity-scheduler/*.jar:$HADOOP_HOME/share/hadoop/tools/lib/*"

ENV SPARK_EXTRA_CLASSPATH="$SPARK_DIST_CLASSPATH"

ENV LD_LIBRARY_PATH /lib64

ENV PATH="${PATH}:${PKG_ROOT}/spark-${SPARK_VERSION}/bin"

WORKDIR /opt/app-root/src

USER 1001

# Install PySpark and cleanup
RUN pip install --no-cache-dir -r requirements.txt && \
    fix-permissions /opt/app-root -P

ENTRYPOINT ["start-notebook.sh"] 




----
  # Copyright 2018 Iguazio
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

ARG MLRUN_PYTHON_VERSION=3.9.13

FROM quay.io/mlrun/jupyter-scipy-notebook:python-${MLRUN_PYTHON_VERSION}

USER root
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get -y upgrade && \
    rm -rf /var/lib/apt/lists/*

RUN apt-get update && apt-get install --no-install-recommends -y \
  graphviz \
 && rm -rf /var/lib/apt/lists/*
USER $NB_UID

ENV PIP_NO_CACHE_DIR=1

ARG MLRUN_PIP_VERSION=22.3.0
RUN python -m pip install --upgrade pip~=${MLRUN_PIP_VERSION}

WORKDIR $HOME

COPY --chown=$NB_UID:$NB_GID ./dockerfiles/jupyter/README.ipynb $HOME
COPY --chown=$NB_UID:$NB_GID ./dockerfiles/jupyter/mlrun.env $HOME
COPY --chown=$NB_UID:$NB_GID ./docs/tutorial $HOME/tutorial
COPY --chown=$NB_UID:$NB_GID ./docs/_static/images/tutorial $HOME/_static/images/tutorial
COPY --chown=$NB_UID:$NB_GID ./docs/_static/images/MLRun-logo.png $HOME/_static/images

RUN mkdir data

COPY --chown=$NB_UID:$NB_GID ./dockerfiles/jupyter/requirements.txt /tmp/requirements/jupyter-requirements.txt
COPY --chown=$NB_UID:$NB_GID ./dockerfiles/mlrun-api/requirements.txt /tmp/requirements/mlrun-api-requirements.txt
COPY --chown=$NB_UID:$NB_GID ./extras-requirements.txt /tmp/requirements/extras-requirement.txt
COPY --chown=$NB_UID:$NB_GID ./requirements.txt /tmp/requirements/requirement.txt
RUN python -m pip install \
    -r /tmp/requirements/jupyter-requirements.txt \
    -r /tmp/requirements/mlrun-api-requirements.txt \
    -r /tmp/requirements/extras-requirement.txt \
    -r /tmp/requirements/requirement.txt

COPY --chown=$NB_UID:$NB_GID . /tmp/mlrun
RUN cd /tmp/mlrun && python -m pip install ".[complete-api]"

# This will usually cause a cache miss - so keep it in the last layers
ARG MLRUN_CACHE_DATE=initial
RUN git clone --branch 1.1.x https://github.com/mlrun/demos.git $HOME/demos && \
    cd $HOME/demos && ./community_edition_align.sh && cd -
RUN git clone --branch master https://github.com/mlrun/functions.git $HOME/functions

ENV JUPYTER_ENABLE_LAB=yes \
    MLRUN_HTTPDB__DATA_VOLUME=$HOME/data \
    MLRUN_HTTPDB__DSN='sqlite:////home/jovyan/data/mlrun.db?check_same_thread=false' \
    MLRUN_HTTPDB__LOGS_PATH=$HOME/data/logs \
    MLRUN_ENV_FILE=$HOME/mlrun.env

# run the mlrun db (api) and the notebook in parallel
CMD MLRUN_ARTIFACT_PATH=$HOME/data python -m mlrun db & MLRUN_DBPATH=http://localhost:8080 start-notebook.sh \
    --ip="0.0.0.0" \
    --port=8888 \
    --NotebookApp.token='' \
    --NotebookApp.password='' \
    --NotebookApp.default_url="/lab"