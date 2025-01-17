# Frontend build
FROM node:14-bullseye AS frontend
WORKDIR /frontend
COPY frontend/package.json frontend/yarn.lock ./
RUN yarn install
COPY frontend/ .
RUN yarn build
CMD ["yarn", "start"]

FROM python:3.9-bullseye AS server

ARG USER_ID=1000
ARG GROUP_ID=1000

ENV DEBIAN_FRONTEND=noninteractive
ENV HGPYTHON3=1

# Python environment variables
ENV PYTHONUNBUFFERED 1
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONPATH /app

# JavaScript applications paths
ENV WEBPACK_BINARY /app/node_modules/.bin/webpack
ENV YUGLIFY_BINARY /app/node_modules/.bin/yuglify
ENV TERSER_BINARY /app/node_modules/.bin/terser

# Install required software.
RUN apt-get update \
    # Enable downloading packages over https
    && apt-get install -y apt-transport-https \
    # Get source for node.js
    && curl -fsSL https://deb.nodesource.com/setup_14.x | bash - \
    # Get source for yarn
    && curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
    && echo 'deb https://dl.yarnpkg.com/debian/ stable main' > /etc/apt/sources.list.d/yarn.list \
    # Install software
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        libmemcached-dev \
        nodejs \
        postgresql-client \
        postgresql-server-dev-13 \
        yarn \
    # Clean up what can be cleaned up.
    && apt-get autoremove -y

WORKDIR /app

# Install Pontoon Python requirements
COPY requirements/* /app/requirements/
RUN pip install -U 'pip>=8' && \
    pip install --no-cache-dir --require-hashes -r requirements/default.txt -r requirements/dev.txt -r requirements/test.txt -r requirements/lint.txt

# Create the app user
RUN groupadd -r --gid=${GROUP_ID} pontoon && useradd --uid=${USER_ID} --no-log-init -r -m -g pontoon pontoon
RUN chown -R pontoon:pontoon /app
USER pontoon

# Install node requirements
COPY package.json .
COPY package-lock.json .
RUN npm install

COPY ./docker/config/server.env .env
COPY --chown=pontoon:pontoon . /app/

# Create the folder for front-end assets
RUN mkdir -p assets
# Run webpack to compile JS files
RUN $WEBPACK_BINARY

COPY --from=frontend /frontend/ ./frontend/

# Run collectstatic in container which puts files in the default place for
# static files.
RUN python manage.py collectstatic --noinput

CMD ["/app/docker/server_run.sh"]
