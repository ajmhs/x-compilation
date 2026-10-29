# Production Dockerfile for Visual Studio Multi-Arch C++ Build Container

# (c) 2026 Copyright, Real-Time Innovations, Inc.  All rights reserved.
# RTI grants Licensee a license to use, modify, compile, and create derivative
# works of the Software.  Licensee has the right to distribute object form only
# for use with RTI products.  The Software is provided "as is", with no warranty
# of any type, including any warranty for fitness for any purpose. RTI is under
# no obligation to maintain or support the Software.  RTI shall not be liable
# for any incidental or consequential damages arising out of the use or
# inability to use the software.

# The Dockerfile is based on the official Ubuntu image for the jammy release. 
# The CONNEXT_VERSION argument is used to specify the version of RTI Connext to install. 
# The RTI_LICENSE_AGREEMENT_ACCEPTED argument is used to accept the RTI license agreement during installation.
# The NDDSHOME environment variable is set to the installation directory of RTI Connext.
##
# To build the local development image from the root repository folder, provide a unique SSH password:
#   docker build -t connext:buildhost --build-arg RTI_LICENSE_AGREEMENT_ACCEPTED=accepted --build-arg CONNEXT_VERSION=7.3.0 --build-arg SSHPASSWORD=<unique-password> .
#
# To run the Docker container, run the following command:
#   docker run -it --rm -p 127.0.0.1:23:23 -v $RTI_LICENSE_FILE:/root/rti_license.dat connext:buildhost
#
#   The -it option is used to run the container in interactive mode.
#   The --rm option is used to remove the container when it exits.
#   The loopback port mapping makes SSH accessible only from the host running Docker.
#   Do not distribute this image: build arguments may be retained in image metadata or build cache.
#
# The container will start and run the sshd service interactively - to interact with the container you'll have to use ssh on port 23.

FROM ubuntu:jammy AS connext-build

ARG CONNEXT_VERSION=7.3.0
ARG RTI_LICENSE_AGREEMENT_ACCEPTED
ARG SSHUSERNAME=developer
ARG SSHPASSWORD

ENV DISPLAY=:0
ENV SHELL=/bin/bash
# CHANGE THE FOLLOWING LINE TO MATCH YOUR TIMEZONE
ENV TZ=Europe/Madrid
ENV NDDSHOME=/opt/rti_connext_dds-${CONNEXT_VERSION}

# Install the required packages
RUN export DEBIAN_FRONTEND=noninteractive \
    && apt-get update \
    && apt-get install -y \
    ca-certificates \
    dash \
    tzdata \
    git \
    build-essential \
    cmake \
    curl \
    libcurlpp-dev \
    zlib1g-dev \
    nano \
    tmux \
    rsync \
    zip \
    ninja-build \
    openssh-server \
    iproute2 \
    file \
    gcc-aarch64-linux-gnu \
    g++-aarch64-linux-gnu \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Provision SSH build user account for Visual Studio Connection Manager
RUN useradd -rm -d /home/${SSHUSERNAME} -s /bin/bash -g root -G sudo -u 1000 ${SSHUSERNAME} \
    && test -n "$SSHPASSWORD" \
    && printf '%s:%s\n' "$SSHUSERNAME" "$SSHPASSWORD" | chpasswd \
     && echo "${SSHUSERNAME} ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
RUN update-rc.d ssh defaults && mkdir -p /run/sshd && chmod 0755 /run/sshd

WORKDIR /tmp/rti_install

# Copy the Connext host and target packages
COPY rti_connext_dds-${CONNEXT_VERSION}-pro-host-x64Linux.run \
     rti_connext_dds-${CONNEXT_VERSION}-pro-target-x64Linux4gcc7.3.0.rtipkg \
     rti_connext_dds-${CONNEXT_VERSION}-pro-target-armv8Linux4gcc7.3.0.rtipkg \
     ./

# Perform unattended silent installation of RTI Connext DDS host and target packages
RUN chmod +x rti_connext_dds-${CONNEXT_VERSION}-pro-host-x64Linux.run \
    && ./rti_connext_dds-${CONNEXT_VERSION}-pro-host-x64Linux.run --disable_copy_examples true --mode unattended \
    && /opt/rti_connext_dds-${CONNEXT_VERSION}/bin/rtipkginstall -u rti_connext_dds-${CONNEXT_VERSION}-pro-target-x64Linux4gcc7.3.0.rtipkg \
    && /opt/rti_connext_dds-${CONNEXT_VERSION}/bin/rtipkginstall -u rti_connext_dds-${CONNEXT_VERSION}-pro-target-armv8Linux4gcc7.3.0.rtipkg \
    && rm -rf /tmp/rti_install
     
# Configure SSH daemon to eliminate Windows Port 22 collisions and enable password auth
RUN mkdir -p /var/run/sshd \
    && sed -i 's/#Port 22/Port 23/' /etc/ssh/sshd_config \
    && sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config \
    && sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
        
# Configure licence file
RUN echo "export RTI_LICENSE_FILE=/root/rti_license.dat" >> /root/.bashrc
RUN rm -f ${NDDSHOME}/rti_license.dat
RUN ln -fs /root/rti_license.dat ${NDDSHOME}/rti_license.dat > /dev/null 2>&1

# Tweaks
RUN echo "source \${NDDSHOME}/resource/scripts/rtisetenv_x64Linux*.bash" >> /root/.bashrc
RUN echo "source \${NDDSHOME}/resource/scripts/rtisetenv_x64Linux*.bash" >> /home/${SSHUSERNAME}/.bashrc

FROM scratch AS connext-runtime

WORKDIR /root

COPY --from=connext-build / /

RUN service ssh start
EXPOSE 23

# Launch SSH daemon interactively
CMD ["/usr/sbin/sshd","-D"]
