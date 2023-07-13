FROM openmicroscopy/omero-server:5.6.7
MAINTAINER luis.kuhn@gmail.com

ARG BIOFORMATS_VERSION=6.12.0

USER root

RUN yum -y install maven

RUN mkdir /opt/omero/bio-formats-omero && \
    cd /opt/omero/bio-formats-omero  && \
    wget https://raw.githubusercontent.com/sbesson/bio-formats-omero/master/pom.xml

RUN echo ${BIOFORMATS_VERSION}
RUN cd /opt/omero/bio-formats-omero  && \
    mvn clean package -Domero.home=/opt/omero/server/OMERO.server -Dbioformats.version=${BIOFORMATS_VERSION}

USER omero-server

RUN cd /opt/omero/server/OMERO.server/bin && \
    ./omero admin diagnostics
