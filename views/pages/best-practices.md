---
title: Best practices
layout: default
---

Wookie best practices
=====================
Here are some tips on making your Wookie experience as smooth as possible. Keep
in mind that this page may update frequently as issues come up and as Wookie is
used more in production.

{{toc}}

### Error handling
There is a [section covering error handling](/docs/error-handling) in the docs,
but I feel this is yet another chance to reiterate.

Error handling in asynchronous systems is very important. A rogue condition
can completely halt the execution of your application and leave your clients
hanging indefinitely.

Familiarize yourself with [cl-async's error handling](http://orthecreedence.github.io/cl-async/event-handling)
and also [Wookie's error handling](/docs/error-handling) before taking your app
to production.

Here are some tips on checking your error handling:

- Try sending bogus data to your app
- Try accessing pages that don't exist
- Intentionally add in conditions that will cause an error. See how the app
responds.

If everything is set up properly, you should get no crashes in your app, even
in the event that errors escape. See [an example/simple error handler for a 
Wookie app](https://github.com/orthecreedence/wookie-doc/blob/master/init.lisp).

*NOTE:* A top-level error handler is no excuse to not catch errors at the
source. It is used as a last resort to ensure the well-being of the application
as a whole. If errors escape to the top-level, try to figure out what happened
and fix the situation.

### Reverse Proxy
Wookie can be run by itself, but because it's new and hasn't been battle tested,
it's important to be weary of running it on its own in the wild. There may be 
uncaught security issues or bugs lurking around waiting to crash your app.
My suggestion is to use some sort of reverse proxy. This can provide a layer of
protection (less chance of malformed HTTP requests crashing your server).

[NginX](http://nginx.org/) is a great reverse proxy + web server. On top of
being really fast, it can serve your static content as well. Wookie's
[directory router](/docs/core-plugins#directory-router) allows an app to serve
static content and is fine for smaller websites or if you are developing an app
locally and don't want to set up some sort of reverse proxy. However I can
*guarantee* that NginX will always beat the directory router performance-wise,
and fully recommend using it in favor of the directory router in production.

#### Sample NginX config
This is a sample of the config used to run the Wookie doc site through NginX.

```
upstream wookiedoc {
    server 127.0.0.1:9061;
}

server {
    # set up the host/port
    listen 80;
    server_name wookie.beeets.com;

    # note we set the location to the static files here
    set $server_root /srv/www/wookie-doc/webroot;
    root $server_root;

    charset utf-8;

    access_log /var/log/www/wookie.beeets.com-access.log main;
    error_log /var/log/www/wookie.beeets.com-error.log;

    location / {
        # serve static content
        if (-f $request_filename) {
            # NginX can add caching headers to your static content, adding to the app's performance
            add_header Cache-Control "public, max-age=86400";
            break;
        }
        if (-d $request_filename) {
            break;
        }

        # file not being loaded, forward request to Wookie
        proxy_pass http://wookiedoc;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 h
        proxy_redirect off;
        proxy_buffering off;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}
```