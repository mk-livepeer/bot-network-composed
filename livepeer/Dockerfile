# build stage
FROM nvidia/cuda:10.1-devel-ubuntu18.04 as builder

RUN export DEBIAN_FRONTEND=noninteractive && \
    apt-get update && apt-get dist-upgrade -y && \
    apt-get install software-properties-common -y && \
    add-apt-repository ppa:longsleep/golang-backports && \
    apt-get update && apt-get dist-upgrade -y && \
    apt-get install -y build-essential pkg-config autoconf gnutls-dev \
            git curl golang-go && \
    curl -O http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/cuda-repo-ubuntu1804_10.0.130-1_amd64.deb && \
    dpkg --force-confdef --force-confnew -i ./cuda-repo-ubuntu1804_10.0.130-1_amd64.deb && \
    apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/7fa2af80.pub && \
    apt-get update && \
    apt-get install cuda-10-0 -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

RUN git clone -b nasm-2.14.02 https://repo.or.cz/nasm.git "$HOME/nasm" && \
    cd "$HOME/nasm" && \
    ./autogen.sh && \
    ./configure && \
    make && \
    make install || echo "Installing docs fails but should be OK otherwise"

RUN git clone http://git.videolan.org/git/x264.git "$HOME/x264" && \
    cd "$HOME/x264" && \
    git checkout 545de2ffec6ae9a80738de1b2c8cf820249a2530 && \
    ./configure --enable-pic --enable-static --disable-cli && \
    make && \
    make install-lib-static

RUN git clone https://git.ffmpeg.org/ffmpeg.git "$HOME/ffmpeg"

RUN git clone --single-branch https://github.com/FFmpeg/nv-codec-headers "$HOME/nv-codec-headers" && \
    cd "$HOME/nv-codec-headers" && \
    git checkout 9fdaf11b8f79d4e41cde9af89656238f25fec6fd && \
    make install && \
    cd .. && \
    rm -rf nv-codec-headers

RUN export PATH=/usr/local/cuda/bin:$PATH && \
    cd "$HOME/ffmpeg" && \
    git checkout 4cfc34d9a8bffe4a1dd53187a3e0f25f34023a09 && \
    ./configure --disable-static --enable-shared \
        --enable-gpl --enable-nonfree --enable-libx264 --enable-cuda --enable-cuvid \
        --enable-nvenc --enable-cuda-nvcc --enable-libnpp --enable-gnutls \
        --extra-ldflags=-L/usr/local/cuda/lib64 \
        --extra-cflags='-pg -I/usr/local/cuda/include' --disable-stripping

RUN export PATH=$HOME/compiled/bin:$PATH && \
    export PATH=/usr/local/cuda/bin:$PATH && \
    export PKG_CONFIG_PATH=$HOME/compiled/lib/pkgconfig && \
    cd "$HOME/ffmpeg" && \
    make && make install

RUN export PATH=$HOME/compiled/bin:$PATH && \
    export PATH=/usr/local/cuda/bin:$PATH && \
    export PKG_CONFIG_PATH=$HOME/compiled/lib/pkgconfig && \
    git clone https://github.com/livepeer/go-livepeer "$HOME/go-livepeer" && \
    cd "$HOME/go-livepeer" && \
    go mod download && \
    go build ./cmd/livepeer/livepeer.go && \
    go build ./cmd/devtool/devtool.go

ENV LD_LIBRARY_PATH /usr/local/lib/

# final stage
FROM nvidia/cuda:10.1-devel-ubuntu18.04

RUN apt-get update && apt-get dist-upgrade -y && \
    apt-get install software-properties-common -y && \
    add-apt-repository ppa:longsleep/golang-backports && \
    apt-get update && apt-get dist-upgrade -y && \
    apt-get install -y build-essential pkg-config autoconf gnutls-dev \
            git curl golang-go && \
    curl -O http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/cuda-repo-ubuntu1804_10.0.130-1_amd64.deb && \
    dpkg --force-confdef --force-confnew -i ./cuda-repo-ubuntu1804_10.0.130-1_amd64.deb && \
    apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/7fa2af80.pub && \
    apt-get update && \
    apt-get install cuda-10-0 -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# everything above this line has already been cached in the build stage -
# we save time and disk space by keeping the common parts of each stage
# identical at the start of each image build

# this is needed to access GPU inside Docker Swarm
ENV NVIDIA_DRIVER_CAPABILITIES=all

COPY --from=builder /usr/local/lib/* /usr/local/lib/
COPY --from=builder /root/go-livepeer/devtool /usr/local/bin/
COPY --from=builder /root/go-livepeer/livepeer /usr/local/bin/

RUN ln -s /usr/local/bin/livepeer /

ENV LD_LIBRARY_PATH /usr/local/lib/

#ENTRYPOINT [ "/root/go/bin/livepeer" ]
