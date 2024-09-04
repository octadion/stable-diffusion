# Base image setup
FROM runpod/stable-diffusion:models-1.0.0 as sd-models
FROM runpod/stable-diffusion-models:2.1 as hf-cache
FROM nvidia/cuda:11.8.0-base-ubuntu22.04 as runtime

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Set working directory and environment variables
ENV SHELL=/bin/bash
ENV PYTHONUNBUFFERED=1
ENV DEBIAN_FRONTEND=noninteractive
WORKDIR /

# Set up system dependencies
RUN apt-get update --yes && \
    apt-get upgrade --yes && \
    apt install --yes --no-install-recommends git wget curl bash libgl1 software-properties-common openssh-server nginx rsync && \
    add-apt-repository ppa:deadsnakes/ppa && \
    apt install python3.10-dev python3.10-venv -y --no-install-recommends && \
    apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* && \
    echo "en_US.UTF-8 UTF-8" > /etc/locale.gen

# Set up Python and pip
RUN ln -s /usr/bin/python3.10 /usr/bin/python && \
    rm /usr/bin/python3 && \
    ln -s /usr/bin/python3.10 /usr/bin/python3 && \
    curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py && \
    python get-pip.py

RUN python -m venv /venv
ENV PATH="/venv/bin:$PATH"

# Install necessary Python packages
RUN pip install --upgrade --no-cache-dir pip && \
    pip install --upgrade setuptools && \
    pip install --upgrade wheel
RUN pip install --upgrade --no-cache-dir torch==2.0.1+cu118 torchvision==0.15.2+cu118 torchaudio==2.0.2 --index-url https://download.pytorch.org/whl/cu118
RUN pip install --upgrade --no-cache-dir jupyterlab ipywidgets jupyter-archive jupyter_contrib_nbextensions triton xformers==0.0.22 gdown

# Set up Jupyter Notebook
RUN pip install notebook==6.5.5
RUN jupyter contrib nbextension install --user && \
    jupyter nbextension enable --py widgetsnbextension

# Install Stable Diffusion WebUI
RUN git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui.git /stable-diffusion-webui && \
    cd /stable-diffusion-webui && \
    pip install -r requirements_versions.txt && \
    pip install -r requirements.txt

# Include necessary scripts for WebUI
COPY webui-user.sh /stable-diffusion-webui/webui-user.sh
COPY relauncher.py /stable-diffusion-webui/relauncher.py
COPY cache-sd-model.py /stable-diffusion-webui/cache-sd-model.py

# Install ComfyUI and ComfyUI Manager
RUN git clone https://github.com/comfyanonymous/ComfyUI.git /ComfyUI && \
    cd /ComfyUI && \
    pip install -r requirements.txt && \
    git clone https://github.com/ltdrdata/ComfyUI-Manager.git custom_nodes/ComfyUI-Manager && \
    cd custom_nodes/ComfyUI-Manager && \
    pip install -r requirements.txt

# Install Animatediff and Reactor for ComfyUI
RUN cd /ComfyUI && \
    git clone https://github.com/Kosinkadink/ComfyUI-AnimateDiff-Evolved.git custom_nodes/ComfyUI-AnimateDiff-Evolved && \
    git clone https://github.com/Gourieff/comfyui-reactor-node.git custom_nodes/comfyui-reactor-node && \
    cd custom_nodes/ComfyUI-AnimateDiff-Evolved && \
    pip install -r requirements.txt && \
    cd ../comfyui-reactor-node && \
    pip install -r requirements.txt

# Create necessary directories and copy necessary files
RUN set -e && mkdir -p /root/.cache/huggingface && mkdir /comfy-models
COPY --from=hf-cache /root/.cache/huggingface /root/.cache/huggingface
COPY --from=sd-models /SDv1-5.ckpt /comfy-models/v1-5-pruned-emaonly.ckpt
COPY --from=sd-models /SDv2-768.ckpt /comfy-models/SDv2-768.ckpt
RUN wget https://huggingface.co/stabilityai/stable-diffusion-xl-base-1.0/resolve/main/sd_xl_base_1.0.safetensors -O /comfy-models/sd_xl_base_1.0.safetensors && \
    wget https://huggingface.co/stabilityai/stable-diffusion-xl-refiner-1.0/resolve/main/sd_xl_refiner_1.0.safetensors -O /comfy-models/sd_xl_refiner_1.0.safetensors

# NGINX Proxy
COPY nginx.conf /etc/nginx/nginx.conf

# Start Scripts
COPY pre_start1.sh /pre_start1.sh
COPY pre_start2.sh /pre_start2.sh
COPY start.sh /
RUN chmod +x /pre_start1.sh /pre_start2.sh /start.sh

CMD [ "/start.sh" ]
