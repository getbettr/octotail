# docker inspect --format='{{index .RepoDigests 0}}' archlinux
FROM archlinux@sha256:e5d672031f7479b0ce222486f1b5c8b07c931327d16050ad6078a8cd68cf870f

COPY prepare.sh /prepare.sh
RUN /prepare.sh

USER e2e
WORKDIR /repo
