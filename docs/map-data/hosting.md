# Map Hosting

Now that you've [generated](generating.md) map data, you need to host it to your users.

## S3

Pros:

- Serverless

Cons:

- Lots of small individual files

## Normal server

Expected cost per month:


- Static IP. As of April 1, 2020, [$.004/hour][static-ip-pricing] or around $3/month.

[static-ip-pricing]: https://cloud.google.com/compute/network-pricing#ipaddress

### Cloudflare

I use Cloudflare to save on bandwidth and improve download speeds. It's free! To
point a URL to your server, add `A` records where the name is the subdomain you
want to point, and the IP address is the static IP address of your server.

You could also set up multiple subdomains that point to the same server.
This allows for [HTTP multiplexing][so-http-multiplexing] with HTTP 1.1,
essentially letting map tile requests be parallelized. It appears with HTTP 2.0
this might be unnecessary, but it's simple and Cloudflare says that currently
10% of my requests are HTTP 1.1 requests.

Essentially, then in the `tiles` section of the `Tile JSON` you can list
multiple domains, and all of them will return data in parallel.

[so-http-multiplexing]: https://stackoverflow.com/a/36519379

### Server Firewall

Since I'm sending all traffic through Cloudflare, I don't want any other HTTP or
HTTPS access to the server. Allowing other access could 1) let attackers DDOS me
by pointing directly to the IP address and 2) I could end up with higher egress
bandwidth fees from Google since not everything is being cached by Cloudflare.

To set up the firewall, add the [current list of Cloudflare IP
addresses][cloudflare-ips] to a new firewall.

[cloudflare-ips]: https://support.cloudflare.com/hc/en-us/articles/360037983412-Configuring-an-Amazon-Web-Services-static-site-to-use-Cloudflare#77nNxWyQf69T1a78gPlCi9

I found that Google got mad if I added IPv6 addresses, so I added just the ones
below. Include `tcp:80` and `tcp:443` as protocols, and apply the firewall to
the desired instances.

```
103.21.244.0/22
103.22.200.0/22
103.31.4.0/22
104.16.0.0/12
108.162.192.0/18
131.0.72.0/22
141.101.64.0/18
162.158.0.0/15
172.64.0.0/13
173.245.48.0/20
188.114.96.0/20
190.93.240.0/20
197.234.240.0/22
198.41.128.0/17
```

Now the server should only be accessible through your Cloudflare proxy!

### Static IP

By default, the IP address you get when you start a Google Cloud server is ephemeral. It often stays the same for a while, but is allowed to change. In order for your DNS provider to make sure your URL always reaches your server, you need to make sure the IP address doesn't change. [Here's an article][google-cloud-static-ip] on how to promote an ephemeral IP to address to a static IP address on Google Cloud.

[google-cloud-static-ip]: https://cloud.google.com/compute/docs/ip-addresses/reserve-static-external-ip-address

### `mbtileserver`

Now that we've set up the server, we need a program to serve the `.mbtiles`
files. [`mbtileserver`][mbtileserver] is a nice lightweight server for that,
mapping HTTP requests to the desired tile within the SQLite file.

[mbtileserver]: https://github.com/consbio/mbtileserver

#### Install

First install Golang, then install the program

```
go get github.com/consbio/mbtileserver
```

Then the executable should reside at
```
~/go/bin/mbtileserver
```

#### Using

```bash
./go/bin/mbtileserver \
    `# Folder with mbtiles files` \
    -d mbtiles/ \
    `# Port to run on` \
    -p 8000 \
    `# Domain of website. Sets endpoint correctly in the Tile JSON` \
    --domain tiles.example.com \
    `# Should allow server to restart if needed` \
    --enable-reload \
    `# Verbose logging` \
    --verbose
```

Each mbtiles file is given its own server path. If an mbtiles file is named
`openmaptiles.mbtiles`, the Tile JSON endpoint would be
```
localhost:8000/services/openmaptiles
```
and the tiles would be at
```
localhost:8000/services/openmaptiles/{z}/{x}/{y}.pbf
```

I haven't figured out how to successfully change the domain name in the Tile
JSON yet, see [#88](https://github.com/consbio/mbtileserver/issues/88).

### Caddy

`mbtileserver` isn't meant to be directly exposed to the internet. For one, it's
nice to add `Cache-Control` headers, which `mbtileserver` doesn't currently
support. I use [Caddy][caddy] as a reverse proxy to call `mbtileserver`
internally on port 8000.

[caddy]: https://caddyserver.com/v1

#### Install

```
curl https://getcaddy.com | bash -s personal http.nobots,http.ratelimit
```

If you want to include the `nobots` and `ratelimit` extensions, use:
```
curl https://getcaddy.com | bash -s personal http.nobots,http.ratelimit
```

The `caddy` executable should now be on the default `PATH`.

#### Using

Add a text file named `Caddyfile` to the current directory (`~` is fine). I use something like:
```
# Define each of the following websites, each on port 80
example.com:80, a_example.com:80, b_example.com:80 {
	proxy / localhost:8000
	proxy /services/openmaptiles localhost:8000 {
		header_downstream Cache-Control "public, max-age=20000, stale-while-revalidate=1000"
	}
	log caddy_log.log {
		rotate_size 10 # Rotate a log when it reaches 10 MB
		rotate_age  14  # Keep rotated log files for 14 days
		rotate_keep 2  # Keep at most 2 rotated log files
		rotate_compress # Compress rotated log files in gzip format
	}
	errors caddy_errors.log {
		rotate_size 10 # Rotate a log when it reaches 10 MB
		rotate_age  14  # Keep rotated log files for 14 days
		rotate_keep 2  # Keep at most 2 rotated log files
		rotate_compress # Compress rotated log files in gzip format
	}
}
```

### Higher `ulimit`

Caddy complains that the default `ulimit` (number of open files at once) is too
small. To [fix this][so-ulimit], in `/etc/security/limits.conf` I added:
```
* soft nofile 8192
* hard nofile 64000
```

[so-ulimit]: https://serverfault.com/a/610135
