FROM ghcr.io/openwrt/sdk:x86_64-v25.12.2

COPY hapt ./package/hapt
CMD make defconfig && make package/hapt/compile V=sc
