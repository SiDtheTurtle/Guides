\# Setting up local hostnames for home servers on Unifi networks using Nginx Reverse Proxy



\## Preamble

I run a Docker server locally which hosts many services such as Plex for watching shows, and Home Assistant for home automation. Because they are both on the same host metal they share the same # Setting up local hostnames for home servers on Unifi networks using Nginx Reverse Proxy



\## Preamble

I run a Docker server locally which hosts many services such as Plex for watching shows, and Home Assistant for home automation. Because they are both on the same host metal they share the same hostname, but just use different ports to access. For example to access Plex in a browser I would go to `\[hostname]:32400`, whereas for Home Assistant I'd use `\[hostname]:8123`.



This is all well and good, but means you need to bookmark or memorise the ports, and it's not family friendly if I want someone to be able to access the services without pestering me.



There are guides on how to do this, most feature using a combination of a Raspberry Pi with PiHole + a reverse proxy, and having to update your router's DHCP server to switch the DNS server for all clients to the PiHole. This fails my family-friendly test as if the Raspberry Pi ever goes down (because I tinkered, because there was a power cut and the Raspberry Pi didn't come on by itself etc.), DNS goes down and I will start getting complaints that 'the Internet is down'.



I run a Unifi-based network, which allows me to enter my own DNS records. This, in combination with a reverse proxy, is all you need to make this work. No Raspberry Pi or other hardware is required. This guide details how to do this as simply as possible. It might seem wordy but I hate guides where they assume you know where to find a specific button, so I've tried to be as prescriptive as possible.



If you don't have a Unifi network, you might still be able to make use of this guide, except you will need to figure out for yourself how to edit DNS records for your network.



This guide is correct as of September 2025.



\## My Requirements



\- If I go to `plex.home`, it should take me to Plex. If I go to `ha.home`, it should take me to Home Assistant.

\- If any component in the solution stops working, client devices should continue to operate and be able to access the home servers, and standard Internet name resolution should still work.



\## Assumptions

\- You are using Unifi as your home network infrastructure and have the ability to log in to the control panel.

\- You are familiar with Docker.

\- You are familiar with networking concepts like DNS records.

\- The new domain names will only be used by people navigating to the services. You \_can\_ use these new domain names in configuration files etc, but then those service will fail if the proxy server ever goes offline. I prefer to keep everything configured to use IP and port or the real server hostname and port.



\# Steps



\## Overview

1\. Configure DNS entries in the Unifi control panel.

2\. Set up Nginx Proxy Manager using Docker.

3\. Configure proxy hosts in Nginx Proxy Manager.

4\. Deal with any app specific issues with reverse proxies.



This guide assumes that you want to use `.home` as the TLD for your services, such that you could have `photos.home`, `plex.home` and so on. You could use anything in theory, though I'd advise sticking to reserved TLDs like `.home` and `.local` in case you 'collide' with an actual web address.



\## 1. Configure DNS entries in the Unifi control panel

We're going to set up a DNS 'A record', which tells Unifi how to translate a hostname into an IP, then a DNS CNAME record for each service, which redirects from the desired subdomains to the primary domain specified in the A record. You could do this entirely with A records and no CNAME records, but you would have to update \_all\_ the A records if the server IP ever changed- this way you only have to change one. Once this step is complete, you can go to the domains you've set up, but it'll only take you to whatever is hosted on the default Internet port, 80.



1\. Log in to the UniFi console.

2\. Go to the settings cog, then `Policy Engine` > `Policy Table`:



!\[A screenshot of where to find the policy engine button](/Screenshots/01\_Policy\_Engine.png)



3\. Click on `Create New Policy`.



!\[A screenshot of where to find the create new policy button](/Screenshots/02\_Create\_New\_policy.png)



4\. Select the radio button `DNS` then enter the information as follows, then click `Add`:

&nbsp;   - Type: `Host (A)`.

&nbsp;   - Domain Name: `home.home` (this will be the 'root' domain that everything redirects to, it could be anything but should end with .home or your chosen TLD).

&nbsp;   - IP Address: `\[the IP address of the server hosting your services]`.

&nbsp;   - TTL: `Auto`.

&nbsp;

!\[A screenshot showing how to add an A record in Unifi](/Screenshots/04\_Create\_New\_Policy\_CNAME\_Record.png)



5\. Click on `Creat New Policy` again, select `DNS` again and enter the information as follows, then click `Add`:

&nbsp;   - Type: `Alias (CNAME)`.

&nbsp;   - Alias Domain Name: `home.home` (or whatever you specified in step 4).

&nbsp;   - Target Domain Name: `plex.home` (or whatever service you're looking to redirect to).

&nbsp;   - TTL: `Auto`.



!\[A screenshot showing how to add a CNAME record in Unifi](/Screenshots/04\_Create\_New\_Policy\_CNAME\_Record.png)



6\. Repeat step 5 for every service you want to add. Here's an example subset of mine:



!\[A screenshot showing example CNAME record entries in Unifi](/Screenshots/05\_Example\_Policies.png)



Now if you go to any of the domains you set up, you'll bounce to whatever's hosted on that server at port 80, rather than where you'd hope. To get this working, we now need a reverse proxy.



\## 2. Set up Nginx Proxy Manager using Docker

This service packages up a proxy server in an idiot-proof UI, and runs on Docker. The steps are relatively simple:

1\. First, make sure you have nothing running on port 80 on your Docker server. If you have, update your `docker-compose.yml` file to remap the port. Don't use 81 as Nginx Proxy Manager uses it. We will fix this later so you don't have to worry that it's not on the default port. For example, I have the 'Homepage' app set up on port 80, so I edited the `docker-compose.yml` file to remap the port to 82:



!\[A screenshot of the edits required to a Docker compose file to remap port 80](/Screenshots/06\_VI\_Example.png)



2\. Follow the install guide for Nginx Proxy Manager found here: https://nginxproxymanager.com/guide/#quick-setup. You can copy/paste the suggested `docker-compose.yml` and it'll work out of the box. Personally in addition I gave the service a name so it's easier to find when doing a `docker ps`, and used real volumes over a bind mount. If you don't know what that means, ignore me.

3\. You should now have a running instance of Nginx Proxy Manager. If you go to `home.home`, you should get a holding page:



!\[A screenshot of the Nginx Proxy Manager holding page](/Screenshots/07\_Holding\_Page.png)



\## 3. Configure proxy hosts in Nginx Proxy Manager

Now we get into the last main step, of creating the proxy hosts so that everything redirects:

1\. Edit the URL to go to `home.home:81`. You should see a login screen like this:



!\[A screnshot of the Nginx Proxy Manager login page](/Screenshots/08\_Proxy\_Login.png)



2\. Enter the email address as `admin@example.com` and the password as `changeme` and press `Sign In`. You will immediately be asked to reset that login to something more secure - do so.

3\. You will now find yourself on the home screen. Click on `Proxy Hosts`:



!\[A screenshot of where to find the proxy hosts menu in Nginx Proxy Manager](/Screenshots/09\_Proxy\_Hosts.png)



4\. Click on `Add Proxy Host`:



!\[A screenshot showing where the button is for adding a proxy host in Nginx Proxy Manager](/Screenshots/10\_Add\_Proxy\_Host.png)



5\. Enter the information as follows (ignore anything not specified) then click `Save`:

&nbsp;   - Domain Names: `plex.home` (or whatever service you want to map to).

&nbsp;   - Scheme: `http`.

&nbsp;   - Forward Hostname / IP: `home.home` (or whatever you set up as your A record).

&nbsp;   - Forward Port: `32400` (or whatever the port number is for your service, this one's the default port for Plex).



!\[A screenshot showing how to add a proxy host in Nginx Proxy Manager](/Screenshots/11\_Proxy\_Host\_Entry.png)



6\. Now in a new browser tab, go to the URL you just created (note if you are following my example, your browser might helpfully redirect you to its search engine rather than navigate to your URL, in which case specifiy `http://` at the start. Once you do this once, you should be fine in future). You should be at the homepage of your service!



7\. Repeat steps 4 - 7 for any other domains you want to set up. Note some apps, like Home Assistant, don't like reverse proxies out of the box, and need additional configuration. Some examples I've encountered are listed in the section below.

&nbsp;  

\# 4. Deal with any app specific issues with reverse proxies



\## General 502 Bad Gateway Error

If you see this:



!\[A screenshot of a gateway error](/Screenshots/12\_Bad\_Gateway\_Error.png)



It means the reverse proxy has gone to the port you specified, but there's nothing running there. Check you've specified the correct port, and that your service is running on it.



\## Home Assistant

If you try to go to your configured Home Assistant domain, such as `ha.home`, you'll be greeted with the error `400: Bad Request`, and the following entry in your logs:



!\[A screenshot from Home Assistant's logs showing a blocked request](/Screenshots/13\_HA\_Log\_Error.png)



Note in my case the address range it's complaining about is not that of my local network or client machine, I assume this is some internal IP mapping the reverse proxy is doing.



To resolve, we need to edit the `configuration.yml` file for Home Assistant, and change a setting in Nginx Proxy Manager.



\### Changes to configuration.yml

1\. Edit your `configuration.yml` according to your own Home Assistant configuration. For example as I am running HA Core, I have to SSH into the docker instance and edit it using VI.

2\. Follow the steps here: https://www.home-assistant.io/integrations/http#reverse-proxies. Note for the IP to specify, I tried using the known IP of my server, but I still kept getting the error. Instead I entered the 172 address I saw in the error logs, and it's working!

3\. Validate your changes are valid using configuration checker, then restart Home Assistant:



!\[A screenshot showing where in Home Assistant to reload your configuration yaml](/Screenshots/14\_HA\_Config.png)



If you try again, you'll get a different error:



!\[A screenshot showing the second Home Assistant error](/Screenshots/15\_HA\_Second\_Error.png)



Follow on to the next section to resolve.



\### Enable Websockets Support in Nginx Proxy Manager

1\. Log in to Nginx Proxy Manager and go to the `Proxy Hosts` screen where we set up all the domains.

2\. Click on the ellipsis next to the Home Assistant entry then click `Edit`:



!\[A screenshot showing how to edit a Proxy Host in Nginx Proxy Manager](/Screenshots/16\_HA\_Web\_Sockets\_Menu.png)



3\. Turn on the toggle for `Websockets Support` and click `Save`:



!\[A screenshot showing the websockets toggle in Nginx Proxy Manager](/Screenshots/17\_HA\_Web\_Sockets\_Setting.png)



You should now be able to go to `ha.home` and correctly see the Home Assistant homepage.



\## Immich

You will get a generic bad gateway error. Enable web sockets as per the guide for Home Assistant above.



\## qBittorrent

You get a very simple error of `Unauthorized`. A small change is required in the qBittorrent client:



1\. Navigate to the qBittorrent client interface using the classic hostname and port and log in.

2\. Go to `Settings` > `WebUI`> `Security`, uncheck `Enable Cross-Site Request Forgery (CSRF) protection` and click `Save`.



!\[A screenshot showing how to natigate to the correct settings in qBittorrent](/Screenshots/18\_QBT.png)



\## Zigbee2MQTT

At first glance it might look like Zizbee2MQTT is working out of the box, but you'll quickly see all your devices are missing!



!\[A screenshot of the weird behaviour in Zigbee2MQTT](/Screenshots/19\_Z2MQTT.png)



To resolve, enable web sockets as per the guide for Home Assistant above.

hostname, but just use different ports to access. For example to access Plex in a browser I would go to `\[hostname]:32400`, whereas for Home Assistant I'd use `\[hostname]:8123`.



This is all well and good, but means you need to bookmark or memorise the ports, and it's not family friendly if I want someone to be able to access the services without pestering me.



There are guides on how to do this, most feature using a combination of a Raspberry Pi with PiHole + a reverse proxy, and having to update your router's DHCP server to switch the DNS server for all clients to the PiHole. This fails my family-friendly test as if the Raspberry Pi ever goes down (because I tinkered, because there was a power cut and the Raspberry Pi didn't come on by itself etc.), DNS goes down and I will start getting complaints that 'the Internet is down'.



I run a Unifi-based network, which allows me to enter my own DNS records. This, in combination with a reverse proxy, is all you need to make this work. No Raspberry Pi or other hardware is required. This guide details how to do this as simply as possible. It might seem wordy but I hate guides where they assume you know where to find a specific button, so I've tried to be as prescriptive as possible.



If you don't have a Unifi network, you might still be able to make use of this guide, except you will need to figure out for yourself how to edit DNS records for your network.



This guide is correct as of September 2025.



\## My Requirements



\- If I go to `plex.home`, it should take me to Plex. If I go to `ha.home`, it should take me to Home Assistant.

\- If any component in the solution stops working, client devices should continue to operate and be able to access the home servers, and standard Internet name resolution should still work.



\## Assumptions

\- You are using Unifi as your home network infrastructure and have the ability to log in to the control panel.

\- You are familiar with Docker.

\- You are familiar with networking concepts like DNS records.

\- The new domain names will only be used by people navigating to the services. You \_can\_ use these new domain names in configuration files etc, but then those service will fail if the proxy server ever goes offline. I prefer to keep everything configured to use IP and port or the real server hostname and port.



\# Steps



\## Overview

1\. Configure DNS entries in the Unifi control panel.

2\. Set up Nginx Proxy Manager using Docker.

3\. Configure proxy hosts in Nginx Proxy Manager.

4\. Deal with any app specific issues with reverse proxies.



This guide assumes that you want to use `.home` as the TLD for your services, such that you could have `photos.home`, `plex.home` and so on. You could use anything in theory, though I'd advise sticking to reserved TLDs like `.home` and `.local` in case you 'collide' with an actual web address.



\## 1. Configure DNS entries in the Unifi control panel

We're going to set up a DNS 'A record', which tells Unifi how to translate a hostname into an IP, then a DNS CNAME record for each service, which redirects from the desired subdomains to the primary domain specified in the A record. You could do this entirely with A records and no CNAME records, but you would have to update \_all\_ the A records if the server IP ever changed- this way you only have to change one. Once this step is complete, you can go to the domains you've set up, but it'll only take you to whatever is hosted on the default Internet port, 80.



1\. Log in to the UniFi console.

2\. Go to the settings cog, then `Policy Engine` > `Policy Table`:



!\[A screenshot of where to find the policy engine button](/Screenshots/01\_Policy\_Engine.png)



3\. Click on `Create New Policy`.



!\[A screenshot of where to find the create new policy button](/Screenshots/02\_Create\_New\_policy.png)



4\. Select the radio button `DNS` then enter the information as follows, then click `Add`:

&nbsp;   - Type: `Host (A)`.

&nbsp;   - Domain Name: `home.home` (this will be the 'root' domain that everything redirects to, it could be anything but should end with .home or your chosen TLD).

&nbsp;   - IP Address: `\[the IP address of the server hosting your services]`.

&nbsp;   - TTL: `Auto`.

&nbsp;

!\[A screenshot showing how to add an A record in Unifi](/Screenshots/04\_Create\_New\_Policy\_CNAME\_Record.png)



5\. Click on `Creat New Policy` again, select `DNS` again and enter the information as follows, then click `Add`:

&nbsp;   - Type: `Alias (CNAME)`.

&nbsp;   - Alias Domain Name: `home.home` (or whatever you specified in step 4).

&nbsp;   - Target Domain Name: `plex.home` (or whatever service you're looking to redirect to).

&nbsp;   - TTL: `Auto`.



!\[A screenshot showing how to add a CNAME record in Unifi](/Screenshots/04\_Create\_New\_Policy\_CNAME\_Record.png)



6\. Repeat step 5 for every service you want to add. Here's an example subset of mine:



!\[A screenshot showing example CNAME record entries in Unifi](/Screenshots/05\_Example\_Policies.png)



Now if you go to any of the domains you set up, you'll bounce to whatever's hosted on that server at port 80, rather than where you'd hope. To get this working, we now need a reverse proxy.



\## 2. Set up Nginx Proxy Manager using Docker

This service packages up a proxy server in an idiot-proof UI, and runs on Docker. The steps are relatively simple:

1\. First, make sure you have nothing running on port 80 on your Docker server. If you have, update your `docker-compose.yml` file to remap the port. Don't use 81 as Nginx Proxy Manager uses it. We will fix this later so you don't have to worry that it's not on the default port. For example, I have the 'Homepage' app set up on port 80, so I edited the `docker-compose.yml` file to remap the port to 82:



!\[A screenshot of the edits required to a Docker compose file to remap port 80](/Screenshots/06\_VI\_Example.png)



2\. Follow the install guide for Nginx Proxy Manager found here: https://nginxproxymanager.com/guide/#quick-setup. You can copy/paste the suggested `docker-compose.yml` and it'll work out of the box. Personally in addition I gave the service a name so it's easier to find when doing a `docker ps`, and used real volumes over a bind mount. If you don't know what that means, ignore me.

3\. You should now have a running instance of Nginx Proxy Manager. If you go to `home.home`, you should get a holding page:



!\[A screenshot of the Nginx Proxy Manager holding page](/Screenshots/07\_Holding\_Page.png)



\## 3. Configure proxy hosts in Nginx Proxy Manager

Now we get into the last main step, of creating the proxy hosts so that everything redirects:

1\. Edit the URL to go to `home.home:81`. You should see a login screen like this:



!\[A screnshot of the Nginx Proxy Manager login page](/Screenshots/08\_Proxy\_Login.png)



2\. Enter the email address as `admin@example.com` and the password as `changeme` and press `Sign In`. You will immediately be asked to reset that login to something more secure - do so.

3\. You will now find yourself on the home screen. Click on `Proxy Hosts`:



!\[A screenshot of where to find the proxy hosts menu in Nginx Proxy Manager](/Screenshots/09\_Proxy\_Hosts.png)



4\. Click on `Add Proxy Host`:



!\[A screenshot showing where the button is for adding a proxy host in Nginx Proxy Manager](/Screenshots/10\_Add\_Proxy\_Host.png)



5\. Enter the information as follows (ignore anything not specified) then click `Save`:

&nbsp;   - Domain Names: `plex.home` (or whatever service you want to map to).

&nbsp;   - Scheme: `http`.

&nbsp;   - Forward Hostname / IP: `home.home` (or whatever you set up as your A record).

&nbsp;   - Forward Port: `32400` (or whatever the port number is for your service, this one's the default port for Plex).



!\[A screenshot showing how to add a proxy host in Nginx Proxy Manager](/Screenshots/11\_Proxy\_Host\_Entry.png)



6\. Now in a new browser tab, go to the URL you just created (note if you are following my example, your browser might helpfully redirect you to its search engine rather than navigate to your URL, in which case specifiy `http://` at the start. Once you do this once, you should be fine in future). You should be at the homepage of your service!



7\. Repeat steps 4 - 7 for any other domains you want to set up. Note some apps, like Home Assistant, don't like reverse proxies out of the box, and need additional configuration. Some examples I've encountered are listed in the section below.

&nbsp;  

\# 4. Deal with any app specific issues with reverse proxies



\## General 502 Bad Gateway Error

If you see this:



!\[A screenshot of a gateway error](/Screenshots/12\_Bad\_Gateway\_Error.png)



It means the reverse proxy has gone to the port you specified, but there's nothing running there. Check you've specified the correct port, and that your service is running on it.



\## Home Assistant

If you try to go to your configured Home Assistant domain, such as `ha.home`, you'll be greeted with the error `400: Bad Request`, and the following entry in your logs:



!\[A screenshot from Home Assistant's logs showing a blocked request](/Screenshots/13\_HA\_Log\_Error.png)



Note in my case the address range it's complaining about is not that of my local network or client machine, I assume this is some internal IP mapping the reverse proxy is doing.



To resolve, we need to edit the `configuration.yml` file for Home Assistant, and change a setting in Nginx Proxy Manager.



\### Changes to configuration.yml

1\. Edit your `configuration.yml` according to your own Home Assistant configuration. For example as I am running HA Core, I have to SSH into the docker instance and edit it using VI.

2\. Follow the steps here: https://www.home-assistant.io/integrations/http#reverse-proxies. Note for the IP to specify, I tried using the known IP of my server, but I still kept getting the error. Instead I entered the 172 address I saw in the error logs, and it's working!

3\. Validate your changes are valid using configuration checker, then restart Home Assistant:



!\[A screenshot showing where in Home Assistant to reload your configuration yaml](/Screenshots/14\_HA\_Config.png)



If you try again, you'll get a different error:



!\[A screenshot showing the second Home Assistant error](/Screenshots/15\_HA\_Second\_Error.png)



Follow on to the next section to resolve.



\### Enable Websockets Support in Nginx Proxy Manager

1\. Log in to Nginx Proxy Manager and go to the `Proxy Hosts` screen where we set up all the domains.

2\. Click on the ellipsis next to the Home Assistant entry then click `Edit`:



!\[A screenshot showing how to edit a Proxy Host in Nginx Proxy Manager](/Screenshots/16\_HA\_Web\_Sockets\_Menu.png)



3\. Turn on the toggle for `Websockets Support` and click `Save`:



!\[A screenshot showing the websockets toggle in Nginx Proxy Manager](/Screenshots/17\_HA\_Web\_Sockets\_Setting.png)



You should now be able to go to `ha.home` and correctly see the Home Assistant homepage.



\## Immich

You will get a generic bad gateway error. Enable web sockets as per the guide for Home Assistant above.



\## qBittorrent

You get a very simple error of `Unauthorized`. A small change is required in the qBittorrent client:



1\. Navigate to the qBittorrent client interface using the classic hostname and port and log in.

2\. Go to `Settings` > `WebUI`> `Security`, uncheck `Enable Cross-Site Request Forgery (CSRF) protection` and click `Save`.



!\[A screenshot showing how to natigate to the correct settings in qBittorrent](/Screenshots/18\_QBT.png)



\## Zigbee2MQTT

At first glance it might look like Zizbee2MQTT is working out of the box, but you'll quickly see all your devices are missing!



!\[A screenshot of the weird behaviour in Zigbee2MQTT](/Screenshots/19\_Z2MQTT.png)



To resolve, enable web sockets as per the guide for Home Assistant above.



