---
title: Advanced Varnish configuration
description: Configure advanced Varnish features, including health check, grace, and saint modes.
---

# Advanced Varnish configuration

Varnish provides several features that prevent customers from experiencing long delays and timeouts when the Commerce server is not functioning properly. These features can be configured in the `default.vcl` file. This topic describes the additions that Commerce provides in the VCL (Varnish Configuration Language) file you download from the Admin.

See the [Varnish Reference Manual](https://varnish-cache.org/docs/6.3/reference/index.html) for details about using the Varnish Configuration Language.

## Health check

The Varnish health check feature polls the Commerce server to determine whether it is responding in a timely manner. If it is responding normally, fresh content is regenerated after the Time to Live (TTL) period expires. If not, Varnish always serves stale content.

Commerce defines the following default health check:

```conf
.probe = {
    .url = "/pub/health_check.php";
    .timeout = 2s;
    .interval = 5s;
    .window = 10;
    .threshold = 5;
    }
```

Every 5 seconds, this health check calls the `pub/health_check.php` script. This script checks the availability of the server, each database, and Redis (if installed). The script must return a response within 2 seconds. If the script determines that any of these resources are down, it returns a 500 HTTP error code. If this error code is received in six out of ten attempts, the [backend](https://glossary.magento.com/backend) is considered unhealthy.

The `health_check.php` script is located in the `pub` directory. If your Commerce root directory is `pub`, then be sure to change the path in the `url` parameter from `/pub/health_check.php` to `health_check.php`.

For more information, see the [Varnish health checks](https://varnish-cache.org/docs/6.3/users-guide/vcl-backends.html?highlight=health%20check#health-checks) documentation.

## Grace mode

Grace mode enables Varnish to keep an object in [cache](https://glossary.magento.com/cache) beyond its TTL value. Varnish can then serve the expired (stale) content while it fetches a new version. This improves the flow of traffic and decreases load times. It is used in the following situations:

- When the Commerce backend is healthy, but a request is taking longer than normal
- When the Commerce backend is not healthy.

The `vcl_hit` subroutine defines how Varnish responds to a request for objects that have been cached.

### When the Commerce backend is healthy

When the health checks determine that the Commerce backend is healthy, Varnish checks whether time remains in the grace period. The default grace period is 300 seconds, but a merchant can set the value from the [Admin](https://glossary.magento.com/admin) as described in [Configure Commerce to use Varnish](configure-varnish-commerce.md). If the grace period has not expired, Varnish delivers the stale content, and asynchronously refreshes the object from the Commerce server. If the grace period has expired, Varnish serves the stale content and synchronously refreshes the object from the Commerce backend.

The maximum amount of time that Varnish serves a stale object is the sum of the grace period (300 seconds by default) and the TTL value (86400 seconds by default).

To change the default grace period from within the `default.vcl` file, edit the following line in the `vcl_hit` subroutine:

```conf
if (obj.ttl + 300s > 0s) {
```

### When the Commerce backend is not healthy

If the Commerce backend is not responsive, Varnish serves stale content from cache for three days (or as defined in `beresp.grace`) plus the remaining TTL (which by default is one day), unless the cached content is manually purged.

## Saint mode

Saint mode excludes unhealthy backends for a configurable amount of time. As a result, unhealthy backends cannot serve traffic when using Varnish as a load balancer. Saint mode can be used with grace mode to allow for complex handling of unhealthy backend servers. For example, if one backend server is unhealthy, retries can be routed to another server. If all other servers are down, then serve stale cached objects. The saint mode backend hosts and blackout periods are defined in the `default.vcl` file.

Saint mode can also be used when Commerce instances are individually taken offline to perform maintenance and upgrade tasks without affecting the availability of the Commerce site.

### Saint mode prerequisites

Designate one machine as the primary installation. On this machine, install the main instance of Commerce, mySQL database, and Varnish.

On all other machines, the Commerce instance must have access the primary machine's mySQL database. The secondary machines should also have access to the files of the primary Commerce instance.

Alternatively, [static files](https://glossary.magento.com/static-files) versioning can be turned off on all machines. This can be accessed from the Admin under **Stores** > Settings > **Configuration** > **Advanced** > **Developer** > **Static Files Settings** > **Sign Static Files** = **No**.

Finally, all Commerce instances must be in production mode. Before Varnish starts, clear the cache on each instance. In the Admin, go to **System** > Tools > **Cache Management** and click **Flush Magento Cache**. You can also run the following command to clear the cache:

```bash
bin/magento cache:flush
```

### Installation

Saint mode is not part of the main Varnish package. It is a separately versioned `vmod` that must be downloaded and installed. As a result, you should recompile Varnish from source, as described in the following articles:

- [Installing Varnish 6.4](https://varnish-cache.org/docs/6.4/installation/install.html)
- [Installing Varnish 6.0](https://varnish-cache.org/docs/6.0/installation/install.html) (LTS)

After you recompile, you can install the Saint mode [module](https://glossary.magento.com/module). In general, follow these steps:

1. Obtain the source code from [Varnish modules](https://github.com/varnish/varnish-modules). Clone the Git version (master version) since the 0.9.x versions contain a source code error.
1. Build the source code with autotools:

    ```bash
    sudo apt-get install libvarnishapi-dev || sudo yum install varnish-libs-devel
    ./bootstrap   # If running from git.
    ./configure
    make
    make check   # optional
    sudo make install
    ```

See [Varnish module collection](https://github.com/varnish/varnish-modules) for information about installing the Saint mode module.

### Sample VCL file

The following code example shows the code that must be added to your VCL file to enable saint mode. Place the `import` statements and `backend` definitions at the top of the file.

```cpp

import saintmode;
import directors;

backend default1 {
    .host = "192.168.0.1";
    .port = "8080";
    .first_byte_timeout = 600s;
    .probe = {
        .url = "/pub/health_check.php";
        .timeout = 2s;
        .interval = 5s;
        .window = 10;
        .threshold = 5;
    }
}

backend default2 {
    .host = "192.168.0.2";
    .port = "8080";
    .first_byte_timeout = 600s;
    .probe = {
        .url = "/pub/health_check.php";
        .timeout = 2s;
        .interval = 5s;
        .window = 10;
        .threshold = 5;
    }
}

sub vcl_init {
    # Instantiate sm1, sm2 for backends tile1, tile2
    # with 10 blacklisted objects as the threshold for marking the
    # whole backend sick.
    new sm1 = saintmode.saintmode(default1, 10);
    new sm2 = saintmode.saintmode(default2, 10);

    # Add both to a director. Use sm0, sm1 in place of tile1, tile2.
    # Other director types can be used in place of random.
    new magedirector = directors.random();
    magedirector.add_backend(sm1.backend(), 1);
    magedirector.add_backend(sm2.backend(), 1);
}

sub vcl_backend_fetch {
    # Get a backend from the director.
    # When returning a backend, the director will only return backends
    # saintmode says are healthy.
    set bereq.backend = magedirector.backend();
}

sub vcl_backend_response {
    if (beresp.status >= 500) {
        # This marks the backend as sick for this specific
        # object for the next 20s.
        saintmode.blacklist(20s);
        # Retry the request. This will result in a different backend
        # being used.
        return (retry);
    }
}
```
