# Utilizamos la imagen base con Java 11 de Microsoft
FROM mcr.microsoft.com/devcontainers/java:1-11-bullseye

# Establecemos el directorio de trabajo
WORKDIR /transactional-producer

# Copiamos el código fuente de la aplicación al contenedor
COPY ./transactional-producer /transactional-producer

# Modificamos los permisos del script sdkman-init.sh
RUN chmod +x /usr/local/sdkman/bin/sdkman-init.sh

# Cargamos SDKMAN automáticamente y ejecutamos el script sdkman-init.sh
RUN /bin/bash -c "source /usr/local/sdkman/bin/sdkman-init.sh \
    && sdk install gradle 8.6"

# Compilamos la aplicación con Gradle
RUN /bin/bash -c "gradle build"