# Create from official mysql image
FROM mysql

# Add a database
ENV MYSQL_DATABASE testdb

# Initial db scripts
COPY ./init-scripts /docker-entrypoint-initdb.d/
