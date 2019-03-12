---
layout: post
title:  "There is life after TLS-SNI-01 deprecation"
date:   2019-03-11 08:39:58 -0300
categories: Tutorials
permalink: /:categories/:title
---
Some people have gone crazy when Let's Encrypt announced the deprecation of the TLS-SNI-01 domain validation method. This post is about my reaction to this news.

I've been a **certbot** user since ever and never realized that the default method for SSL-only domain validation [was going to reach it's EOL](https://community.letsencrypt.org/t/march-13-2019-end-of-life-for-all-tls-sni-01-validation-support/74209). Because there wasn't a viable way of using HTTP-01 or DNS-01 challenges, a decision was made to adopt the newly implemented TLS-ALPN-01 method.

**ALPN** is a next generation protocol of negotiating TLS handshakes. It makes less roundtrips on the certificate negotiation phase, and connection can be handled earlier by reverse proxies.

Certbot wasn't going to cut it, so I rolled up my sleeves and began reading - until I found [an article talking about dehydrated](https://medium.com/@decrocksam/deploying-lets-encrypt-certificates-using-tls-alpn-01-https-18b9b1e05edf). I just had to run a python script that responded the ALPN challenges correctly and badabim-badabum certificates were issued. It is nothing like just running certbot, but I was adopting one of the most secure and fast validation methods.

Basically what [Sam Decrock](https://medium.com/@decrocksam) says is that I have to stop my NGINX and start the ALPN challenges script - **every time** I needed to renew a certificate. This is a **bad idea** in a production environment, so I had to adapt those instructions into a more reliable solution.

I came out with something using a [NGINX module called ngx_stream_ssl_preread_module](http://nginx.org/en/docs/stream/ngx_stream_ssl_preread_module.html). With it you can use your proxy as a TCP/UDP traffic intercepter that prereads your SSL connections, enabling redirection based on the **ALPN protocol list** sent by the client. So now I'm able to redirect ACME-TLS requests through **dehydrated** and all other SSL connections to **port 4433** back to the server:

```nginx
# /etc/nginx/nginx.conf
# Bottom of file, don't erase previous configuration.
stream {
        map $ssl_preread_alpn_protocols $tls_port {
                ~\bacme-tls/1\b 10443; # If it is a challenge use python script port
                default 4433; # Default behavior is to redirect to port 4433
        }
        server {
                listen 443;
                listen [::]:443;
                proxy_pass 127.0.0.1:$tls_port; # Goes back to the server in the correspondent port
                ssl_preread on;
        }
}
```

This way [I can keep the default setup on dehydrated python script](https://github.com/lukas2511/dehydrated/blob/master/docs/tls-alpn.md), and just run it on my server with:

```shell
$ sudo /path/to/dehydrated_script.py
```

Then after that I had to **change all SSL server blocks** to listen to that non-default port and *voilà*, no need to stop or restart anything:

```nginx
# /etc/nginx/sites-available/yourdomain.com
server {
    # other stuff...
    listen 4433; # Instead of => listen 443;
    listen [::]:4433; # Instead of => listen [::]:443;
    # other stuff...
}
```

To generate your certificates with TLS-ALPN just clone **dehydrated** somewhere and configure it to deploy keys using that method:
```config
# /etc/dehydrated/config
CHALLENGETYPE="tls-alpn-01"
ALPNCERTDIR="/etc/dehydrated/alpn-certs"
CERTDIR="/etc/nginx/certs"
```

Then, [add your domains and subdomains](https://github.com/lukas2511/dehydrated/blob/master/docs/domains_txt.md) into `domains.txt` and turn dehydrated on:

```shell
$ /path/to/dehydrated -c -f /etc/dehydrated/config
```

Your keys will live under `/etc/nginx/certs`, just [configure your server blocks to use them](https://www.digicert.com/csr-ssl-installation/nginx-openssl.htm#step_4) and that's it.

## Making things easier on Ubuntu

This proccess is a little manual, but you can make it more automatic. First, add dehydrated as a **cronjob** by running `$ sudo crontab -e` and adding this to the end of the file:

```shell
15 3 * * * /path/to/dehydrated -c -f /etc/dehydrated/config
```

Then, to *avoid* your ALPN validation script **not** being executed, add this to `/etc/init/alpn-script.conf` (Use `/etc/systemd/alpn-script.conf` in Ubuntu 15.x):

```conf
# /etc/init/alpn-script.conf
start on runlevel [2345]
stop on runlevel [!2345]

exec /path/to/dehydrated_script.py
```

This way your server can start your ALPN validation script on boot and use administrative commands like:

```shell
$ sudo service alpn-script start
$ sudo service alpn-script stop
```

If by any chance you have to renew certificates with dehydrated and your ALPN script is not running all challenges will fail.

## Known Caveat
There is something you have to keep in mind: Since your server is redirecting traffic to itself, the NGINX variable `$binary_remote_addr` will always be your own IP. So you if there is any configuration depending on it (like rate limiting or upstreams) [you should use another method to do it](https://ypereirareis.github.io/blog/2017/02/15/nginx-real-ip-behind-nginx-reverse-proxy/).

What do you think about the solution, do you have any doubts, comments or you can help me make a better tutorial? Reach out on the comments below, I'll be glad to discuss it with you.
