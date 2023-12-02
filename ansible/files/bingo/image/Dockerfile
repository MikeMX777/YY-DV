FROM busybox AS builder
#Шоб heathcheck работал
ARG BUSYBOX_VERSION=1.31.0-i686-uclibc
ADD https://busybox.net/downloads/binaries/$BUSYBOX_VERSION/busybox_WGET /wget
RUN chmod a+x /wget


FROM gcr.io/distroless/base-debian11

COPY --from=builder /wget /usr/bin/wget

COPY bingo /
USER nobody:nobody
ENTRYPOINT ["/bingo"]
