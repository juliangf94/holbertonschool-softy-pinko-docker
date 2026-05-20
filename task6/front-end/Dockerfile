# Use the official Nginx image as the base (already has Nginx installed)
FROM nginx:latest

# Copy the front-end static files into the directory Nginx will serve from
COPY ./softy-pinko-front-end /var/www/html/softy-pinko-front-end

# Replace the default Nginx config with our custom one
COPY ./softy-pinko-front-end.conf /etc/nginx/conf.d/default.conf
