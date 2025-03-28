# Base
FROM debian:12-slim AS base

ENV DEBIAN_FRONTEND "noninteractive"
ENV EXTERNAL_CONFIG 0

SHELL [ "/usr/bin/bash", "-c" ]

RUN echo "Installing packages" \
    && dpkg --add-architecture i386 \
    && apt-get update \
    && apt-get install --no-install-recommends -y \
        procps \
        ca-certificates \
        winbind \
        dbus \
        libfreetype6 \
        curl \
        jq \
        locales \
        lib32gcc-s1

RUN echo "Configuring locales" \
    && echo 'LANG="en_US.UTF-8"' > /etc/default/locale \
    && echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen \
    && locale-gen

RUN echo "Settings machine id" \
    && rm -f /etc/machine-id \
    && dbus-uuidgen --ensure=/etc/machine-id

RUN echo "Cleaning up" \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean \
    && apt-get autoremove -y 



# Steam
FROM base AS steam

ARG CONTAINER_GID=10000
ARG CONTAINER_UID=10000

ENV STEAM_HOME "/home/steam"
ENV STEAMCMD_PATH="${STEAM_HOME}/steamcmd"
ENV STEAM_SDK64_PATH="${STEAM_HOME}/.steam/sdk64"
ENV STEAM_SDK32_PATH="${STEAM_HOME}/.steam/sdk32"
ENV STEAM_COMPAT_CLIENT_INSTALL_PATH "$STEAMCMD_PATH"
ENV STEAM_APP_ID "2278520"
ENV STEAM_COMPAT_DATA_PATH "${STEAMCMD_PATH}/steamapps/compatdata/${STEAM_APP_ID}"
ENV STEAM_COMPAT_TOOLS_PATH "${STEAMCMD_PATH}/compatibilitytools.d"

ENV UMU_ID 0

# GE_PROTON_VERSION MUST be in this stage! It is referenced not only by `proton` stage
# but also in entrypoint.sh
ENV GE_PROTON_VERSION "9-26"
ENV PROTON_UNPACKED "${STEAM_HOME}/proton"

RUN echo "Creating user steam" \
    && groupadd --gid $CONTAINER_GID steam \
    && useradd --gid $CONTAINER_GID --uid $CONTAINER_UID --create-home --home-dir $STEAM_HOME steam

WORKDIR ${STEAM_HOME}
USER steam



# Steam
FROM steam AS enshrouded

ADD --chown=$CONTAINER_GID:$CONTAINER_GID \
    --checksum=sha256:cebf0046bfd08cf45da6bc094ae47aa39ebf4155e5ede41373b579b8f1071e7c \
    https://steamcdn-a.akamaihd.net/client/installer/steamcmd_linux.tar.gz /tmp/steam-cmd.tar.gz

RUN echo "Loading SteamCMD" \
    && mkdir -p "${STEAM_COMPAT_DATA_PATH}" \
    && tar zxvf "/tmp/steam-cmd.tar.gz" -C ${STEAMCMD_PATH} \
    && chmod +x ${STEAMCMD_PATH}/steamcmd.sh \
    && ${STEAMCMD_PATH}/steamcmd.sh +quit \
    && mkdir -p "${STEAM_HOME}/.steam" \
    && mkdir -p "${STEAM_COMPAT_TOOLS_PATH}"

RUN echo "Configuring Steam SDK" \
    && ln -s ${STEAMCMD_PATH}/linux64 ${STEAM_SDK64_PATH} \
    && ln -s ${STEAM_SDK64_PATH}/steamclient.so ${STEAM_SDK64_PATH}/steamservice.so \
    && ln -s ${STEAMCMD_PATH}/linux32 ${STEAM_SDK32_PATH} \
    && ln -s ${STEAM_SDK32_PATH}/steamclient.so ${STEAM_SDK32_PATH}/steamservice.so



# Proton
FROM steam AS proton

ENV GE_PROTON_URL "https://github.com/GloriousEggroll/proton-ge-custom/releases/download/GE-Proton${GE_PROTON_VERSION}/GE-Proton${GE_PROTON_VERSION}.tar.gz"
ENV GE_PROTON_SHA "sha256:f299a23a91e0096e1a438ef10304318ecb960cdc2c4204a33b725ddb4737dbdd"

ADD --chown=$CONTAINER_GID:$CONTAINER_GID \
    --checksum=$GE_PROTON_SHA \
    $GE_PROTON_URL /tmp/proton.tar.gz

RUN echo "Unpacking Proton" \
    && mkdir -p "${PROTON_UNPACKED}" \
    && tar zxf /tmp/proton.tar.gz -C "${PROTON_UNPACKED}"



# Final
FROM steam AS final

ENV ENSHROUDED_PATH "${STEAM_HOME}/enshrouded"
ENV ENSHROUDED_CONFIG "${ENSHROUDED_PATH}/enshrouded_server.json"

COPY --from=enshrouded ${STEAM_HOME} ${STEAM_HOME}
COPY --from=proton ${PROTON_UNPACKED} ${STEAM_COMPAT_TOOLS_PATH}

COPY --chown=$CONTAINER_GID:$CONTAINER_GID entrypoint.sh /home/steam/entrypoint.sh
COPY --chown=$CONTAINER_GID:$CONTAINER_GID enshrouded_server_example.json /home/steam/enshrouded_server_example.json


RUN ls -lR /home/steam

CMD ["/home/steam/entrypoint.sh"]
