Nginx Basic ConfigurationThe Nginx Command
The nginx command can be used to perform some useful actions when Nginx is running.

Get current Nginx version and its configured compiling parameters: nginx -V
Test the current Nginx configuration file and / or check its location: nginx -t
Reload the configuration without restarting Nginx: nginx -s reload
Rewrite and Redirection
Force www
The right way is to define a separated server for the naked domain and redirect it.

server {
    listen 80;
    server_name example.org;
    return 301 $scheme://www.example.org$request_uri;
}

server {
    listen 80;
    server_name www.example.org;
    ...
}
Note that this also works with HTTPS site.

                   # Force no-www
Again, the right way is to define a separated server for the www domain and redirect it.

server {
    listen 80;
    server_name example.org;
}

server {
    listen 80;
    server_name www.example.org;
    return 301 $scheme://example.org$request_uri;
}
                   # Force HTTPS
This is also handled by the 2 server blocks approach.

server {
    listen 80;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;

    # let the browsers know that we only accept HTTPS
    add_header Strict-Transport-Security max-age=2592000;

    ...
}
                    # Force Trailing Slash
This configuration only add trailing slash to URL that does not contain a dot because you probably don't want to add that trailing slash to your static files. Source.

rewrite ^([^.]*[^/])$ $1/ permanent;
Redirect a Single Page
server {
    location = /oldpage.html {
        return 301 http://example.org/newpage.html;
    }
}
                    Redirect an Entire Site
server {
    server_name old-site.com
    return 301 $scheme://new-site.com$request_uri;
}
Redirect an Entire Sub Path
location /old-site {
    rewrite ^/old-site/(.*) http://example.org/new-site/$1 permanent;
}
Performance
Contents Caching
Allow browsers to cache your static contents for basically forever. Nginx will set both Expires and Cache-Control header for you.

location /static {
    root /data;
    expires max;
}
If you want to ask the browsers to never cache the response (e.g. for tracking requests), use -1.

location = /empty.gif {
    empty_gif;
    expires -1;
}
