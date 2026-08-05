FROM docker.m.daocloud.io/library/mysql:8.0

COPY deploy/mysql/init/000-create-soccer.sql /docker-entrypoint-initdb.d/000-create-soccer.sql
COPY platform/deploy/mysql/init/001_schema.sql /docker-entrypoint-initdb.d/010-platform-schema.sql
