FROM nvidia/cuda:11.8.0-devel-ubuntu22.04

ARG WEBUI_VERSION
ARG TORCH_VERSION
ARG XFORMERS_VERSION

ENV DEBIAN_FRONTEND noninteractive
ENV SHELL=/bin/bash
ENV LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/lib/x86_64-linux-gnu
ENV PATH="/workspace/venv/bin:$PATH"
ENV PYTHONUNBUFFERED=1

WORKDIR /workspace

# Set up shell and update packages
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Install system dependencies
RUN apt update --yes && \
    apt upgrade --yes && \
    apt install --yes --no-install-recommends \
    git openssh-server libglib2.0-0 libsm6 libgl1 libxrender1 libxext6 ffmpeg wget curl psmisc rsync vim bc nginx \
    pkg-config libffi-dev libcairo2 libcairo2-dev libgoogle-perftools4 libtcmalloc-minimal4 apt-transport-https \
    software-properties-common ca-certificates && \
    update-ca-certificates

# Install Python 3.10
RUN add-apt-repository ppa:deadsnakes/ppa && \
    apt install python3.10-dev python3.10-venv -y --no-install-recommends && \
    ln -s /usr/bin/python3.10 /usr/bin/python && \
    rm /usr/bin/python3 && \
    ln -s /usr/bin/python3.10 /usr/bin/python3 && \
    curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py && \python get-pip.py && \
    pip install -U --no-cache-dir pip

# Create directories for models
RUN mkdir /sd-models && mkdir /cn-models && mkdir /comfy-models

# Download models
RUN wget https://huggingface.co/stabilityai/stable-diffusion-xl-base-1.0/resolve/main/sd_xl_base_1.0.safetensors -O /sd-models/sd_xl_base_1.0.safetensors && \
    wget https://huggingface.co/stabilityai/stable-diffusion-xl-refiner-1.0/resolve/main/sd_xl_refiner_1.0.safetensors -O /sd-models/sd_xl_refiner_1.0.safetensors && \
    wget https://huggingface.co/lllyasviel/ControlNet-v1-1/resolve/main/control_v11p_sd15_canny.pth -O /cn-models/control_v11p_sd15_canny.pth && \
    wget https://huggingface.co/stabilityai/stable-diffusion-xl-base-1.0/resolve/main/sd_xl_base_1.0.safetensors -O /comfy-models/sd_xl_base_1.0.safetensors && \
    wget https://huggingface.co/stabilityai/stable-diffusion-xl-refiner-1.0/resolve/main/sd_xl_refiner_1.0.safetensors -O /comfy-models/sd_xl_refiner_1.0.safetensors

# Create a virtual environment
RUN python -m venv /workspace/venv && \
    pip install -U --no-cache-dir jupyterlab jupyterlab_widgets ipykernel ipywidgets

# Install Automatic1111's WebUI
RUN git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui.git && \
    cd stable-diffusion-webui && \
    pip install -r requirements_versions.txt && \
    python -c "from launch import prepare_environment; prepare_environment()" --skip-torch-cuda-test

# Install torch and xformers
RUN cd /workspace/stable-diffusion-webui && \
    pip install torch===${TORCH_VERSION} torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118 && \
    pip install xformers==${XFORMERS_VERSION} --index-url https://download.pytorch.org/whl/cu118

# Install DeForum
RUN cd /workspace/stable-diffusion-webui && \
    git clone https://github.com/deforum-art/sd-webui-deforum extensions/deforum && \
    cd extensions/deforum && \
    pip install basicsr && \
    pip install -r requirements.txt

# Install ControlNet
RUN cd /workspace/stable-diffusion-webui && \
    git clone https://github.com/Mikubill/sd-webui-controlnet.git extensions/sd-webui-controlnet && \
    cd extensions/sd-webui-controlnet && \
    pip install -r requirements.txt

COPY cache-sd-model.py /workspace/stable-diffusion-webui/
RUN cd /workspace/stable-diffusion-webui/ && \
    python cache-sd-model.py --use-cpu=all --ckpt /sd-models/SDv1-5.ckpt

RUN mv /workspace/venv /venv && \
    mv /workspace/stable-diffusion-webui /stable-diffusion-webui && \
    mkdir /workspace/downloader && git clone https://github.com/jjangga0214/sd-models-downloader.git /workspace/downloader

COPY relauncher.py webui-user.sh /stable-diffusion-webui/

# Install ComfyUI and extensions
RUN git clone https://github.com/comfyanonymous/ComfyUI.git && \
    cd /ComfyUI && \
    pip install -r requirements.txt && \
    git clone https://github.com/ltdrdata/ComfyUI-Manager.git custom_nodes/ComfyUI-Manager && \
    cd custom_nodes/ComfyUI-Manager && \
    pip install -r requirements.txt

# Install AnimateDiff for ComfyUI
RUN cd /ComfyUI/custom_nodes && \
    git clone https://github.com/guoyww/animatediff.git && \
    cd animatediff && \
    pip install -r requirements.txt

# Install ReActor for ComfyUI
RUN cd /ComfyUI/custom_nodes && \
    git clone https://github.com/Gourieff/comfyui-reactor-node.git && \
    cd comfyui-reactor-node && \
    python install.py

# NGINX Proxy
COPY nginx.conf /etc/nginx/nginx.conf

# Start Scripts
COPY pre_start1.sh /pre_start1.sh
COPY pre_start2.sh /pre_start2.sh
COPY start.sh /
RUN chmod +x /start.sh && chmod +x /pre_start1.sh && chmod +x /pre_start2.sh

SHELL ["/bin/bash", "--login", "-c"]
CMD [ "/start.sh" ]