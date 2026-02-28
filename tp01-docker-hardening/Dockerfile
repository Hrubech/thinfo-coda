FROM nginx:alpine

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

RUN mkdir -p /var/cache/nginx /var/run /var/log/nginx && \
    chown -R appuser:appgroup \
      /var/cache/nginx /var/run /var/log/nginx \
      /usr/share/nginx/html /etc/nginx/conf.d

COPY default.conf /etc/nginx/conf.d/default.conf
COPY index.html /usr/share/nginx/html/index.html

USER appuser
EXPOSE 8080
