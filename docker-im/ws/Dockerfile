FROM java:8

RUN mkdir -p /apps/projects/logs \
    && mkdir -p /apps/jar \
    && mkdir -p /apps/ymls/ws

COPY im-ws-0.0.1-exec.jar /apps/jar
COPY config-extend.yml /apps/ymls/ws

VOLUME /apps/projects/logs

EXPOSE 20000
CMD ["java","-jar","/apps/jar/im-ws-0.0.1-exec.jar"]
