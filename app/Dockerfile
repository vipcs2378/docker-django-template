# python full image is around 900MB and slim is around 130MB
# FROM python:3.11-bullseye as python-base
FROM python:3.11-slim-bullseye as base

ENV PYTHONFAULTHANDLER=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONHASHSEED=random \
    PYTHONDONTWRITEBYTECODE=1 \
    # PATH for pipx and poetry
    PATH="$PATH:/root/.local/bin"

# disable pip version check and cache during build time
ARG PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_NO_CACHE_DIR=1

# RUN pip install --upgrade pip
# Create a user to avoid running containers as root in production
# Probably don't need this for rootless docker user
# RUN addgroup --system web \
#     && adduser --system --ingroup web web

RUN apt-get update && apt-get install -y -q --no-install-recommends \
    # dependencies for building Python packages
    build-essential \
    # postgress client (psycopg3) dependencies
    # check out https://www.psycopg.org/psycopg3/docs/basic/install.html
    libpq5 \
    # cleaning up unused files to reduce the image size
    && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /home/web/code

# Layer for Poetry installation
FROM base as poetry

ENV POETRY_VERSION=1.4.2 \
    POETRY_NO_INTERACTION=1 \
    POETRY_CACHE_DIR='/var/cache/pypoetry'


# Option 1 - install poetry with pip with poetry==version number
# ARGs already disabled version check and cache
RUN --mount=type=cache,target=/root/.cache \
    # python3 -m pip install poetry==${POETRY_VERSION}
    pip install poetry==${POETRY_VERSION}

# Option 2 - install poetry with pipx
# install pipx
# RUN pip install pipx
# install poetry
# RUN pipx install poetry==${POETRY_VERSION}

# Option 3 - install poetry with pip and requirements.txt file 
# requirements.txt is only for Poetry installation, so it includes only poetry with its version
# COPY requirements.txt ./
# see option 1 to remove --disable-pip-version-check
# RUN --mount=type=cache,target=/root/.cache \
#     python3 -m pip install --disable-pip-version-check --requirement=requirements.txt && \
#     rm requirements.txt


# Switch to the non-root user
# USER web
# Create a directory for the source code and use it as base path

# Copy the python depencencies list for pip
# COPY --chown=web:web ./requirements/pyproject.toml poetry.lock /requirements
COPY /requirements/. /
# Switch to the root user temporary, to grant execution permissions.
# USER root

# COPY poetry.lock pyproject.toml ./
# COPY pyproject.toml /app
RUN poetry export --output requirements-prod.txt && \
    poetry export --with dev --output requirements-dev.txt




# About multiple FROMs
# https://stackoverflow.com/questions/33322103/multiple-froms-what-it-means

# => ERROR [internal] load metadata for docker.io/library/poetry:latest
# check if FROM poetry_builder matches to FROM base as poetry_builder


# how to enable the Buildkit
# go to /etc/docker on your host computer
# create daemon.json and insert below content, remove the hash tag
# {
#   "features": {
#     "buildkit": true
#   }
# }


# Need to know about "from=poetry" in the code below. poetry is from "FROM base as poetry"

# using poetry layer without installing poetry in the container

# how to deal with multiple FROMS
# docker build -f Dockerfile -t containername --target (layername)
# in "FROM base as poetry", poetry is the layer name
# ex) in CLI, "docker build -f Dockerfile -t pytest3 --target poetry ."
# development mode
FROM base as dev

RUN --mount=type=cache,target=/root/.cache \
    --mount=type=bind,from=poetry,source=/home/web/code,target=/home/web/code \
    pip install --no-deps --requirement=/home/web/code/requirements-dev.txt
    # python3 -m pip install --no-deps --requirement=/app/requirements-dev.txt

COPY . /home/web/code
# it seems you should use only the file name without ".py"
# "python -m test" instead of "python -m test.py
EXPOSE 8080

RUN chmod +x start-dev-server.sh
CMD ["start-dev-server.sh"]


# Production mode
FROM base as prod

RUN --mount=type=cache,target=/root/.cache \
    --mount=type=bind,from=poetry,source=/home/web/code,target=/home/web/code \
    pip install --no-deps --requirement=/home/web/code/requirements-prod.txt
    # python3 -m pip install --no-deps --requirement=/app/requirements.txt

COPY /autobot/. /app
# it seems you should use only the file name without ".py"
# "python -m test" instead of "python -m test.py"
CMD ["python", "-m", "test"]