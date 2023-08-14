# Stage 1: Base
FROM nvidia/cuda:11.8.0-cudnn8-devel-ubuntu22.04 as base

SHELL ["/bin/bash", "-o", "pipefail", "-c"]
ENV DEBIAN_FRONTEND=noninteractive \
    TZ=Europe/London \
    PYTHONUNBUFFERED=1 \
    SHELL=/bin/bash

# Create workspace working directory
WORKDIR /

# Install Ubuntu packages
RUN apt update && \
    apt -y upgrade && \
    apt install -y --no-install-recommends \
        build-essential \
        software-properties-common \
        python3.10-venv \
        python3-pip \
        python3-tk \
        python3-dev \
        bash \
        git \
        ncdu \
        nginx \
        net-tools \
        nvtop \
        openssh-server \
        libglib2.0-0 \
        libsm6 \
        libgl1 \
        libxrender1 \
        libxext6 \
        ffmpeg \
        wget \
        curl \
        psmisc \
        rsync \
        vim \
        zip \
        unzip \
        p7zip-full \
        htop \
        pkg-config \
        libcairo2-dev \
        libgoogle-perftools4 libtcmalloc-minimal4 \
        apt-transport-https ca-certificates && \
    update-ca-certificates && \
    apt clean && \
    rm -rf /var/lib/apt/lists/* && \
    echo "en_US.UTF-8 UTF-8" > /etc/locale.gen

# Set Python
RUN ln -s /usr/bin/python3.10 /usr/bin/python

# Stage 2: Install applications
FROM base as setup

ARG WEBUI_VERSION=v1.5.1
ARG DREAMBOOTH_COMMIT=c2a5617c587b812b5a408143ddfb18fc49234edf
ARG KOHYA_VERSION=v21.8.7
ARG COMFYUI_COMMIT=8c730dc4a7225b2b103497ea1c2f67e48db145ab

# Clone the git repo of the Stable Diffusion Web UI by Automatic1111
# and set version
WORKDIR /
RUN git clone --depth=1 --branch $WEBUI_VERSION --single-branch https://github.com/AUTOMATIC1111/stable-diffusion-webui.git

WORKDIR /stable-diffusion-webui
RUN python3 -m venv --system-site-packages /venv && \
    source /venv/bin/activate && \
    pip install --no-cache-dir torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118 && \
    pip install --no-cache-dir xformers && \
    deactivate

# Install the dependencies for the Automatic1111 Stable Diffusion Web UI
COPY requirements.txt requirements_versions.txt ./
COPY cache-sd-model.py install-automatic.py ./
RUN source /venv/bin/activate && \
    python -m install-automatic --skip-torch-cuda-test && \
    deactivate

# Clone the Automatic1111 Extensions
RUN git clone https://github.com/d8ahazard/sd_dreambooth_extension.git extensions/sd_dreambooth_extension && \
    git clone --depth=1 --single-branch https://github.com/deforum-art/sd-webui-deforum.git extensions/deforum && \
    git clone --depth=1 --single-branch https://github.com/Mikubill/sd-webui-controlnet.git extensions/sd-webui-controlnet && \
    git clone --depth=1 --single-branch https://github.com/ashleykleynhans/a1111-sd-webui-locon.git extensions/a1111-sd-webui-locon && \
    git clone --depth=1 --single-branch https://github.com/s0md3v/sd-webui-roop.git extensions/sd-webui-roop && \
    git clone --depth=1 --single-branch https://github.com/wcde/sd-webui-refiner.git extensions/sd-webui-refiner

# Install depenencies for Deforum, ControlNet and roop
RUN source /venv/bin/activate && \
    cd /stable-diffusion-webui/extensions/deforum && \
    pip3 install -r requirements.txt && \
    cd /stable-diffusion-webui/extensions/sd-webui-controlnet && \
    pip3 install -r requirements.txt && \
    cd /stable-diffusion-webui/extensions/sd-webui-roop && \
    pip3 install -r requirements.txt && \
    deactivate

# Set Dreambooth extension version
WORKDIR /stable-diffusion-webui/extensions/sd_dreambooth_extension
RUN git checkout main && \
    git reset $DREAMBOOTH_COMMIT --hard

# Install the dependencies for the Dreambooth extension
WORKDIR /stable-diffusion-webui
COPY requirements_dreambooth.txt ./requirements.txt
RUN source /venv/bin/activate && \
    cd /stable-diffusion-webui/extensions/sd_dreambooth_extension && \
    pip3 install -r requirements.txt && \
    deactivate

# Fix Tensorboard
RUN source /venv/bin/activate && \
    pip3 uninstall -y tensorboard tb-nightly && \
    pip3 install tensorboard tensorflow && \
    pip3 cache purge && \
    deactivate

# Install Kohya_ss
RUN git clone --depth=1 --branch $KOHYA_VERSION --single-branch https://github.com/bmaltais/kohya_ss.git /kohya_ss
WORKDIR /kohya_ss
RUN python3 -m venv --system-site-packages venv && \
    source venv/bin/activate && \
    pip3 install --no-cache-dir torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118 && \
    pip3 install --no-cache-dir xformers==0.0.20 \
        bitsandbytes==0.35.0 \
        tensorboard==2.12.3 \
        tensorflow==2.12.0 \
        wheel \
        tensorrt && \
    pip3 install -r requirements.txt && \
    pip3 install . && \
    pip3 cache purge && \
    deactivate

# Install ComfyUI
RUN git clone --depth=1 --single-branch https://github.com/comfyanonymous/ComfyUI.git /ComfyUI
WORKDIR /ComfyUI
RUN python3 -m venv --system-site-packages venv && \
    source venv/bin/activate && \
    pip3 install --no-cache-dir torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/cu118 xformers && \
    pip3 install --no-cache-dir wheel && \
    pip3 install -r requirements.txt && \
    pip3 cache purge && \
    deactivate

# Install ComfyUI manager
WORKDIR /ComfyUI/custom_nodes
RUN git clone --depth=1 --single-branch https://github.com/ltdrdata/ComfyUI-Manager.git

# Install Jupyter
RUN pip3 install -U --no-cache-dir jupyterlab \
        jupyterlab_widgets \
        ipykernel \
        ipywidgets \
        gdown

# Install runpodctl
RUN wget https://github.com/runpod/runpodctl/releases/download/v1.10.0/runpodctl-linux-amd -O runpodctl && \
    chmod a+x runpodctl && \
    mv runpodctl /usr/local/bin

# Install CivitAI Model Downloader
RUN git clone --depth=1 --single-branch https://github.com/ashleykleynhans/civitai-downloader.git && \
    mv civitai-downloader/download.sh /usr/local/bin/download-model && \
    chmod +x /usr/local/bin/download-model

# Copy Stable Diffusion Web UI config files
COPY relauncher.py webui-user.sh config.json ui-config.json /stable-diffusion-webui/

# Copy ComfyUI config files
COPY extra_model_paths.yaml /ComfyUI/

# NGINX Proxy
COPY nginx.conf /etc/nginx/nginx.conf
COPY 502.html /usr/share/nginx/html/502.html

# Copy the README.md
COPY README.md /usr/share/nginx/html/README.md

# Set up the container startup script
WORKDIR /
COPY pre_start.sh start.sh fix_venv.sh accelerate.yaml download_sdxl_models.sh ./
RUN chmod +x /start.sh && \
    chmod +x /pre_start.sh && \
    chmod +x /fix_venv.sh && \
    chmod +x /download_sdxl_models.sh

# Cleanup
WORKDIR /
RUN rm -rf /root/.cache/pip

# Start the container
SHELL ["/bin/bash", "--login", "-c"]
CMD [ "/start.sh" ]
