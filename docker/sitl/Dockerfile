FROM ubuntu:24.04

# ── Environment ────────────────────────────────────────────────────────────────
ENV DEBIAN_FRONTEND=noninteractive \
    TZ=UTC \
    LANG=C.UTF-8 \
    # Explicitly set USER and HOME so scripts that rely on them (e.g. usermod $USER)
    # work inside Docker RUN steps — Docker does NOT populate these from USER directive
    USER=ardupilot \
    HOME=/home \
    # Tell the install script to skip the STM32 bare-metal toolchain (not needed for SITL)
    DO_AP_STM_ENV=0 \
    # Skip git submodule update inside the script (we handle it in the clone step)
    SKIP_AP_GIT_CHECK=1 \
    # Virtual display — Gazebo/Qt GUIs will render to Xvfb on :1
    DISPLAY=:1

# ── System bootstrap (root) ────────────────────────────────────────────────────
RUN apt-get update && apt-get install -y --no-install-recommends \
        sudo \
        git \
        curl \
        lsb-release \
        ca-certificates \
        tzdata \
        # Virtual framebuffer — provides a fake display so Qt/Gazebo don't crash
        xvfb \
        x11vnc \
        openbox \
        # Software OpenGL renderer — GPU-less 3D for Gazebo inside Docker
        # libgl1-mesa-glx \
        libgl1-mesa-dri \
    && rm -rf /var/lib/apt/lists/*

# ── Non-root user with passwordless sudo ───────────────────────────────────────
ARG USERNAME=ardupilot
ARG USER_UID=1000
ARG USER_GID=${USER_UID}

RUN apt-get update && apt-get install -y sudo \
    # 1. Remove the default 'ubuntu' user to free up UID/GID 1000
    && touch /var/mail/ubuntu && chown ubuntu /var/mail/ubuntu \
    && userdel -r ubuntu || true \
    && groupdel ubuntu || true \
    # 2. Create the ardupilot user
    && groupadd --gid $USER_GID $USERNAME \
    && useradd --uid $USER_UID --gid $USER_GID -m -s /bin/bash $USERNAME \
    && usermod -a -G dialout $USERNAME \
    # 3. Setup passwordless sudo
    && echo "$USERNAME ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME \
    && rm -rf /var/lib/apt/lists/*

USER $USERNAME
ENV USER=$USERNAME \
    HOME=/home/$USERNAME
    
WORKDIR ${HOME}

# ── Clone ArduPilot ────────────────────────────────────────────────────────────
ARG ARDUPILOT_BRANCH=master
RUN git clone \
        --recurse-submodules \
        --depth 1 \
        --branch ${ARDUPILOT_BRANCH} \
        https://github.com/ArduPilot/ardupilot.git \
        /home/ardupilot/ardupilot

WORKDIR ${HOME}/ardupilot

# ── Run the official prereqs script ───────────────────────────────────────────
# bash is explicit so bash-specific syntax in the script (;;& fallthrough,
# [[ ]], etc.) doesn't break under /bin/sh.d
# USER and HOME are already set in ENV above — no export needed at runtime.
RUN bash Tools/environment_install/install-prereqs-ubuntu.sh -y

ENV PATH="/home/ardupilot/ardupilot/Tools/autotest:/home/ardupilot/venv-ardupilot/bin:${PATH}"

# ── Activate Python venv on every shell ───────────────────────────────────────
# The script creates the venv at ~/venv-ardupilot on Ubuntu 24.04 (noble).
RUN echo "source /home/ardupilot/venv-ardupilot/bin/activate" \
        >> /home/ardupilot/.bashrc \
    && echo "source /home/ardupilot/ardupilot/Tools/completion/completion.bash" \
        >> /home/ardupilot/.bashrc \
    && echo "cd /home/ardupilot/ardupilot" \
        >> /home/ardupilot/.bashrc \
    # Start a virtual framebuffer automatically on login so Gazebo/MAVProxy GUI
    # can open without needing a real display or X11 forwarding from the host.
    && echo 'Xvfb :1 -screen 0 1920x1080x24 &' \
        >> /home/ardupilot/.bashrc

# Install Gazebo harmonic

USER root

RUN apt-get update -y && \ 
    apt-get install curl lsb-release gnupg -y && \
    curl https://packages.osrfoundation.org/gazebo.gpg --output /usr/share/keyrings/pkgs-osrf-archive-keyring.gpg && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/pkgs-osrf-archive-keyring.gpg] https://packages.osrfoundation.org/gazebo/ubuntu-stable $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/gazebo-stable.list > /dev/null && \
    apt-get update -y && \
    apt-get install gz-harmonic -y

# ── Build Dependencies (Switch to root) ───────────────────────────────────────
USER root

# 1. Update and install dependencies for Harmonic (Version 8)
# Note: libgz-sim8-dev is the correct header for Harmonic
RUN apt-get update && apt-get install -y --no-install-recommends \
    libgz-sim8-dev \
    rapidjson-dev \
    libopencv-dev \
    libgstreamer1.0-dev \
    libgstreamer-plugins-base1.0-dev \
    gstreamer1.0-plugins-bad \
    gstreamer1.0-libav \
    gstreamer1.0-gl \
    && rm -rf /var/lib/apt/lists/*

# ── Plugin Configuration (Switch back to user) ────────────────────────────────
USER ${USERNAME}
WORKDIR /home/${USERNAME}

# Environment variables for Gazebo Harmonic
ENV GZ_VERSION=harmonic \
    GZ_SIM_SYSTEM_PLUGIN_PATH=/home/${USERNAME}/ardupilot_gazebo/build:${GZ_SIM_SYSTEM_PLUGIN_PATH} \
    GZ_SIM_RESOURCE_PATH=/home/${USERNAME}/ardupilot_gazebo/models:/home/${USERNAME}/ardupilot_gazebo/worlds:${GZ_SIM_RESOURCE_PATH}

# Clone repository and build 
RUN git clone --depth 1 https://github.com/ArduPilot/ardupilot_gazebo

WORKDIR ${HOME}/ardupilot_gazebo

RUN mkdir build && \
    cd build && \
    cmake .. -DCMAKE_BUILD_TYPE=RelWithDebInfo && \
    make -j4

ENV GZ_SIM_SYSTEM_PLUGIN_PATH=$HOME/ardupilot_gazebo/build:$GZ_SIM_SYSTEM_PLUGIN_PATH
ENV GZ_SIM_RESOURCE_PATH=$HOME/ardupilot_gazebo/models:$HOME/ardupilot_gazebo/worlds:$GZ_SIM_RESOURCE_PATH

WORKDIR ${HOME}

RUN git clone --depth 1 https://github.com/ArduPilot/SITL_Models.git

ENV GZ_SIM_RESOURCE_PATH="${GZ_SIM_RESOURCE_PATH}:${HOME}/SITL_Models/Gazebo/models:${HOME}/SITL_Models/Gazebo/worlds"

# Install QGroundControl dependencies
USER root

# RUN usermod -a -G dialout $USER && \
#     apt-get remove modemmanager -y && \
#     apt install gstreamer1.0-plugins-bad gstreamer1.0-libav gstreamer1.0-gl -y && \
#     apt install libfuse2 -y && \
#     apt install libxcb-xinerama0 libxkbcommon-x11-0 libxcb-cursor-dev -y

RUN wget https://d176tv9ibo4jno.cloudfront.net/latest/QGroundControl-x86_64.AppImage

ENV LIBGL_ALWAYS_SOFTWARE=1

# Move the AppImage to a stable library directory (like /opt)
RUN mkdir -p /opt/qgc && \
    mv QGroundControl-x86_64.AppImage /opt/qgc/qgc.AppImage && \
    chmod +x /opt/qgc/qgc.AppImage

# Create the wrapper script in /usr/local/bin
RUN echo '#!/bin/bash\n/opt/qgc/qgc.AppImage --appimage-extract-and-run "$@"' > /usr/local/bin/qgroundcontrol && \
    chmod +x /usr/local/bin/qgroundcontrol

USER ${USERNAME}

# ── MAVLink / GCS ports ────────────────────────────────────────────────────────
EXPOSE 5760 5761 14550 14551

# ── Default shell ──────────────────────────────────────────────────────────────
SHELL ["/bin/bash", "-l", "-c"]
CMD ["bash"]