FROM golang:1.12.9-alpine
WORKDIR /go/src/
RUN   apk add git && \
      mkdir /app && \
      git clone https://github.com/ustinsky/example-go-httpd.git && \
      cd example-go-httpd && \
      go build -o /go/bin/httpd
WORKDIR /app
VOLUME /app
EXPOSE 8000
USER 1001
CMD /go/bin/httpd