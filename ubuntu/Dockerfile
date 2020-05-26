#
# Builds a custom docker image for ShinobiCCTV Pro
#

# FFMPEG GPU專用
##############################################################################################################
FROM nvidia/cuda:10.0-devel-ubuntu18.04 as builder

ENV NASM_VERSION 2.14
ENV NVCODEC_VERSION 8.2.15.6
ENV FFMPEG_VERSION 4.1

RUN apt-get update && apt-get install -y autoconf curl git pkg-config

RUN curl -fsSLO https://www.nasm.us/pub/nasm/releasebuilds/$NASM_VERSION/nasm-$NASM_VERSION.tar.bz2 \
  && tar -xjf nasm-$NASM_VERSION.tar.bz2 \
  && cd nasm-$NASM_VERSION \
  && ./autogen.sh \
  && ./configure \
  && make -j$(nproc) \
  && make install

RUN git clone -b n$NVCODEC_VERSION --depth 1 https://git.videolan.org/git/ffmpeg/nv-codec-headers \
  && cd nv-codec-headers \
  && make install

ENV PKG_CONFIG_PATH /usr/local/lib/pkgconfig
RUN curl -fsSLO https://ffmpeg.org/releases/ffmpeg-$FFMPEG_VERSION.tar.bz2 \
  && tar -xjf ffmpeg-$FFMPEG_VERSION.tar.bz2 \
  && cd ffmpeg-$FFMPEG_VERSION \
  && ./configure --enable-nvenc \
  && make -j$(nproc) \
  && make install

##############################################################################################################

FROM nvidia/cuda:10.0-cudnn7-devel

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

# FFMPEG GPU
COPY --from=builder /usr/local/bin/ffmpeg /usr/local/bin/ffmpeg
COPY --from=builder /usr/local/bin/ffprobe /usr/local/bin/ffprobe

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
