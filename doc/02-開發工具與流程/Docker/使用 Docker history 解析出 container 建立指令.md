
你可以使用 `docker history` 命令來分析映像層的變更，然後按正確順序（反轉 `docker history` 的輸出順序）來建構 `Dockerfile`。以下是一種方法：

### 1. 取得 `docker history` 資訊：

使用以下命令顯示完整 `history` 記錄：

```sh
docker history <image_id> --format "{{.CreatedBy}}" --no-trunc
```

這將會列出所有 `CreatedBy` 命令，但順序是從最近的變更到最早的變更，需要反轉順序才能用於 `Dockerfile`，需要加上`--no-trunc` ，否則太長指令會被截掉而得不到完整指令。

### 2. 根據 history 生成 `Dockerfile`

你可以手動處理這些資訊，並按照正確的順序（從最早的指令開始）組織成 `Dockerfile`。例如：

```Dockerfile
ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y wget openjdk-8-jdk git ccache automake \
   lzop bison gperf build-essential zip curl \
   zlib1g-dev g++-multilib python3-networkx \
   libxml2-utils bzip2 libbz2-dev libbz2-1.0 \
   libghc-bzlib-dev squashfs-tools pngcrush \
   schedtool dpkg-dev liblz4-tool make optipng maven \
   libssl-dev bc bsdmainutils gettext python3-mako \
   libelf-dev sbsigntool dosfstools mtools efitools \
   python3-pystache git-lfs python-is-python3 flex clang libncurses5 \
   fakeroot ncurses-dev xz-utils cryptsetup-bin \
   apt-transport-https ca-certificates curl lsb-release \
   rsync vim python-six kmod glslang-tools \
   software-properties-common cpio python3-pip ninja-build \
   cutils cmake pkg-config xorriso mtools libjson-c-dev file

RUN wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB  \
   | gpg --dearmor | tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null && \
   echo "deb [signed-by=/usr/share/keyrings/oneapi-archive-keyring.gpg] https://apt.repos.intel.com/oneapi all main" \
   | tee /etc/apt/sources.list.d/oneAPI.list

RUN apt-get update && \
   apt-get install -y intel-oneapi-ipp-devel-2021.10 \
   intel-oneapi-mkl-devel-2021.1.1

RUN apt-get update && apt-get install sudo && \
    useradd --create-home -d ${HOME} --shell /bin/bash --user-group --groups adm,sudo ${MYNAME} && \
    echo "${MYNAME}:${MYPASS}" | chpasswd && \
    echo "${MYNAME} ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/${MYNAME} && \
    chmod 0440 /etc/sudoers.d/${MYNAME}

RUN pip3 install meson==0.60.0 mako==1.1.0 dataclasses pycryptodome ply==3.11

RUN curl -o /usr/local/bin/repo https://storage.googleapis.com/git-repo-downloads/repo \
 && chmod a+x /usr/local/bin/repo

USER ${MYNAME}

RUN unset DEBIAN_FRONTEND

```

### 3. 自動化處理（使用 Shell 指令）

如果你想要自動反轉 `docker history` 並生成 `Dockerfile`，可以嘗試使用 `tac` 來達到反向輸出的效果：

```sh
docker history <image_id> --format "{{.CreatedBy}}" --no-trunc | tac > docker_commands.txt
```

然後，你可以使用腳本將 `docker_commands.txt` 轉換為 `Dockerfile` 格式。

```docker_commands.txt
/bin/sh -c #(nop)  ARG RELEASE
/bin/sh -c #(nop)  ARG LAUNCHPAD_BUILD_ARCH
/bin/sh -c #(nop)  LABEL org.opencontainers.image.ref.name=ubuntu
/bin/sh -c #(nop)  LABEL org.opencontainers.image.version=22.04
/bin/sh -c #(nop) ADD file:59e67123ba6a5d9eea9813e7b2a767696f767c15c5b23c61c4d5bd6ba6fa9ac6 in / 
/bin/sh -c #(nop)  CMD ["/bin/bash"]
ARG MYNAME=arbor
ARG MYPASS=arbor
ARG HOME=/home/arbor
ENV DEBIAN_FRONTEND=noninteractive
RUN |3 MYNAME=arbor MYPASS=arbor HOME=/home/arbor /bin/sh -c apt-get update && apt-get install -y wget openjdk-8-jdk git ccache automake    lzop bison gperf build-essential zip curl    zlib1g-dev g++-multilib python3-networkx    libxml2-utils bzip2 libbz2-dev libbz2-1.0    libghc-bzlib-dev squashfs-tools pngcrush    schedtool dpkg-dev liblz4-tool make optipng maven    libssl-dev bc bsdmainutils gettext python3-mako    libelf-dev sbsigntool dosfstools mtools efitools    python3-pystache git-lfs python-is-python3 flex clang libncurses5    fakeroot ncurses-dev xz-utils cryptsetup-bin    apt-transport-https ca-certificates curl lsb-release    rsync vim python-six kmod glslang-tools    software-properties-common cpio python3-pip ninja-build    cutils cmake pkg-config xorriso mtools libjson-c-dev file # buildkit
RUN |3 MYNAME=arbor MYPASS=arbor HOME=/home/arbor /bin/sh -c wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB     | gpg --dearmor | tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null &&    echo "deb [signed-by=/usr/share/keyrings/oneapi-archive-keyring.gpg] https://apt.repos.intel.com/oneapi all main"    | tee /etc/apt/sources.list.d/oneAPI.list # buildkit
RUN |3 MYNAME=arbor MYPASS=arbor HOME=/home/arbor /bin/sh -c apt-get update &&    apt-get install -y intel-oneapi-ipp-devel-2021.10    intel-oneapi-mkl-devel-2021.1.1 # buildkit
RUN |3 MYNAME=arbor MYPASS=arbor HOME=/home/arbor /bin/sh -c apt-get update && apt-get install sudo &&     useradd --create-home -d ${HOME} --shell /bin/bash --user-group --groups adm,sudo ${MYNAME} &&     echo "${MYNAME}:${MYPASS}" | chpasswd &&     echo "${MYNAME} ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/${MYNAME} &&     chmod 0440 /etc/sudoers.d/${MYNAME} # buildkit
RUN |3 MYNAME=arbor MYPASS=arbor HOME=/home/arbor /bin/sh -c pip3 install meson==0.60.0 mako==1.1.0 dataclasses pycryptodome ply==3.11 # buildkit
RUN |3 MYNAME=arbor MYPASS=arbor HOME=/home/arbor /bin/sh -c curl -o /usr/local/bin/repo https://storage.googleapis.com/git-repo-downloads/repo  && chmod a+x /usr/local/bin/repo # buildkit
USER arbor
RUN |3 MYNAME=arbor MYPASS=arbor HOME=/home/arbor /bin/sh -c unset DEBIAN_FRONTEND # buildkit
/bin/bash
```

這樣，你可以更輕鬆地從 `docker history` 來構建出完整的 `Dockerfile`！希望這有幫助 🚀