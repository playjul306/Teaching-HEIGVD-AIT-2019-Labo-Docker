# The base image is one of the offical one
FROM node:latest

# TODO: [GEN] Replace with your name and email
MAINTAINER Julien Benoit <julien.benoit@heig-vd.ch>

# Install the required tools to run our webapp and some utils
RUN apt-get update && apt-get -y install wget curl vim && apt-get clean && npm install -g bower

# We copy the application and make sure the dependencies are installed before
# other operations. Doing so will reduce the time required to build this image
# as downloading NPM dependencies can be quite long.
COPY app /backend/app
RUN cd /backend/app && npm install && bower install --allow-root

# Download and install S6 overlay
RUN curl -sSLo /tmp/s6.tar.gz https://github.com/just-containers/s6-overlay/releases/download/v1.17.2.0/s6-overlay-amd64.tar.gz \
  && tar xzf /tmp/s6.tar.gz -C / \
  && rm -f /tmp/s6.tar.gz

# Install serf (for decentralized cluster membership: https://www.serf.io/)
RUN mkdir /opt/bin \
    && curl -sSLo /tmp/serf.gz https://releases.hashicorp.com/serf/0.7.0/serf_0.7.0_linux_amd64.zip \
    && gunzip -c /tmp/serf.gz > /opt/bin/serf \
    && chmod 755 /opt/bin/serf \
    && rm -f /tmp/serf.gz

# Copy the S6 service and make the run script executable
COPY services/node /etc/services.d/node
RUN chmod +x /etc/services.d/node/run

# Copy the serf agent and make the run script executable
COPY services/serf /etc/services.d/serf
RUN chmod +x /etc/services.d/serf

# Clean up
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Expose the ports for Serf
EXPOSE 7946 7373

# Expose the web application port
EXPOSE 3000

# Define an environment variable for the role of the container
ENV ROLE backend

# This will start S6 as our main process in our container
ENTRYPOINT ["/init"]

