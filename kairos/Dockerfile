FROM quay.io/kairos/kairos-init:v0.6.2 AS kairos-init
FROM ubuntu:24.04

ARG VERSION=1.0.0

RUN --mount=type=bind,from=kairos-init,src=/kairos-init,dst=/kairos-init \
    /kairos-init --version "${VERSION}"

