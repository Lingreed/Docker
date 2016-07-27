FROM java:8

RUN mkdir -p /apps/projects/logs \
    && mkdir -p /apps/projects/launchrAttachment \
    && mkdir -p /apps/jar \
    && mkdir -p /apps/ymls/api

COPY im-api-0.0.1-exec.jar /apps/jar
COPY application.yml /apps/ymls/api

VOLUME /apps/projects/logs
VOLUME /apps/projects/launchrAttachment

EXPOSE 20001
CMD ["java","-jar","/apps/jar/im-api-0.0.1-exec.jar","--spring.config.location=/apps/ymls/api/application.yml"]
