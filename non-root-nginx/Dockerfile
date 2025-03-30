# Use official NGINX base image
FROM nginx:latest

# Create a non-root user
RUN useradd -u 1000 appuser \
    && mkdir -p /home/appuser/nginx /tmp/nginx \
    && chown -R appuser:appuser /home/appuser /var/cache/nginx /var/log/nginx /tmp/nginx \
    && rm -f /etc/nginx/conf.d/default.conf  # Remove default config

# Copy the custom NGINX configuration to a writable directory
COPY nginx.conf /home/appuser/nginx/nginx.conf

# Switch to non-root user
USER appuser

# Expose the correct port
EXPOSE 8080

# Start NGINX with the new config location
CMD ["nginx", "-c", "/home/appuser/nginx/nginx.conf", "-g", "daemon off;"]
