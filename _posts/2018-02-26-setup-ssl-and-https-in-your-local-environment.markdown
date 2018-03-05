---
layout:           post
title:            "Create a wildcard SSL certificate for your local development environment."
date:             2018-02-26 19:55:00 +0900
last_modified_at: 2018-02-26 20:35:00 +0900
tags:             [ssl, https, local, development, environment, keychain, openssl, san, google, chrome, virtualbox, vm, docker]
introduction:     "Google Chrome 63 is now forcing HTTPS and SSL for .dev top-level domains, so it's time to setup support for HTTPS in your local development environment and here are some instructions to get you going quickly."
---

I mostly use Docker in of my local and cloud environments but I also work on projects that aren't quite there yet and still running in VirtualBox VMs. One of them needed some tweaks and after opening it up after having left it untouched for a long while I noticed that it needed some tweaks since Google Chrome now forces .dev TLDs to use HTTPS and this app runs on that top-level domain. I wrote more about why I didn't switch to .localhost but instead decided to install a self-signed certificate in this [blog](/2018/02/26/chrome-does-not-resolve-localhost-tld.html).

I tried a bunch of SSL certificate generators on the internet but couldn't find any that gave me the beautiful green padlock icon in Google Chrome. The same happened when I tried generating certificates with the OpenSSL command on my command line. It took quite a lot of time doing trial and error until I finally found the right arguments to satisfy Google Chrome's standards to get the "Secure" display next to the URL in the address bar.

Since I have a bunch of custom subdomains on my domain for this project I wanted to take it one step further and creating a wildcard SSL certificate that I can use with all of those subdomains instead of having one certificate per subdomain. I also didn't want to worry about the certificate expiring in a year so I put the validity period to 10 years.

This is the final argument that it all boiled down to, enjoy!
{% highlight plaintext %}
openssl req \
    -newkey rsa:2048 \
    -x509 \
    -nodes \
    -keyout ~/Desktop/my-domain.dev.key \
    -new \
    -out ~/Desktop/my-domain.dev.crt \
    -reqexts SAN \
    -extensions SAN \
    -config <(cat /System/Library/OpenSSL/openssl.cnf \
        <(printf '
[req]
default_bits = 2048
prompt = no
default_md = sha256
x509_extensions = v3_req
distinguished_name = dn

[dn]
C = JP
ST = Tokyo
L = Tokyo
O = MyDomain Inc.
OU = Technology Group
emailAddress = hello@my-domain.jp
CN = my-domain.dev

[v3_req]
subjectAltName = @alt_names

[SAN]
subjectAltName = @alt_names

[alt_names]
DNS.1 = *.my-domain.dev
DNS.2 = my-domain.dev
')) \
    -sha256 \
    -days 3650
{% endhighlight %}

Now, import the generated certificate (my-domain.dev.crt) into your System Keychain in your Keychain Access app by selecting "Import Items" from the File menu or use the command line and run the following command: `sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/Desktop/my-domain.dev.crt`.

To make this self-signed SSL certificate valid you have to right-click and select "Get Info" or simply double-click on it Keychain Access to open up the details of the certificate. Next, select "Always Trust" to make this certificate trusted by your apps. Close and input your user's password and you should be good to go.

The final step will be to setup your web server to use this newly generated private key and certificate, I will leave it up to you to google for instructions on how to set it up for your web server.

I hope that this was informative and that it will save someone out there a couple of hours of frustration trying to get this to work.
