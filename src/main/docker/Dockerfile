FROM eclipse-temurin:11.0.15_10-jre-focal@sha256:f9ee9b3a0a0b0e36ae33256ed13f73df34059f427cfc39ccd43b953f5752e1f2 AS jre-build

FROM debian:bullseye-20220711-slim@sha256:f576b8067b77ff85c70725c976b7b6cde960898e2f19b9abab3fb148407614e2

# Arguments that can be passed at build time
# Directory names must end with / to avoid errors when ADDing and COPYing
ARG COMMIT_SHA=unknown
ARG APP_VERSION=0.0.0
ARG APP_DIR=/opt/owasp/dependency-track/
ARG DATA_DIR=/data/
ARG UID=1000
ARG GID=1000
ARG WAR_FILENAME=dependency-track-apiserver.jar

ENV TZ=Etc/UTC \
    # Dependency-Track's default logging level
    LOGGING_LEVEL=INFO \
    # Environment variables that can be passed at runtime
    JAVA_OPTIONS="-XX:+UseParallelGC -XX:MaxRAMPercentage=90.0" \
    # The web context defaults to the root. To override, supply an alternative context which starts with a / but does not end with one
    # Example: /dtrack
    CONTEXT="/" \
    # Injects the build-time ARG "WAR_FILENAME" as an environment variable that can be used in the CMD.
    WAR_FILENAME=${WAR_FILENAME} \
    # Set JAVA_HOME for the copied over JRE
    JAVA_HOME=/opt/java/openjdk \
    PATH="/opt/java/openjdk/bin:${PATH}" \
    LANG=C.UTF-8 \
    # Ensure user home is always set to DATA_DIR, even for arbitrary UIDs (such as used by OpenShift)
    HOME=${DATA_DIR} \
    # Default notification publisher templates override environment variables
    DEFAULT_TEMPLATES_OVERRIDE_ENABLED=false \
    DEFAULT_TEMPLATES_OVERRIDE_BASE_DIRECTORY=${DATA_DIR}

# Create the directories where the WAR will be deployed to (${APP_DIR}) and Dependency-Track will store its data (${DATA_DIR})
# Create a user and assign home directory to a ${DATA_DIR}
# Ensure UID 1000 & GID 1000 own all the needed directories
RUN mkdir -p ${APP_DIR} ${DATA_DIR} \
    && addgroup --system --gid ${GID} dtrack || true \
    && adduser --system --disabled-login --ingroup dtrack --no-create-home --home ${DATA_DIR} --gecos "dtrack user" --shell /bin/false --uid ${UID} dtrack || true \
    && chown -R dtrack:0 ${DATA_DIR} ${APP_DIR} \
    && chmod -R g=u ${DATA_DIR} ${APP_DIR} \
    \
    # Install wget for health check
    && apt-get -yqq update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -yqq --no-install-recommends wget \
    && rm -rf /var/lib/apt/lists/*

# Copy JRE from temurin base image
COPY --from=jre-build /opt/java/openjdk $JAVA_HOME

# Copy the compiled WAR to the application directory created above
COPY ./target/${WAR_FILENAME} ${APP_DIR}

# Specify the user to run as (in numeric format for compatibility with Kubernetes/OpenShift's SCC)
USER ${UID}

# Specify the container working directory
WORKDIR ${APP_DIR}

# Launch Dependency-Track
CMD java ${JAVA_OPTIONS} -DdependencyTrack.logging.level=${LOGGING_LEVEL} -jar ${WAR_FILENAME} -context ${CONTEXT}

# Specify which port Dependency-Track listens on
EXPOSE 8080

# Add a healthcheck using the Dependency-Track version API
HEALTHCHECK --interval=5m --timeout=3s CMD wget --no-proxy -q -O /dev/null http://127.0.0.1:8080${CONTEXT}api/version || exit 1

# metadata labels
LABEL \
    org.opencontainers.image.vendor="OWASP" \
    org.opencontainers.image.title="Official Dependency-Track Container image" \
    org.opencontainers.image.description="Dependency-Track is an intelligent Component Analysis platform" \
    org.opencontainers.image.version="${APP_VERSION}" \
    org.opencontainers.image.url="https://dependencytrack.org/" \
    org.opencontainers.image.source="https://github.com/DependencyTrack/dependency-track" \
    org.opencontainers.image.revision="${COMMIT_SHA}" \
    org.opencontainers.image.licenses="Apache-2.0" \
    maintainer="steve.springett@owasp.org"
