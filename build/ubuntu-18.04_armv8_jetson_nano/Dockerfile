# syntax = docker/dockerfile:1.1.1-experimental
FROM ubuntu:18.04

ARG PACKAGE_NAME

LABEL jp.shiguredo.momo=$PACKAGE_NAME

RUN rm -f /etc/apt/apt.conf.d/docker-clean; echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache

# パッケージのインストール

COPY script/apt_install_arm.sh /root/
RUN --mount=type=cache,id=$PACKAGE_NAME,target=/var/cache/apt --mount=type=cache,id=$PACKAGE_NAME,target=/var/lib/apt \
  /root/apt_install_arm.sh

# RootFS の構築

COPY script/init_rootfs_arm64.sh /root/
COPY arm64.conf /root/
RUN --mount=type=cache,id=$PACKAGE_NAME,target=/var/cache/apt --mount=type=cache,id=$PACKAGE_NAME,target=/var/lib/apt \
  /root/init_rootfs_arm64.sh /root/rootfs /root/arm64.conf

# 構築した RootFS を Jetson Nano のファイルで上書きする

COPY jetson.sh /root/
RUN /root/jetson.sh

# WebRTC の取得

ARG WEBRTC_BUILD_VERSION

COPY script/get_webrtc.sh /root/
RUN /root/get_webrtc.sh "$WEBRTC_BUILD_VERSION" ubuntu-18.04_armv8 /root

# コンパイラの取得

COPY script/get_llvm.sh /root/
RUN /root/get_llvm.sh /root/webrtc /root

# Boost のビルド

ARG BOOST_VERSION

COPY script/setup_boost.sh /root/
RUN \
  set -ex \
  && /root/setup_boost.sh "$BOOST_VERSION" /root/boost-source \
  && cd /root/boost-source/source \
  && echo 'using clang : : /root/llvm/clang/bin/clang++ : ;' > project-config.jam \
  && ./b2 \
    cxxflags=' \
      -D_LIBCPP_ABI_UNSTABLE \
      -nostdinc++ \
      -isystem/root/llvm/libcxx/include \
      --target=aarch64-linux-gnu \
      --sysroot=/root/rootfs \
      -I/root/rootfs/usr/include/aarch64-linux-gnu \
    ' \
    linkflags=' \
      -L/root/rootfs/usr/lib/aarch64-linux-gnu \
      -B/root/rootfs/usr/lib/aarch64-linux-gnu \
    ' \
    toolset=clang \
    visibility=global \
    target-os=linux \
    architecture=arm \
    address-model=64 \
    link=static \
    variant=release \
    install \
    -j`nproc` \
    --ignore-site-config \
    --prefix=/root/boost \
    --with-filesystem


# SDL2 のビルド

ARG SDL2_VERSION

COPY script/setup_sdl2.sh /root/
RUN \
  set -ex \
  && /root/setup_sdl2.sh "$SDL2_VERSION" /root/sdl2-source \
  && mkdir -p /root/sdl2-source/build \
  && cd /root/sdl2-source/build \
  && CC='/root/llvm/clang/bin/clang --target=aarch64-linux-gnu --sysroot=/root/rootfs' \
      CFLAGS=' \
        -I/root/rootfs/usr/include/aarch64-linux-gnu \
      ' \
      LDFLAGS=' \
        -L/root/rootfs/usr/lib/aarch64-linux-gnu \
        -B/root/rootfs/usr/lib/aarch64-linux-gnu \
      ' \
      ../source/configure --disable-shared --prefix=/root/sdl2 --host=aarch64-linux-gnu --with-sysroot=/root/rootfs --disable-pulseaudio --disable-esd --disable-video-wayland \
  && make \
  && make install

# nlohmann/json の取得

ARG JSON_VERSION
RUN git clone --branch v$JSON_VERSION --depth 1 https://github.com/nlohmann/json.git /root/json

# CLI11 の取得

ARG CLI11_VERSION
RUN git clone --branch v$CLI11_VERSION --depth 1 https://github.com/CLIUtils/CLI11.git /root/CLI11
