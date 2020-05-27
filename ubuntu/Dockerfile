#
# Builds a custom docker image for ShinobiCCTV Pro
#

# FFMPEG GPU專用 
# https://gitlab.com/nvidia/container-images/samples/-/blob/master/cuda/ubuntu16.04/ffmpeg-gpu/Dockerfile
# 
##############################################################################################################
FROM nvidia/cuda:9.0-devel as ffmpeg

RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates \
        git \
        libgl1-mesa-glx \
        make \
        nasm && \
    rm -rf /var/lib/apt/lists/*

RUN git clone --depth 1 --branch n3.4.1 https://github.com/ffmpeg/ffmpeg ffmpeg && \
    cd ffmpeg && \
    ./configure --enable-cuda --enable-cuvid --enable-nvenc --enable-nonfree --enable-libnpp \
                # 修復 ERROR: libx264 not found
                --enable-shared \
                # 修復 ffmpeg: Unrecognized option 'crf'. Error splitting the argument list: Option not found
                --enable-gpl \
                --extra-cflags=-I/usr/local/cuda/include \
                --extra-ldflags=-L/usr/local/cuda/lib64 \
                --prefix=/usr/local/ffmpeg --enable-shared --disable-static \
                --disable-manpages --disable-doc --disable-podpages && \
                make -j"$(nproc)" install && \
                ldconfig

##############################################################################################################

FROM nvidia/cuda:9.0-base

COPY --from=ffmpeg /usr/local/ffmpeg /usr/local

ENV NVIDIA_DRIVER_CAPABILITIES $NVIDIA_DRIVER_CAPABILITIES,video

LABEL Author="MiGoller"
# 安裝前置套件
RUN apt-get update && apt-get install -y curl sudo

# 安裝Node8 
#
######################################################################################
RUN curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash - && \
    apt-get install -y nodejs

# Set environment variables to default values
ENV ADMIN_USER=m0724001@gm.nuu.edu.tw \
    ADMIN_PASSWORD=123456 \
    CRON_KEY=fd6c7849-904d-47ea-922b-5143358ba0de \
    PLUGINKEY_MOTION=b7502fd9-506c-4dda-9b56-8e699a6bc41c \
    PLUGINKEY_OPENCV=f078bcfe-c39a-4eb5-bd52-9382ca828e8a \
    PLUGINKEY_OPENALPR=dbff574e-9d4a-44c1-b578-3dc0f1944a3c \
    MOTION_HOST=localhost \ 
    MOTION_PORT=8080 

# Create the custom configuration dir
RUN mkdir -p /config

# Create the working dir
RUN mkdir -p /opt/shinobi

WORKDIR /opt/shinobi

# Install package dependencies
RUN apt-get update \
    && apt-get install -y python pkg-config libcairo-dev make g++ libjpeg-dev git mysql-client \
    && apt-get clean

# GPU FFMPEG
RUN apt-get update && apt-get install -y --no-install-recommends \
        cuda-npp-9-0 && \
    rm -rf /var/lib/apt/lists/*

# Clone the Shinobi CCTV PRO repo
RUN mkdir master_temp
RUN git clone https://gitlab.com/Shinobi-Systems/Shinobi.git master_temp
RUN cp -R -f master_temp/* .
RUN rm -rf $distro master_temp

# Install NodeJS dependencies
RUN npm install pm2 -g

RUN npm install && \
    npm install canvas@1.6.5 moment --unsafe-perm

# Copy code
COPY docker-entrypoint.sh .
COPY pm2Shinobi.yml .
RUN chmod -f +x ./*.sh

# Copy default configuration files
COPY ./config/conf.sample.json /opt/shinobi/conf.sample.json
COPY ./config/super.sample.json /opt/shinobi/super.sample.json
COPY ./config/motion.conf.sample.json /opt/shinobi/plugins/motion/conf.sample.json

VOLUME ["/opt/shinobi/videos"]
VOLUME ["/config"]

EXPOSE 8080

# Set the user to use when running this image
# See docker-entrypoint.sh on how to change the uid/gid of the user.
#USER node

ENTRYPOINT ["/opt/shinobi/docker-entrypoint.sh"]

CMD ["pm2-docker", "pm2Shinobi.yml"]
