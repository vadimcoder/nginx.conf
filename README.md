# This config sets up three main optimization layers plus security headers:
# Compression — Two tiers of on-the-fly compression for text-based assets (HTML, CSS, JS, JSON, XML, SVG). Brotli is the primary (smaller output, faster decompression in browsers), and gzip acts as a fallback for clients that don't support Brotli. brotli_static on means nginx will also serve pre-compressed .br files from disk if they exist, avoiding runtime compression cost entirely. Compression levels (gzip 6, brotli 5) are moderate — good ratio without burning too much CPU.

# File cache — open_file_cache keeps file descriptors, metadata (size, modification time), and existence info for up to 10,000 files in memory for 60s of inactivity. Files accessed at least twice get cached for 120s. This eliminates repeated stat() and open() syscalls on every request, which matters a lot for static sites with many assets.

# Client-side caching (in the site config) — A two-part strategy: index.html is explicitly never cached (no-store), so users always get the latest version and any updated asset references. All hashed/immutable assets (JS, CSS, images, fonts) get a 1-year cache with the immutable directive, telling browsers not to even revalidate them. This is the standard pattern for apps that use content-hashed filenames (e.g., app.3fa9c1.js). access_log off for those assets reduces I/O.

# Security headers — Standard hardening set: HSTS with a 1-year max-age forces HTTPS (including subdomains), X-Frame-Options SAMEORIGIN prevents clickjacking, nosniff stops MIME-type sniffing, X-XSS-Protection enables the legacy browser XSS filter, and Referrer-Policy limits referrer leakage to origin-only on cross-origin requests.


# /etc/nginx/nginx.conf:

http {
        http2 on; # ENABLE HTTP2
        
        # disable broadcasting Nginx version
        server_tokens off;

        # Basic — global rate limit per IP:
        # Define zone in http block (10MB shared memory, 10 req/sec per IP):
        limit_req_zone $binary_remote_addr zone=global:10m rate=10r/s;
        
        ## GZIP (fallback compression)
        gzip on;
        gzip_comp_level 6;
        gzip_min_length 1000;
        gzip_vary on;
        gzip_types
            text/plain
            text/css
            application/json
            application/javascript
            application/xml
            application/rss+xml
            image/svg+xml;

        ## BROTLI (IF MODULE IS INSTALLED, may require additional libnginx-mod-http-brotli-filter):
        brotli on;
        brotli_comp_level 5;
        # brotli_static on; # <- Only for serving pre-compressed .br files (may require additional module libnginx-mod-http-brotli-static)
        brotli_types
            text/plain
            text/css
            application/javascript
            application/json
            image/svg+xml;

        ## FILE CACHE (very important for static sites):
        open_file_cache max=10000 inactive=60s;
        open_file_cache_valid 120s;
        open_file_cache_min_uses 2;
        open_file_cache_errors on;
}

## /etc/nginx/sites-available/my-site.com:

server {
    listen 443 ssl;
    listen [::]:443 ssl; 
   
    # DO NOT CACHE HTML:
    location = /index.html {
        include snippets/security-headers.conf;
        add_header Cache-Control "no-cache, no-store, must-revalidate" always;
    }
   
    # LONG CACHE IMMUTABLE ASSETS:
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|webp)$ {
        include snippets/security-headers.conf;
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }
}

## /etc/nginx/snippets/security-headers.conf:
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
