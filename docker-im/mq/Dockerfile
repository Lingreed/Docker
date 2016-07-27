FROM java:8

RUN mkdir -p /apps/projects/logs \
    && mkdir -p /apps/jar \
    && mkdir -p /apps/ymls/mq \
    && mkdir -p /apps/projects/mqcenter/cert

COPY mqcenter-1.0.0-exec.jar /apps/jar
COPY application.yml /apps/ymls/mq

VOLUME /apps/projects/logs

CMD ["java","-jar","/apps/jar/mqcenter-1.0.0-exec.jar","--spring.config.location=/apps/ymls/mq/application.yml"]
