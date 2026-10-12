# Sample app image: unprivileged nginx serving a static page on :8080.
FROM nginxinc/nginx-unprivileged:1.27-alpine

LABEL org.opencontainers.image.title="sample-webapp" \
      org.opencontainers.image.description="Sample app for the azure-devops-pipeline-templates repo"

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 8080

# nginx-unprivileged already runs as the non-root 'nginx' user and listens on 8080.
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://127.0.0.1:8080/ || exit 1
