# OpenVPN for Docker

Quick instructions:

## Build the Image

```bash
# Don't know how to specify a branch in "docker build github.com/..." style.
# So manually clone the repository and check the branch out.
git clone https://github.com/tatsuya6502/dockvpn.git
cd dockvpn/
git checkout tatsuya6502

docker build -t openvpn .
```

## Run It

```bash
CID=$(docker run -d --privileged -p 1194:1194/udp -p 443:443/tcp -t openvpn)
docker run -t -i -p 8080:8080 --volumes-from $CID openvpn serveconfig
```

Now download the file located at the indicated URL. You will get a
certificate warning, since the connection is done over SSL, but we are
using a self-signed certificate. After downloading the configuration,
stop the `serveconfig` container. You can restart it later if you need
to re-download the configuration, or to download it to multiple devices.

The file can be used immediately as an OpenVPN profile. It embeds all the
required configuration and credentials. It has been tested successfully on
Linux, Windows, and Android clients. If you can test it on OS X and iPhone,
let me know!

**Note:** there is a [bug in the Android Download Manager](
http://code.google.com/p/android/issues/detail?id=3492) which prevents
downloading files from untrusted SSL servers; and in that case, our
self-signed certificate means that our server is untrusted. If you
try to download with the default browser on your Android device,
it will show the download as "in progress" but it will remain stuck.
You can download it with Firefox; or you can transfer it with another
way: Dropbox, USB, micro-SD card...

If you reboot the server (or stop the container), if you `docker run`
again, you will create a new service (with a new configuration) and
you will have to re-download the configuration file. However, you can
use `docker start` to restart the service without touching the configuration.


## How does it work?

When the `jpetazzo/openvpn` image is started, it generates:

- Diffie-Hellman parameters,
- a private key,
- a self-certificate matching the private key,
- two OpenVPN server configurations (for UDP and TCP),
- an OpenVPN client profile.

Then, it starts two OpenVPN server processes (one on 1194/udp, another
on 443/tcp).

The configuration is located in `/etc/openvpn`, and the Dockerfile
declares that directory as a volume. It means that you can start another
container with the `-volumes-from` flag, and access the configuration.
Conveniently, `jpetazzo/openvpn` comes with a script called `serveconfig`,
which starts a pseudo HTTPS server on `8080/tcp`. The pseudo server
does not even check the HTTP request; it just sends the HTTP status line,
headers, and body right away.


## OpenVPN details

We use `tun` mode, because it works on the widest range of devices.
`tap` mode, for instance, does not work on Android, except if the device
is rooted.

The topology used is `net30`, because it works on the widest range of OS.
`p2p`, for instance, does not work on Windows.

The TCP server uses `192.168.255.0/25` and the UDP server uses
`192.168.255.128/25`.

The server profile specifies the followings:

- `push "redirect-gateway def1"`, meaning that after a client
  establishing the VPN connection, all traffic from the client will go
  through the VPN.
- `push "dhcp-option DNS 8.8.4.4"` and `push "dhcp-option DNS 8.8.8.8"`,
  to make a client to use public DNS resolvers from Google.


## Security discussion

For simplicity, the client and the server use the same private key and
certificate. This is certainly a terrible idea. If someone can get their
hands on the configuration on one of your clients, they will be able to
connect to your VPN, and you will have to generate new keys. Which is,
by the way, extremely easy, since each time you `docker run` the OpenVPN
image, a new key is created. If someone steals your configuration file
(and key), they will also be able to impersonate the VPN server (if they
can also somehow hijack your connection).

It would probably be a good idea to generate two sets of keys.

It would probably be even better to generate the server key when
running the container for the first time (as it is done now), but
generate a new client key each time the `serveconfig` command is
called. The command could even take the client CN as argument, and
another `revoke` command could be used to revoke previously issued
keys.
