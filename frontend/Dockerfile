FROM node:16-alpine AS builder

WORKDIR /frontend

COPY package.json package-lock.json /frontend/

RUN npm ci

ARG VUE_APP_API_URL=/api
ENV VUE_APP_API_URL=${VUE_APP_API_URL}

COPY . .

RUN npm run build

FROM nginx:1.27-alpine AS runtime

RUN adduser -D -H -u 1001 -s /sbin/nologin svc-frontend

COPY --from=builder --chown=svc-frontend:svc-frontend /frontend/dist /usr/share/nginx/html/momo-store
COPY --chown=svc-frontend:svc-frontend nginx.conf /etc/nginx/conf.d/default.conf

RUN touch /var/run/nginx.pid \
 && chown svc-frontend:svc-frontend /var/run/nginx.pid \
 && chown -R svc-frontend:svc-frontend /var/cache/nginx

USER svc-frontend

EXPOSE 8080

HEALTHCHECK --interval=20s --timeout=2s --start-period=5s --retries=3 \
    CMD wget -q -O /dev/null http://127.0.0.1:8080/healthz || exit 1

CMD ["nginx", "-g", "daemon off;"]
