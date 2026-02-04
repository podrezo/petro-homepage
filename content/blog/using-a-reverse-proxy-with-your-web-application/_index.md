---
title: "Using a reverse proxy with your web application"
image: /blog/using-a-reverse-proxy-with-your-web-application/cover.jpg
summary: "Use AWS Lambda and Ruby to create a bot that posts CloudWatch logs to a Slack channel."
first_published: 2019-09-11
---


# Using a reverse proxy with your web application

Running a web application such as a Node.js API, Ruby on Rails site, or really anything that accepts HTTP requests simultaneously with Apache or NGINX may seem counter-intuitive: you might ask yourself “why the heck would I want to run a web server with my application that already acts as a web server?” — but that’s the wrong question to be asking. The right question is “What kind of advantages do I gain by using the two together?” and the answer may surprise you.

----------

### Why use a reverse proxy?

There are several huge advantages to be gained.

-   You can set up a ‘under maintenance page which will be displayed when your app crashes or is restarting (instead of the user getting a connection refused message)
-   You can run not  _just_  your one web app on the HTTP(S) port but multiple web apps or static sites using the same port using virtual hosts. This is especially useful if your application is an API and you’d like to take care of serving static content as a separate concern
-   Kind of a subpoint to the last one, but this creates a single point of entry for your application including all its pieces to the outside world which can make for a more cohesive
-   You can scale your application by performing load balancing for production environments

These are just some of the ‘pros’ to using a reverse proxy with your web app. There are of course a few drawbacks, albeit minor ones:

-   You will incur additional latency by adding the extra layer. This is usually negligible, but still is a factor
-   You increase the complexity of your application (from the end-to-end sense) and will create an additional failure point which moreover means you may find the task of finding where an issue is coming from more time consuming

### What is a reverse proxy?

A proxy in terms of the internet is typically an HTTP (or SOCKS protocol) server that relays requests on behalf of the client to the internet. This is typically used in a corporate network environment to be able to filter and monitor requests coming from the business’s location (e.g. to block sites like Facebook at work). So what is a reverse proxy then? A reverse proxy is exactly like it sounds like — it takes requests from the internet and exposes some host to it by relaying the requests. That is, the proxy is exposed to the internet and the protected host in an internal network that is not but allows connections from the proxy which will relay the requests from the web. Web servers like Apache and NGINX offer the reverse proxy functionality. Here’s a step-by-step summary of how it works:

1.  User requests mydomain.com/some_resource
2.  The web server at mydomain.com recognizes the mydomain.com vhost but but is configured such that  _some_resource_  triggers the reverse proxy (by way of using a regular expression to catch the path, or using a static path) instead of trying to serve a file
3.  The web server forwards all requests for  _some_resource_  them to some other server; say a node app running express on port 8000 that’s on the same server. So the reverse proxy is configured to say “send all requests to some_resource to http://localhost:8000”
4.  The web app at localhost:8000 receives the request from the web server. Crucial to note here is that this was not a redirect! The web browser making the request has no idea this is happening behind the scenes and just sits and waits until the transaction between the web server and the web application is complete.
5.  Once the web server has completed the request to the web app, it now forwards the results to the user who originally requested it (the browser)
6.  The user has now received  _some_resource_; however as far as they are aware the file came from mydomain.com not from localhost:8000 on the same server

In the midst of all this, we can have additional directives that can describe what kind of page to return in case the web application is down.

### Getting Started with Configuring Apache as a Reverse Proxy (NGINX config at the end of this article)

Before continuing, make sure the proxy module is enabled using “sudo a2enmod proxy_http” if applicable on your machine.

You will have to create the proper vhost for your site. In our example, let’s suppose we’re trying to set up some sort of pastebin site as  **pastebin.mydomain.com**  where  **mydomain.com**  is configured to point to some server running Apache on port 80. Our pastebin application is listening on port 8000 and we  [have it running via forever](http://blog.podrezo.com/init-d-startupshutdown-script-for-node-js-applications-via-forever/ "Init.d startup/shutdown script for Node.JS applications via forever")  in a continuous loop. Open up your apache2 configuration (for me it is located in  **/etc/apache2/sites-available**  with a separate file per vhost/site) and create the vhost as follows:

```apacheconf
<VirtualHost *:80>
ServerName pastebin.mydomain.com
ServerAlias www.pastebin.mydomain.com
DocumentRoot /var/www/pastebinjs/
Options -Indexes
ErrorDocument 503 /maintenance.html
</VirtualHost>
```

So what does this get us? Let’s go through it line by line:

1.  Apache is going to listen on port 80 at any address/host for the connections to this vhost
2.  The specific host header we’re looking for is ‘pastebin.mydomain.com’
3.  We allow an alias called ‘www.pastebin.mydomain.com' for some legacy support and fool-proofing
4.  We point all requests to the document root of /var/www/pastebinjs/
5.  Disable indexes as a bit of a security thing and just good practice
6.  Set up a custom 503 (“Service Unavailable”) error page to be /maintenance.html on the server

The final result is that we have a static website rooted at /var/www/pastebinjs with a custom error message. In this folder we would create maintenance.html and put some message like “we are under maintenance and looking into it! We’ll be back shortly!” so as not to scare the users. But now what? What about the node app? Alright, so the next part is almost as simple. We expand our vhost to the following:

```apacheconf
<VirtualHost *:80>
ServerName pastebin.mydomain.com
ServerAlias www.pastebin.mydomain.com
DocumentRoot /var/www/pastebinjs/
Options -Indexes
ErrorDocument 503 /maintenance.html

ProxyRequests on
ProxyPass /maintenance.html !
ProxyPass / http://localhost:8000/
</VirtualHost>
```

What do we have now then? Well, we enabled the proxy module in Apache and we told it that maintenance.html will not be proxied through but will instead be served by the Apache server as a ‘normal’ page (the ! means it won’t be sent through; for more info on the proxypass directive  [see the Apache official documentation](http://httpd.apache.org/docs/2.2/mod/mod_proxy.html#proxypass "Apache2 ProxyPass Directive")). This directive is necessary because in the next line we pass  _all_  requests through to localhost:8000 running our web app which means if it crashed, the 503 “service unavailable” custom error page would not be reachable as it will be passing the request for /maintenance.html to the web server which is down.

### Security Heads Up

Couple things security related here, in case they are not obvious. Firstly, if you’re intending users to only use your app via the reverse proxy, ensure that your firewall is blocking outside requests from going directly to the web app! Make it only accessible to your web server and nothing else as otherwise your users will be able to potentially mess with your application by accessing it directly (which may be a case you forgot to account for in your application’s security, for instance)

Moreover, for your web application’s logging purposes, please understand that you will not be able to make use of any kind of ‘remote IP’ detection by literally looking at the remote IP. Your web app will receive all its requests from the reverse proxy, therefore the remote IP will  _always_  be the reverse proxy’s IP (e.g. 127.0.0.1 in our example) and not the actual client. To get around this, proxy servers by default append a header to their request called “[X-Forwarded-For](http://en.wikipedia.org/wiki/X-Forwarded-For "Wikipedia: X-Forwarded-For Header")” which is a comma separated list of IP’s for which the request was proxied. According to wikipedia it takes the form:

```
X-Forwarded-For: client, proxy1, proxy2
```

However, in practice it may be the reverse order (at least for me it was, so do some experimenting to see how your web server handles it). Your web application can read the header and get the client’s IP from there.

### Static front-end assets

For anything more than basic applications you will want to serve your front-end assets via the web server directly. This is advantageous because you don’t bog down the web application by making it do menial tasks like serving the static content and you also reduce latency for getting those resources. To do this, simply change your ProxyPass directive from the vhost configuration to read something like:

```nginx
ProxyPass /api http://localhost:8000/
```

Now all requests sent to /api will be instead re-routed to the web app, but all other requests will go through to the usual document root. Note that Apache will automatically reformat the URL; that is if the user requests /api/a/b/c then the web app will only receive the request for /a/b/c without the /api part as that is not mentioned in the ProxyPass directive for the 2nd parameter. Very useful!

### Conclusion

Web apps and Apache work together quite well and you can accomplish a lot by harnessing their strengths together. I’d recommend using this configuration or at least something similar for anything more than a local or development server. I hope this guide has helped you get set up for your next web project.

### NGINX Configuration

For those of you who prefer using NGINX, here’s a comparable configuration to what I had for Apache above.

```nginx
server {
  listen 80;
  server_tokens off;
  server_name pastebin.mydomain.com;
  error_page 502 /500.html;
  location /500.html {
    root /home/apps/default;
    allow all;
    internal;
  }
  location / {
    proxy_pass http://localhost:8000;
    proxy_set_header  X-Forwarded-for $remote_addr;
  }
}
```
