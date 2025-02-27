---
title: Configure web server
description: Learn how to configure your web server to work with Varnish.
---

# Configure your web server

Configure your web server to listen on a port other than the default port 80 because Varnish responds directly to incoming HTTP requests, not the web server.

The following sections use port 8080 as an example.

**To change the Apache 2.4 listen port**:

1. Open `/etc/httpd/conf/httpd.conf` in a text editor.
1. Locate the `Listen` directive.
1. Change the value of the listen port to `8080`. (You can use any available listen port.)
1. Save your changes to `httpd.conf` and exit the text editor.

## Modify the Varnish system configuration

To modify the Varnish system configuration:

1. As a user with `root` privileges, open your Vanish configuration file in a text editor:

   - CentOS 6: `/etc/sysconfig/varnish`
   - CentOS 7: `/etc/varnish/varnish.params`
   - Debian: `/etc/default/varnish`
   - Ubuntu: `/etc/default/varnish`

1. Set the Varnish listen port to 80:

   ```conf
   VARNISH_LISTEN_PORT=80
   ```

   For Varnish 4.x, make sure that DAEMON_OPTS contains the correct listening port for the `-a` parameter (even if VARNISH_LISTEN_PORT is set to the correct value):

   ```conf
   DAEMON_OPTS="-a :80 \
      -T localhost:6082 \
      -f /etc/varnish/default.vcl \
      -S /etc/varnish/secret \
      -s malloc,256m"
   ```

1. Save your changes to the Varnish configuration file and exit the text editor.

### Modify the default VCL

This section discusses how to provide minimal configuration so Varnish returns HTTP response headers. This enables you to verify that Varnish works before you configure the [!DNL Commerce] application to use Varnish.

To minimally configure Varnish:

1. Back up `default.vcl`:

   ```bash
   cp /etc/varnish/default.vcl /etc/varnish/default.vcl.bak
   ```

1. Open `/etc/varnish/default.vcl` in a text editor.
1. Locate the following stanza:

   ```conf
   backend default {
       .host = "127.0.0.1";
       .port = "80";
   }
   ```

1. Replace the value of `.host` with the fully qualified hostname or IP address and listen port of the Varnish _backend_ or _origin server_; that is, the server providing the content Varnish will accelerate.

   Typically, this is your web server. See [Backend servers](https://varnish-cache.org/docs/trunk/users-guide/vcl-backends.html) in the _Varnish guide_.

1. Replace the value of `.port` with the web server's listen port (8080 in this example).

   Example: Apache is installed on host 192.0.2.55 and Apache is listening on port 8080:

   ```conf
   backend default {
       .host = "192.0.2.55";
       .port = "8080";
   }
   ```

   >[!INFO]
   >
   >If Varnish and Apache are running on the same host, Adobe recommends that you use an IP address or hostname and not `localhost`.

1. Save your changes to `default.vcl` and exit the text editor.

1. Restart Varnish:

   ```bash
   service varnish restart
   ```

If Varnish fails to start, try running it from the command line as follows:

```bash
varnishd -d -f /etc/varnish/default.vcl
```

This should display error messages.


>[!INFO]
>
>If Varnish does not start as a service, you must configure SELinux rules to allow it to run.

## Verify Varnish is working

The following sections discuss how you can verify that Varnish is working but _without_ configuring Commerce to use it. You should try this before you configure Commerce.

Perform the tasks discussed in the following sections in the order shown:

- [Start Varnish](#start-varnish)
- [`netstat`](#netstat)

### Start Varnish

Enter: `service varnish start`

If Varnish fails to start as a service, start it from the command line as follows:

1. Start the Varnish CLI:

   ```bash
   varnishd -d -f /etc/varnish/default.vcl
   ```

1. Start the Varnish child process:

   When prompted, enter `start`

   The following messages display to confirm a successful start:

   ```terminal
   child (29805) Started
   200 0

   Child (29805) said
   Child (29805) said Child starts
   ```

### netstat

Log in to the Varnish server and enter the following command:

```bash
netstat -tulpn
```

Look for the following output in particular:

```terminal
tcp        0      0 0.0.0.0:80                  0.0.0.0:*                   LISTEN      32614/varnishd
tcp        0      0 127.0.0.1:58484             0.0.0.0:*                   LISTEN      32604/varnishd
tcp        0      0 :::8080                     :::*                        LISTEN      26822/httpd
tcp        0      0 ::1:48509                   :::*                        LISTEN      32604/varnishd
```

The preceding shows Varnish running on port 80 and Apache running on port 8080.

If you do not see output for `varnishd`, make sure that Varnish is running.

See [`netstat` options](https://tldp.org/LDP/nag2/x-087-2-iface.netstat.html).

## Install the Commerce software

Install the Commerce software if you have not already done so. When prompted for a Base URL, use the Varnish host and port 80 (for Varnish) because Varnish receives all incoming HTTP requests.

Possible error installing Commerce:

```terminal
Error 503 Service Unavailable
Service Unavailable
XID: 303394517
Varnish cache server
```

If you experience this error, edit `default.vcl` and add a timeout to the `backend` stanza as follows:

```conf
backend default {
   .host = "127.0.0.1";
   .port = "8080";
   .first_byte_timeout = 600s;
}
```

## Verify HTTP response headers

Now you can verify that Varnish is serving pages by looking at [HTML](https://glossary.magento.com/html) response headers returned from any page.

Before you can look at headers, you must set Commerce for developer mode. There are several ways to do it, the simplest of which is to modify `.htaccess` in the Commerce application root. You can also use the [`magento deploy:mode:set`](../cli/set-mode.md) command.

### Set Commerce for developer mode

To set Commerce for developer mode, use the [`magento deploy:mode:set`](../cli/set-mode.md#change-to-developer-mode) command.

### Look at the Varnish log

Make sure that Varnish is running then enter the following command on the Varnish server:

```bash
varnishlog
```

In a web browser, go to any Commerce page.

A long list of response headers display in your command prompt window. Look for headers like the following:

```terminal
-   BereqHeader    X-Varnish: 3
-   VCL_call       BACKEND_FETCH
-   VCL_return     fetch
-   BackendOpen    17 default(10.249.151.10,,8080) 10.249.151.10 60914
-   Backend        17 default(10.249.151.10,,8080)
-   Timestamp      Bereq: 1440449534.261791 0.000618 0.000618
-   ReqHeader      Host: 10.249.151.10
-   ReqHeader      Connection: keep-alive
-   ReqHeader      Content-Length: 86
-   ReqHeader      Cache-Control: max-age=0
-   ReqHeader      Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
-   ReqHeader      Origin: http://10.249.151.10
```

If headers like these do _not_ display, stop Varnish, check your `default.vcl`, and try again.

### Look at HTML response headers

There are several ways to look at response headers, including using a browser [plug-in](https://glossary.magento.com/plug-in) or a browser inspector.

The following example uses `curl`. You can enter this command from any machine that can access the Commerce server using HTTP.

```bash
curl -I -v --location-trusted '<your Commerce base URL>'
```

For example,

```bash
curl -I -v --location-trusted 'http://192.0.2.55/magento2'
```

Look for headers like the following:

```terminal
Content-Type: text/html; charset=iso-8859-1
X-Varnish: 15
Age: 0
Via: 1.1 varnish-v6
X-Magento-Cache-Debug: HIT
```
