FROM node:14-alpine AS webui-builder
WORKDIR /src

ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true

COPY webui/package.json webui/package-lock.json .
RUN npm ci --save-dev --no-audit

COPY webui .
RUN mkdir -p ./public/assets/styles && \
  node dark && \
  node light && \
  node cyber && \
  printf "%s\n" \
    "@import url(/api/setting/getBackground.less);" \
    ".body-bg {" \
    "  background:@vt-bg-image;" \
    "  background-position-x:center;" \
    "  background-position-y:center;" \
    "  background-size:cover;" \
    "}" \
    "" \
    ".login-layout {" \
    "  background:@body-background;" \
    "}" \
    "" >> ./public/assets/styles/cyber.less && \
  : > ./public/assets/styles/follow.less && \
  npm run build && \
  rm -rf node_modules

RUN mkdir -p /out && \
  cp -a /app/static /out/static

FROM node:18-alpine AS app-builder
WORKDIR /src

ENV PUPPETEER_SKIP_CHROMIUM_DOWNLOAD=true

RUN apk add --no-cache python3 make g++ gcc libc-dev cmake

COPY package.json package-lock.json ./
RUN npm ci --save-dev --no-audit

COPY app ./app
COPY docker ./docker
COPY --from=webui-builder /out/static ./app/static

RUN mkdir -p /out && \
  cp -a /src/node_modules /out/node_modules && \
  cp -a /src/app /out/app && \
  cp -a /src/docker /out/docker

FROM node:18-alpine AS runtime
WORKDIR /app/vertex

ENV TZ=Asia/Shanghai
ENV NODE_ENV=production

RUN apk add --no-cache \
  bash tzdata redis xvfb chromium \
  ca-certificates ttf-freefont nss freetype harfbuzz \
  shadow libstdc++

# 只拷贝运行所需
COPY --from=app-builder /out .

RUN set -eux; \
  mkdir -p /vertex; \
  ln -s /vertex/data /app/vertex/app/data; \
  ln -s /vertex/db /app/vertex/app/db; \
  ln -s /vertex/logs /app/vertex/logs; \
  ln -s /vertex/torrents /app/vertex/torrents; \
  mv /app/vertex/app/config /app/vertex/app/config_backup; \
  ln -s /vertex/config /app/vertex/app/config; \
  ln -s /app/localtime /etc/localtime; \
  if [ -x /usr/bin/chromium-browser ] && [ ! -e /usr/bin/chromium ]; then ln -s /usr/bin/chromium-browser /usr/bin/chromium; fi; \
  printf "bind 127.0.0.1\ndaemonize yes\n" > /app/redis.conf; \
  adduser -D -h /app/vertex -s /bin/sh vt; \
  chown -R vt /vertex

EXPOSE 3000
CMD ["bash", "/app/vertex/docker/start.sh"]
