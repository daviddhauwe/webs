# Secure and fast acces to your Phoenix Contact Webvisit PLC visualisation with help of a Raspberry Pi

![screenshot](screenshot.png#center)

At home, I have a webvisit visualisation on my phoenix contact plc. Viewing this secure and fast from anywhere and on any device is not so easy. I used to have a vpn connection to my home network on my pc and smartphone. Over this vpn tunnel, I browsed to the html5 webpage or used the microbrowser app. Connecting the vpn or leave the connection open is not so convenient, certainly not on a mobile device. Also the html5 webpage is rather slow.

So it's time for another solution which must be secure, fast and above all user friendly!
The solution is an apache webserver running on a Raspberry Pi:

- apache serves the static webvisit pages and allows the client to cache them
- reverse proxy is used to get the dynamic content from the plc
- users are authenticated before they can see anything
- everyting is encrypted using ssl
- dynamic dns is used to be able to contact your router from anywhere

Using above solution on a mobile device, it's almost as fast as the microbrowser app, with all the security included.
Below, I tried to explain step-by-step how to do this. My PLC has ip 192.168.0.10 if your's is different, do not forget to change this!

If you have an older plc which does not support html5 (only java-applet, like the ILC 170) it is also possible to use html5, as the pages are served by apache. Browsing the plc website will no longer work, but it is still reachable with microbrowser. If you still want to use the PLC website, you could put the html5 files directly on the Pi and skip the copy step as described below.

## What you need

- a plc with a working webvisit visualisation
- a Raspberry Pi with Raspbian Stretch (lite is sufficient)
- some tools to access your Pi like [putty](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) and [winscp](https://winscp.net). By right clicking, you can paste the commands into putty.

## Install Apache

Update your Pi

```bash
sudo apt-get update
sudo apt-get dist-upgrade
```

Install apache2

```bash
sudo apt-get install apache2 -y
```

Test if the webserver is working by browsing to the Pi's IP address, you should see the standard test page.

## Copy the static webpages from the PLC

Make a script that copies the files from the PLC's ftp server to the Pi

```bash
nano /home/pi/webs-sync.sh
```

And copy following content in it (change your plc ip)

```bash
#!/bin/sh
# sync webpages from plc to apache webserver
wget ftp://192.168.0.10/webs/* -P /var/www/webs/ -N -nv
```

Make the script executable

```bash
chmod +x /home/pi/webs-sync.sh
```

Execute the script to copy the files

```bash
/home/pi/webs-sync.sh
```

Setup a cron script to copy these files daily so every change on the PLC is updated

```bash
crontab -e
```

And add following lines

```bash
# sync plc webs every day at 6h
0 6 * * * /home/pi/webs-sync.sh
```

## Turn the PLC website into an app

In chrome for mobile, you can use the add to start screen functionality, which adds an icon for your webpage to the start screen and gives it an app like look.
I used [RealFaviconGenerator](https://realfavicongenerator.net) to generate the necessary icons and config files. You can download them with below command. If you want to make yours, go ahead, but don't forget to remove the trailing / in the browserconfig.xml and site.webmanifest.

Copy the icons and related files

```bash
wget -q -O - https://api.github.com/repos/daviddhauwe/webs/tarball/ | tar xz --strip-components=2 -C /var/www/webs/
```

Copy entry.html to index.shtml and make some canges to it. This file is used by apache.

```bash
cp /var/www/webs/entry.html /var/www/webs/index.shtml
nano /var/www/webs/index.shtml
```

Add the following in the `<head>` section, the `initial-scale=1.7` might be adjusted to fit yours

```html
<!-- additions -->
<link rel="apple-touch-icon" sizes="180x180" href="apple-touch-icon.png">
<link rel="icon" type="image/png" sizes="32x32" href="favicon-32x32.png">
<link rel="icon" type="image/png" sizes="16x16" href="favicon-16x16.png">
<link rel="manifest" href="site.webmanifest">
<link rel="mask-icon" href="safari-pinned-tab.svg" color="#5bbad5">
<meta name="msapplication-TileColor" content="#da532c">
<meta name="theme-color" content="#ffffff">
<!-- on mobile devices, initial zoom to fit content, might need some tuning -->
<meta name="viewport" content="width=device-width, initial-scale=1.7, shrink-to-fit=yes">
<!-- end additions -->
```

Change the cgi-bin root path which is used by the html5 page (hmi.js). It is also possible to change this in webvisit project configurations - runtime configurations. But this would break the functionality directly on the PLC.

```javascript
CGI_RELATIVTOROOT: "webs/", //adapted
```

Some lines later, afther the last `SpiderControl.LoadView()` add this to let webvisit know the logged in user

```javascript
//pass httpd_username to container variable
SpiderControl.WriteContainer('httpd_username','<!--#echo var="REMOTE_USER" -->');
```

## Configure apache

Make the website available under /var/www/html by creating a symlink to it

```bash
sudo ln -s /var/www/webs /var/www/html/webs
```

Enable the necessary modules in apache

```bash
sudo a2enmod proxy proxy_http headers auth_form request session_cookie session_crypto ssl expires include
```

Make a new site configuration

```bash
sudo nano /etc/apache2/sites-available/webs.conf
```

And paste following code into it (adjust your plc ip)

```bash
#webs config
<Location "/webs">
    #fix for already gzip'ed svgz files
    SetEnvIfNoCase Request_URI "\.(?:svgz)$" no-gzip
    Header set Content-Encoding gzip env=no-gzip
</Location>
#Reverse proxy to PLC : adjust CGI_RELATIVTOROOT: "webs/" in index.shtml
ProxyPass /webs/cgi-bin http://192.168.0.10/cgi-bin ttl=120
```

Activate the new site

```bash
sudo a2ensite webs
```

Test the apache configuration

```bash
sudo apachectl configtest
```

Restart apache to apply the configuration

```bash
sudo apachectl restart
```

Test if it's working by browsing to `http://your-pi-ip/webs` You should see a working website!

You can put a redirect into the root of the webserver, avoiding to the /webs

```bash
sudo nano /var/www/html/index.html
```

And replace it's contents with

```html
<!doctype html>
    <head>
        <meta http-equiv="refresh" content="0; url=webs" />
    </head>
</html>
```

Now it's sufficient to go to `http://your-pi-ip`

## Password protect your site

Now that the Pi is serving your plc's website and the reverse proxy is in place to get the live data, it's time to move on and make some improvements. We will protect the site with a login (forms authentication) and also add caching to speed up loading.

Get the login screen. If you want, you can change the page title

```bash
wget https://github.com/daviddhauwe/webs/blob/master/login.html -P /var/www/webs/
nano /var/www/webs/login.html
```

Get the webs.config file and adjust your plc ip address

```bash
sudo wget https://github.com/daviddhauwe/webs/blob/master/webs.conf -P /etc/apache2/sites-available/
sudo nano /etc/apache2/sites-available/webs.conf
```

Create a password file and add some users, htpasswd will ask for a password

```bash
sudo htpasswd -c /etc/apache2/passwd username1
sudo htpasswd /etc/apache2/passwd username2
```

Encrypt the session cookie with a passphrase, enter some passphrase in this file, it can be difficult, as you don't have to remember it.

```bash
sudo nano /etc/apache2/crypto
```

Check the config and restart apache

```bash
sudo apachectl configtest
sudo apachectl restart
```

Test if it's working by browsing to `http://your-pi-ip/`
You should get a login screen, enter you username and password to get access.

## Secure your site by encryption

We have our site protected by a username and password, so only you can get in. Now we will encrypt the traffic using ssl (https) so nobody can see your password or other data. First we'll make a self-signed certificate to test our setup, which means you'll get a browser warning and you will have to accept the certificate.

Make a self-signed certificate

```bash
sudo make-ssl-cert generate-default-snakeoil --force-overwrite
```

Enable apache ssl site

```bash
sudo a2ensite ssl
```

Check the config and restart apache

```bash
sudo apachectl configtest
sudo apachectl restart
```

Test if ssl is working by browsing to `https://your-pi-ip/`
You should get a certificate warning, which you have to accept.

## Prepare access from outside

Now that everything works and is secured, we can make our site accessible from outside. For that we will need a public dns name. There are a lot of dynamic dns providers out there which also have some client to update your dns record when your public ip changes. Unfortunatelly, most of them are not free any more. I found [Two-DNS](https://www.twodns.de), which is free and reliable. Once you've got a public dns, we can use [Let’s Encrypt](https://letsencrypt.org/) to generate official certificates.

Before we open any ports from the outside, we will tell apache not to show it's version number, as this is bad practice on a public server. So edit security.conf :

```bash
sudo nano /etc/apache2/conf-available/security.conf
```

And uncomment following lines (and comment the counterpart) :

```bash
ServerTokens Prod
ServerSignature Off
```

Check the config and restart apache

```bash
sudo apachectl configtest
sudo apachectl restart
```

Now you can configure your router and forward the ports 80 and 443 to your Pi's ip.

## Get a dynamic dns name

I explain how to use [Two-DNS](https://www.twodns.de), but feel free to choose another and make it working on you own. Create an account and pick a dns name that you like. There ar no automatic update scripts, but the API is quite easy. As my public ip almost never changes, I made a script that run's every day to update the ip to the dns server.

```bash
nano /home/pi/ddns-update.sh
```

Copy following content in it (change to fit your account details). The logger at the end makes an entry in the systemlog every time it runs.

```bash
#!/bin/sh
# dynamic dns update script
curl -X PUT -u "your_email:your_api_token" -d '{"ip_address": "auto"}' https://api.twodns.de/hosts/your_dns_name | logger
```

Make the script executable

```bash
chmod +x /home/pi/ddns-update.sh
```

Execute the script and verify that it works

```bash
/home/pi/ddns-update.sh
```

Setup a cron script to do a daily update

```bash
crontab -e
```

And add following lines

```bash
# update ddns every day at 7h
0 7 * * * /home/pi/ddns-update.sh
```

## Get an official ssl certificate

[Let’s Encrypt](https://letsencrypt.org/) certificates expire after three months, so they must be renewed frequently. Fortunately there is a tool that does the creation and renewal for us. Go to [Certbot](https://certbot.eff.org/lets-encrypt/debianstretch-apache) and follow the instructions, it will setup the certificate, configure apache to use it and setup the renewal.

The wizard will ask you some questions, here are the answers:

- webroot = /var/www/html
- redirect http to https - option 2

When apache is restarted, you can test your site by browsing to `https://your-dns-name`.

If you want, you can verify the ssl configuration with [SSL Labs](https://www.ssllabs.com/ssltest/). It should be grade A as certbot makes the right settings for you.

## Optional automatic login in webvisit

The logged in username is available in the container variable `httpd_username`. You can bypass you normal login mechanism by assiging a userlevel based on the username. This can be done with the event painter EventP_write_EqualAdvanced. You can hide the default login button when the httpd_username container is not empty.

## Done

Congatulations, you can now browse to your PLC's webserver from anywhere in a secure and fast way. When you use chrome on you mobile device, you can use the *add to homescreen* (behind three dots in upper right corner) to add a nice icon which gives direct access to the website.

As we use cache for one month, after changes to you webvisit project, it might be necessary to refresh using shift-reload in your browser. On chrome for mobile,  you should go to settings - site settings - storage, select your site and press the clear button.