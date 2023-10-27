ARG APP_NAME=insiders-clustering
ARG APP_PATH=/opt/${APP_NAME}
ARG PYTHON_VERSION=3.11
ARG POETRY_VERSION=1.5.0

FROM python:${PYTHON_VERSION}-slim-bullseye AS staging
ARG PYTHON_VERSION
ARG POETRY_VERSION

# Update libraries and install some helper tools
RUN apt-get update -y && \
  apt-get upgrade -y && \
  apt-get install -y --no-install-recommends \
  bash ca-certificates curl git gnupg openjdk-11-jdk sudo wget && \
  rm -rf /var/lib/apt/lists/*

# Set environment variables
ENV \
  PYTHONDONTWRITEBYTECODE=0 \
  PYTHONUNBUFFERED=1 \
  PYTHONFAULTHANDLER=1
ENV \
  POETRY_VERSION=${POETRY_VERSION} \
  POETRY_HOME="/opt/poetry" \
  PATH="${POETRY_HOME}/bin:${PATH}" \
  POETRY_VIRTUALENVS_IN_PROJECT=false \
  POETRY_NO_INTERACTION=1
ENV \
  {DOCKERIZE_VERSION}="v0.6.1"

# Install dockerize to ensure availability of container dependencies
RUN wget https://github.com/jwilder/dockerize/releases/download/${DOCKERIZE_VERSION}/dockerize-linux-amd64-${DOCKERIZE_VERSION}.tar.gz \
  && tar -C /usr/local/bin -xzvf dockerize-linux-amd64-${DOCKERIZE_VERSION}.tar.gz \
  && rm dockerize-linux-amd64-${DOCKERIZE_VERSION}.tar.gz

# Install poetry
RUN curl -sSL https://install.python-poetry.org | POETRY_HOME=/opt/poetry python3 - \
  && cd /usr/local/bin \
  && ln -s /opt/poetry/bin/poetry

FROM staging AS development
# Setup Jupyter Lab for development
WORKDIR /home/jovyan
EXPOSE 8888

ARG USERNAME=unpython
ARG USER_UID=1000
ARG USER_GID=${USER_UID}
RUN groupadd --gid ${USER_GID} ${USERNAME} \
  && useradd --uid ${USER_UID} --gid ${USER_GID} -m ${USERNAME} -s /bin/bash \
  && echo ${USERNAME} ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/${USERNAME} \
  && chmod 0440 /etc/sudoers.d/${USERNAME}
USER ${USERNAME}

COPY pyproject.toml .
RUN poetry install --no-root

ENTRYPOINT ["poetry", "run"]
CMD ["python", "-m", "jupyter", "lab", "--ip=0.0.0.0", "--port=8888", "--allow-root", "--no-browser"]

FROM development AS build
WORKDIR ${APP_PATH}
COPY --from=development ${APP_PATH}/pyproject.toml .
RUN poetry export \
  --without dev \
  --without hashes \
  -f requirements.txt \
  --output requirements.txt

FROM python:${PYTHON_VERSION}-slim-bullseye AS production
ARG APP_NAME
ARG APP_PATH
WORKDIR ${APP_PATH}

ENV \
  PYTHONDONTWRITEBYTECODE=1 \
  PYTHONUNBUFFERED=1 \
  PYTHONFAULTHANDLER=1
ENV \
  PIP_NO_CACHE_DIR=off \
  PIP_DISABLE_PIP_VERSION_CHECK=on \
  PIP_DEFAULT_TIMEOUT=100
EXPOSE ${API_PORT}

# Download and install only production dependencies
COPY --from=development ${APP_PATH}/insiders-clustering ./
COPY --from=development ${APP_PATH}/.env .
COPY --from=build ${APP_PATH}/requirements.txt .
RUN pip install --upgrade pip && \
  pip install -r requirements.txt

# Wait until database is ready to run the ASGI web server
COPY --from=staging /usr/local/bin/dockerize /usr/local/bin
ENTRYPOINT ["dockerize", "-wait", "tcp://db:${DB_PORT}", "-timeout", "30s"]
CMD ["poetry", "run", "src.api.server"]
