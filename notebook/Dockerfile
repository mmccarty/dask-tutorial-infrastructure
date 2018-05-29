FROM jupyter/base-notebook

USER root
RUN apt-get update \
  && apt-get install -yq --no-install-recommends graphviz git \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*

RUN apt-get install -y --no-install-recommends ca-certificates apt-transport-https gnupg-curl && \
    rm -rf /var/lib/apt/lists/* && \
    NVIDIA_GPGKEY_SUM=d1be581509378368edeec8c1eb2958702feedf3bc3d17011adbf24efacce4ab5 && \
    NVIDIA_GPGKEY_FPR=ae09fe4bbd223a84b2ccfce3f60f4b3d7fa2af80 && \
    apt-key adv --fetch-keys https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/7fa2af80.pub && \
    apt-key adv --export --no-emit-version -a $NVIDIA_GPGKEY_FPR | tail -n +5 > cudasign.pub && \
    echo "$NVIDIA_GPGKEY_SUM  cudasign.pub" | sha256sum -c --strict - && rm cudasign.pub && \
    echo "deb https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64 /" > /etc/apt/sources.list.d/cuda.list && \
    echo "deb https://developer.download.nvidia.com/compute/machine-learning/repos/ubuntu1604/x86_64 /" > /etc/apt/sources.list.d/nvidia-ml.list

ENV CUDA_VERSION 9.0.176

ENV CUDA_PKG_VERSION 9-0=$CUDA_VERSION-1
RUN apt-get update && apt-get install -y --no-install-recommends \
        cuda-cudart-$CUDA_PKG_VERSION && \
    ln -s cuda-9.0 /usr/local/cuda && \
    rm -rf /var/lib/apt/lists/*

# nvidia-docker 1.0
LABEL com.nvidia.volumes.needed="nvidia_driver"
LABEL com.nvidia.cuda.version="${CUDA_VERSION}"

RUN echo "/usr/local/nvidia/lib" >> /etc/ld.so.conf.d/nvidia.conf && \
    echo "/usr/local/nvidia/lib64" >> /etc/ld.so.conf.d/nvidia.conf

ENV PATH /usr/local/nvidia/bin:/usr/local/cuda/bin:${PATH}
ENV LD_LIBRARY_PATH /usr/local/nvidia/lib:/usr/local/nvidia/lib64

# nvidia-container-runtime
ENV NVIDIA_VISIBLE_DEVICES all
ENV NVIDIA_DRIVER_CAPABILITIES compute,utility
ENV NVIDIA_REQUIRE_CUDA "cuda>=9.0"

USER $NB_USER

RUN conda install --yes \
    bokeh \
    cytoolz \
    dask=0.17.4 \
    dask-ml \
    distributed=1.21.8 \
    fastparquet \
    gcsfs \
    ipywidgets \
    jupyterlab \
    lz4 \
    matplotlib \
    netcdf4 \
    nomkl \
    numba \
    numcodecs \
    numpy \
    pandas=0.22.0 \
    python-blosc \
    s3fs \
    scipy \
    scikit-image \
    scikit-learn \
    tornado \
    xarray \
    zict \
    && conda clean -tipsy

RUN pip install kubernetes graphviz --upgrade --no-cache-dir

RUN pip install daskernetes==0.1.3 \
                git+https://github.com/mrocklin/dask-kubernetes@e9456dbca8989a4551661aa344cd15f30af428cf \
                git+https://github.com/zarr-developers/zarr \
                git+https://github.com/pydata/xarray \
                git+https://github.com/dask/gcsfs@2fbdc27e838a531ada080886ae778cb370ae48b8\
                git+https://github.com/jupyterhub/nbserverproxy \
                --no-cache-dir \
                --upgrade
RUN jupyter labextension install \
    @jupyter-widgets/jupyterlab-manager \
    @jupyterlab/hub-extension \
    jupyterlab_bokeh

RUN jupyter serverextension enable --py nbserverproxy --sys-prefix

USER root

COPY prepare.sh /usr/bin/prepare.sh
RUN chmod +x /usr/bin/prepare.sh
RUN mkdir /home/$NB_USER/examples && chown -R $NB_USER /home/$NB_USER/examples
RUN git clone https://github.com/TomAugspurger/dask-tutorial-pycon-2018 /pre-home/ && chown -R $NB_USER /pre-home
# COPY examples/ /pre-home/examples/
COPY worker-template.yaml /pre-home

ENV DASK_TICK_MAXIMUM_DELAY=5s
ENV DASKERNETES_DIAGNOSTICS_LINK {JUPYTERHUB_SERVICE_PREFIX}proxy/{port}/status
ENV DASKERNETES_WORKER_TEMPLATE_PATH /home/$NB_USER/worker-template.yaml
ENV DASKERNETES_WORKER_NAME dask-{JUPYTERHUB_USER}-{uuid}
ENV DASK_DIAGNOSTICS_LINK {JUPYTERHUB_SERVICE_PREFIX}proxy/{port}/status
ENV DASK_KUBERNETES_WORKER_TEMPLATE_PATH /home/$NB_USER/worker-template.yaml
ENV DASK_KUBERNETES_WORKER_NAME dask-{JUPYTERHUB_USER}-{uuid}
ENV DASK_KUBERNETES_MAXIMUM_WORKERS 12

RUN mkdir /gcs && chown -R $NB_USER /gcs
RUN mkdir /opt/app

# Add NB_USER to sudo
RUN echo "$NB_USER ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/notebook
RUN sed -ri "s#Defaults\s+secure_path=\"([^\"]+)\"#Defaults secure_path=\"\1:$CONDA_DIR/bin\"#" /etc/sudoers
USER $NB_USER

ENTRYPOINT ["tini", "--", "/usr/bin/prepare.sh"]
CMD ["start.sh jupyter lab"]
