FROM postgres:14.1-alpine

COPY init /docker-entrypoint-initdb.d

ENV POSTGRES_DB=postgres \
   POSTGRES_USER=postgres \
   POSTGRES_PASSWORD=postgres