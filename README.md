# scrappy-ddns
Runs [Scrappy DDNS](https://github.com/rhasselbaum/scrappy-ddns) in a Docker container with SSL/TLS support out of the box. Scrappy DDNS is a Dynamic DNS-like service that sends push notifications to your mobile devices whenever your public IP address changes. From the [project description](https://github.com/rhasselbaum/scrappy-ddns):

> So what good is a DDNS service that doesn't actually update DNS records? Scrappy might be right for you if:
> * Your DNS hosting service doesn't support true Dynamic DNS and your IP address rarely changes.
> * You prefer to manage DNS records manually.
> * You just want to know whenever your IP address changes.

This dockerized version of Scrappy DDNS runs the web service behind an nginx reverse proxy and enforces SSL/TLS encryption on all connections. This is the recommended way to run Scrappy DDNS on the public Internet.

# Getting started

Read the [Scrappy DDNS README](https://github.com/rhasselbaum/scrappy-ddns) to get up to speed on the basic concepts. You need to create a `token.list` file as described there and obtain an [application key](https://pushover.net/apps/clone/Scrappy_DDNS) on [Pushover](https://pushover.net/). You will also need an X.509 certificate signed by a Certificate Authority (CA) that your dynamic DNS clients trust. A self-signed certificate works fine but your clients must still validate it to prevent eavesdroppers from intercepting your tokens.

With these items in hand, create a configuration directory on the host to hold your `token.list` file, certificate, and private key arranged like this:

```
<config_dir>
  +--cert.pem
  +--cert.key
  +--token.list
```

The directory can be placed anywhere (e.g. `/etc/scrappyddns`), but the filenames within it must match the ones above. `cert.pem` and `cert.key` are the certificate file and matching private key in PEM-format (base64). These will be used for incoming SSL/TLS connections. The `token.list` file maps alphanumeric tokens in HTTPS requests to servers/networks you want to monitor.

Since the private key and token list files contain sensitive data, you may want to restrict read access to them on the host. The `token.list` file must be readable by UID/GID 33 (the `www-data` or `http` user in many distros). In most cases, the following commands will lock down the files appropriately:

```
sudo chmod 600 cert.key
sudo chmod 600 token.list
sudo chown 33:33 token.list
```

Now you can start a new Scrappy DDNS container like this:

```
docker run -d -p 443:443 -v <config_dir>:/etc/scrappyddns:ro \
 -e SCRAPPY_PUSH_USER_KEY=<user_key> \
 -e SCRAPPY_PUSH_APP_KEY=<app_key> \
 --name scrappy-ddns rhasselbaum/scrappy-ddns
```

Where:
* `<config_dir>` is the full path to the configuration directory on the host.
* `<user_key>` is your Pushover user key.
* `<app_key>` is the Pushover [application key](https://pushover.net/apps/clone/Scrappy_DDNS) for your copy of Scrappy DDNS.

This starts the service listening for HTTPS connections over port 443 on the host. You can change the exposed port in typical Docker fashion. Whatever port you choose should map to 443 in the container.

# Advanced options
The basic configuration shown above will probably suffice for most people. But some more advanced options are also available.

## Configuration variables
Any variable in the main [scrappyddns.conf](https://github.com/rhasselbaum/scrappy-ddns/blob/master/scrappyddns.conf) configuration file can be overridden with an environment variable by prepending `SCRAPPY_` to its name. For example, to turn up the log level to `DEBUG`, modify the `docker run` command like this:

```
docker run -d -p 443:443 -v <config_dir>:/etc/scrappyddns:ro \
 -e SCRAPPY_PUSH_USER_KEY=<user_key> \
 -e SCRAPPY_PUSH_APP_KEY=<app_key> \
 -e SCRAPPY_LOG_LEVEL=DEBUG \
 --name scrappy-ddns rhasselbaum/scrappy-ddns
```

If you don't like passing environment variables to `docker run`, you can instead place a [copy](https://raw.githubusercontent.com/rhasselbaum/scrappy-ddns/master/scrappyddns.conf) of `scrappyddns.conf` into your `<config_dir>` and customize it. You only need to include variables that you want to override from the image-supplied defaults. With an external configuration file, the `docker run` command can be made much simpler:

```
docker run -d -p 443:443 -v <config_dir>:/etc/scrappyddns:ro \
 --name scrappy-ddns rhasselbaum/scrappy-ddns
```
If the same variable appears in `scrappyddns.conf` and an environment variable, the latter takes precedence.

## IP address cache
The persistent cache that stores the mappings of tokens to IP addresses resides in a Docker volume and can be shared among containers or mounted to a directory on the host like so:

```
docker run -d -p 443:443 -v <cache_dir>:/var/cache/scrappyddns [...] \
 --name scrappy-ddns rhasselbaum/scrappy-ddns
```
Where `<cache_dir>` is the full path to the directory on the host. This directory must be readable and writable by UID/GID 33 (the `www-data` or `http` user in many distros).

# Troubleshooting

Log output is available through the `docker logs` facility. Misconfigurations or Pushover service interruptions will yield warnings or errors in the log. Currently, there's no retry mechanism for failed calls to the Pushover service.

You can get more detail about every request Scrappy DDNS receives by setting the `LOG_LEVEL` variable to `DEBUG`. The nginx error log is also included, which cam alert you to problems with the SSL/TLS certificate.