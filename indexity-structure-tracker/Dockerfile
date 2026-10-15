FROM node:10-alpine as builder

WORKDIR /build

COPY package.json package-lock.json /build/
RUN apk add --no-cache python3 make g++
RUN npm ci
COPY . /build
RUN npm run build

# ---

FROM nikolaik/python-nodejs:python3.6-nodejs12

ENV NODE_ENV production
ENV CRON_SCHEDULE="0 0 * * *"
ENV DELETE_LOGS_AFTER_DAYS="10"

ARG CRON_VERSION=3.0pl1-134+deb10u1

WORKDIR /indexity-tracker

COPY --from=builder /build/package*.json /indexity-tracker/
COPY --from=builder /build/dist/ /indexity-tracker/dist/
COPY --from=builder /build/node_modules /indexity-tracker/node_modules
COPY --from=builder /build/python/ /indexity-tracker/python/

RUN pip install --upgrade pip && pip install -r python/requirements.txt

# Buster has moved to the Debian archive. Use only Debian sources for cron;
# the base image's NodeSource/Yarn repositories are not needed here.
# Archived metadata no longer refreshes, but package signatures remain checked.
RUN printf '%s\n' \
      'deb [check-valid-until=no] https://archive.debian.org/debian buster main' \
      'deb [check-valid-until=no] https://archive.debian.org/debian-security buster/updates main' \
      > /tmp/cron-sources.list \
    && apt-get -o Dir::Etc::sourcelist=/tmp/cron-sources.list -o Dir::Etc::sourceparts=- update \
    && apt-get -o Dir::Etc::sourcelist=/tmp/cron-sources.list -o Dir::Etc::sourceparts=- \
      -y install --no-install-recommends "cron=${CRON_VERSION}" \
    && rm -f /tmp/cron-sources.list \
    && rm -rf /var/lib/apt/lists/*
RUN mkdir /indexity-tracker/logs

RUN chmod 0744 /etc/crontab \
    && echo "${CRON_SCHEDULE} \ 
          find /indexity-tracker/logs/ -name '*.log' -ctime +${DELETE_LOGS_AFTER_DAYS} -exec rm -v -f {} \; \ 
          > /proc/1/fd/1 2>/proc/1/fd/2" >> /etc/crontab \
    && echo "#" >> /etc/crontab \
    && crontab /etc/crontab

COPY docker-entrypoint.sh /
ENTRYPOINT ["sh", "/docker-entrypoint.sh"]
