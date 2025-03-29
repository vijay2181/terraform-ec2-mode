# Use official NGINX base image
FROM nginx:latest

# Create a non-root user with UID 1000
RUN useradd -u 1000 appuser \
    && mkdir -p /home/appuser /tmp/nginx \
    && chown -R appuser:appuser /home/appuser /var/cache/nginx /var/log/nginx /tmp/nginx

# Copy the custom NGINX configuration
COPY nginx.conf /etc/nginx/nginx.conf

# Switch to non-root user
USER appuser

# Expose the default NGINX port
EXPOSE 8080

# Start NGINX with the new configuration
CMD ["nginx", "-g", "daemon off;"]
