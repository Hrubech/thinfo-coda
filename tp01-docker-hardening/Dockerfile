FROM nginx:alpine

# Crée un utilisateur avec UID fixe (1001) pour simplifier la gestion des droits tmpfs
RUN addgroup -S -g 1001 appgroup && adduser -S -u 1001 -G appgroup appuser

# Empêche l'entrypoint Nginx de tenter de modifier default.conf (bloqué en read-only)
RUN rm -f /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh

# Nettoie la directive 'user' du config principal (inutile en non-root et provoque des warnings)
RUN sed -i '/user  nginx;/d' /etc/nginx/nginx.conf

# Prépare les dossiers pour l'utilisateur non-root
RUN mkdir -p /var/cache/nginx /var/run /var/log/nginx && \
    chown -R appuser:appgroup /var/cache/nginx /var/run /var/log/nginx /usr/share/nginx/html /etc/nginx/conf.d

COPY default.conf /etc/nginx/conf.d/default.conf
COPY index.html /usr/share/nginx/html/index.html

USER appuser
EXPOSE 8080