# Self-Hosted Shared Photo & Video Storage

A small, self-hosted shared file space running on an Oracle Cloud VM,
powered by **SFTPGo**, **Docker Compose**, and **Caddy**.

This setup is intended for a small group sharing one common folder. Both
people use the same SFTPGo account and see the same files.

> **Important:** This README documents the setup discussed in this
> project. Check the live Compose file and Caddyfile before making
> changes; configuration may have changed since this guide was written.

------------------------------------------------------------------------

## 1. What the setup looks like

``` text
                         INTERNET
                            |
                 https://retrorookie.duckdns.org
                            |
                       Ports 80/443
                            |
                    +---------------+
                    |     CADDY     |
                    | Reverse proxy |
                    +---------------+
                       /         \
              normal paths      /dav/*
                  |                |
           127.0.0.1:8080   127.0.0.1:10080
                  |                |
             SFTPGo Web       SFTPGo WebDAV
                  \                /
                   \              /
                    +------------+
                    | SFTPGo     |
                    |            |
                    +------------+
                          |
                  /srv/shared (container)
                          |
                    Docker bind mount
                          |
                   /srv/storage (VM)
```

There is one storage directory on the VM:

-   **Host path:** `/srv/storage`
-   **Path inside the SFTPGo container:** `/srv/shared`

The browser interface and WebDAV both access this same directory. A file
uploaded through one interface should appear through the other.

### Main URLs

  Purpose                 Address
  ----------------------- ----------------------------------------------
  Browser file manager    `https://retrorookie.duckdns.org/web/client`
  SFTPGo administration   `https://retrorookie.duckdns.org/web/admin`
  WebDAV endpoint         `https://retrorookie.duckdns.org/dav/`

The admin interface is sensitive. Use a strong administrator password
and do not share the admin account. The shared file account should be a
separate, non-admin SFTPGo user.

------------------------------------------------------------------------

## 2. Components and why they are used

### Oracle Cloud VM

The VM is the computer that stays online and stores the files. It runs
Ubuntu, Docker, SFTPGo, and Caddy.

### Docker and Docker Compose

Docker runs SFTPGo in a container. Compose describes the container
configuration in one YAML file, including ports, volumes, environment
variables, and restart policy.

A container is isolated from the host in several ways, but it can access
host files that are deliberately mounted into it.

### SFTPGo

SFTPGo provides:

-   A browser-based file manager (WebClient).
-   An administration interface (WebAdmin).
-   WebDAV access for mounting the storage as a remote folder.
-   User accounts and file permissions.

The WebClient and WebDAV are two ways of accessing the same files; they
are not separate storage locations.

### Caddy

Caddy is the public-facing web server and reverse proxy. It receives
HTTPS requests on ports 80/443, obtains/manages TLS certificates, and
forwards requests to the appropriate local SFTPGo service.

### DuckDNS

DuckDNS provides a hostname, `retrorookie.duckdns.org`, that points to
the VM's public IP address. This means you can use a domain instead of
remembering the IP.

------------------------------------------------------------------------

## 3. Important networking concepts

### `localhost` / `127.0.0.1`

`127.0.0.1` is the loopback address. It means "this same machine" from
the perspective of the process making the connection.

Your laptop's `127.0.0.1` is your laptop. The VM's `127.0.0.1` is the
VM. They are different network interfaces on different machines.

### `0.0.0.0`

When a server binds to `0.0.0.0`, it listens on all of its available
IPv4 interfaces. This is useful when other machines need to connect
directly to that server.

It does **not** automatically make a service reachable from the
internet. Firewalls, routing, cloud security rules, and port publishing
still matter.

In this setup, SFTPGo's host ports are bound to the VM's loopback
interface. Caddy runs on the same VM and can reach them locally, so
SFTPGo does not need to be directly exposed to the internet.

### Reverse proxy

A reverse proxy receives requests on behalf of a server and forwards
them to an internal service.

Here:

1.  A browser connects to `https://retrorookie.duckdns.org`.
2.  DNS points the hostname to the VM.
3.  Caddy receives the HTTPS request on port 443.
4.  Caddy checks the request path.
5.  Requests under `/dav` go to SFTPGo's WebDAV listener on
    `127.0.0.1:10080`.
6.  Other requests go to SFTPGo's HTTP listener on `127.0.0.1:8080`.
7.  Caddy relays the response to the browser.

This keeps SFTPGo's application ports private while Caddy handles public
HTTPS traffic.

### SSH tunnel / local port forwarding

Earlier, the SFTPGo admin page was tested before the public
reverse-proxy configuration was ready.

The command was:

``` bash
ssh -L 8080:127.0.0.1:8080 ubuntu@retrorookie.duckdns.org
```

The format is:

``` text
-L LOCAL_PORT:REMOTE_DESTINATION_HOST:REMOTE_DESTINATION_PORT
```

In this example:

-   The first `8080` opens port 8080 on your laptop.
-   `127.0.0.1:8080` is interpreted from the VM's point of view.
-   SSH securely forwards traffic from your laptop's local port through
    the SSH connection to the VM's port 8080.

Then the browser can visit:

``` text
http://127.0.0.1:8080/web/admin
```

The browser connects to the local SSH tunnel, and SSH carries the
traffic to SFTPGo on the VM. The service is not made public just for
this test.

To create a tunnel without opening a remote shell:

``` bash
ssh -N -L 8080:127.0.0.1:8080 ubuntu@retrorookie.duckdns.org
```

Keep that terminal open while using the tunnel. Press `Ctrl+C` to stop
it.

You can forward another service in the same way. For example, to access
a VM-side PostgreSQL service on port 5432 through local port 15432:

``` bash
ssh -N -L 15432:127.0.0.1:5432 ubuntu@retrorookie.duckdns.org
```

Your local database client would connect to `127.0.0.1:15432`.

------------------------------------------------------------------------

## 4. Current deployment configuration

### Directory layout

``` text
/opt/sftpgo/
├── compose.yaml
├── config/
└── data/

/srv/storage/
└── shared files and folders
```

The Docker bind mount maps `/srv/storage` on the VM to `/srv/shared`
inside the SFTPGo container.

### Docker Compose

The deployment was configured approximately as follows. Treat the actual
file on the VM as authoritative:

``` yaml
services:
  sftpgo:
    image: drakkan/sftpgo:latest
    container_name: sftpgo
    restart: unless-stopped

    ports:
      - "127.0.0.1:8080:8080"
      - "127.0.0.1:10080:10080"

    volumes:
      - /opt/sftpgo/data:/srv/sftpgo
      - /opt/sftpgo/config:/var/lib/sftpgo
      - /srv/storage:/srv/shared

    environment:
      SFTPGO_HTTPD__BINDINGS__0__PORT: "8080"
      SFTPGO_WEBDAVD__BINDINGS__0__PORT: "10080"
      SFTPGO_WEBDAVD__BINDINGS__0__PREFIX: "/dav"
      SFTPGO_SFTPD__BINDINGS__0__PORT: "2022"

    stop_grace_period: 30s
```

The WebDAV prefix setting makes the WebDAV endpoint use `/dav`. The SFTP
listener is configured inside SFTPGo, but port 2022 is not published to
the VM in this Compose example. WebDAV is the intended network-folder
method here.

The database and SFTPGo data are persisted in `/opt/sftpgo/data` through
the bind mount. The shared files are persisted separately in
`/srv/storage`.

### Caddyfile

The Caddy configuration is:

``` caddyfile
retrorookie.duckdns.org {
    encode zstd gzip

    request_body {
        max_size 10GB
    }

    @webdav path /dav*
    handle @webdav {
        reverse_proxy 127.0.0.1:10080
    }

    handle {
        reverse_proxy 127.0.0.1:8080
    }
}
```

Caddy preserves the request host when proxying by default. This matters
for WebDAV operations such as `COPY` and `MOVE`.

------------------------------------------------------------------------

## 5. User, ownership, and permissions

Linux checks the identity of the process accessing a file. It does not
care which human originally created the file; it checks the process's
user ID (UID), group ID (GID), and permissions.

The storage directory was set up with UID/GID `1000:1000`, matching the
expected default identity used by the SFTPGo container image in this
deployment.

Check the real values on the VM:

``` bash
ls -ldn /srv/storage
stat /srv/storage
```

The `-n` option displays numeric UID/GID values rather than resolving
names.

Useful commands:

``` bash
whoami                 # Current username
id                     # Current UID, GID and groups
groups                 # Groups for the current user
ls -l                  # File permissions and ownership
ls -ld DIRECTORY       # Directory's own permissions
stat FILE              # Detailed file metadata
namei -l /srv/storage  # Permissions on each component of a path
```

### Permission bits

For files and directories, Linux has three permission classes:

-   Owner
-   Group
-   Others

And three basic permissions:

  Bit   Files                  Directories
  ----- ---------------------- -----------------------------------
  `r`   Read file contents     List directory entries
  `w`   Modify file contents   Create, delete, or rename entries
  `x`   Execute file           Enter/traverse directory

Directory execute permission is especially important. A process may have
permission to read a file but still be unable to reach it if it cannot
traverse one of the parent directories.

### Numeric `chmod`

The numeric values are:

-   Read = 4
-   Write = 2
-   Execute = 1

Add them together for each class:

  Mode   Meaning
  ------ ---------
  `7`    `rwx`
  `6`    `rw-`
  `5`    `r-x`
  `4`    `r--`
  `0`    `---`

Examples:

``` bash
chmod 755 script.sh
chmod 644 config.txt
chmod 700 private-directory
```

### `chown`

`chown` changes ownership:

``` bash
sudo chown username:groupname FILE
```

Recursive ownership changes affect everything below the path:

``` bash
sudo chown -R username:groupname DIRECTORY
```

Be careful with recursive changes. Do not blindly run `chmod -R 777` or
`chown -R` on system directories. `777` gives everyone read, write, and
execute permissions and is usually an unsafe workaround rather than a
real fix.

### Troubleshooting permission denied

When an upload or file operation fails:

1.  Check the host directory's numeric owner and group:
    `ls -ldn /srv/storage`.
2.  Check the container's running identity.
3.  Check that the SFTPGo user's home directory points to `/srv/shared`.
4.  Check that the SFTPGo user's permissions allow the required
    operation.
5.  Check every parent directory's traversal permissions with
    `namei -l`.
6.  Inspect SFTPGo logs:

``` bash
sudo docker logs --tail 100 sftpgo
```

Do not change ownership or permissions until you know which identity is
being denied and which path is involved.

------------------------------------------------------------------------

## 6. Access through the browser

Use the browser file manager from any device with a modern browser:

``` text
https://retrorookie.duckdns.org/web/client
```

Sign in with the shared SFTPGo file account.

You can browse, upload, download, create directories, rename, and delete
files according to the permissions assigned to that account.

The admin interface is separate:

``` text
https://retrorookie.duckdns.org/web/admin
```

Only the administrator should use this interface to manage SFTPGo
settings and accounts.

------------------------------------------------------------------------

## 7. Access through WebDAV

WebDAV is a protocol that lets compatible clients interact with remote
files over HTTP or HTTPS. It can make remote storage appear like a
network folder in a file manager.

The WebDAV URL is:

``` text
https://retrorookie.duckdns.org/dav/
```

Use the shared SFTPGo file account credentials, not the SFTPGo
administrator credentials.

### Linux

There are several ways to connect.

#### Option A: GNOME Files / Nautilus

1.  Open **Files**.

2.  Choose **Other Locations**.

3.  Find **Connect to Server**.

4.  Enter:

    ``` text
    davs://retrorookie.duckdns.org/dav/
    ```

5.  Connect and enter the shared SFTPGo username and password.

`davs://` means WebDAV over HTTPS.

#### Option B: KDE Dolphin

1.  Open Dolphin.

2.  In the location bar, enter:

    ``` text
    webdavs://retrorookie.duckdns.org/dav/
    ```

3.  Press Enter and authenticate when prompted.

Depending on the desktop environment and installed KIO support, the
exact URL scheme supported by the file manager may vary.

#### Option C: Mount using davfs2

On Debian/Ubuntu-based distributions:

``` bash
sudo apt update
sudo apt install davfs2
mkdir -p ~/SharedStorage
```

Mount it:

``` bash
sudo mount -t davfs https://retrorookie.duckdns.org/dav/ ~/SharedStorage
```

Enter the SFTPGo shared account credentials when prompted.

Unmount it when finished:

``` bash
sudo umount ~/SharedStorage
```

For repeated use, configure credentials securely using the davfs2
documentation rather than putting a password directly into shell history
or a script.

### Windows 10 / 11

Windows has a built-in WebDAV redirector.

1.  Open **File Explorer**.

2.  Right-click **This PC**.

3.  Choose **Map network drive**.

4.  Select a drive letter.

5.  In the folder field, enter:

    ``` text
    https://retrorookie.duckdns.org/dav/
    ```

6.  Enable **Connect using different credentials** if needed.

7.  Finish and enter the shared SFTPGo username and password.

If Windows reports that the network location cannot be reached, check
that the **WebClient** service is running. Windows WebDAV support can
also be affected by system policy, authentication settings, and service
configuration.

Do not weaken Windows authentication or enable insecure basic
authentication over plain HTTP. This setup uses HTTPS.

### Android

Android does not provide one universal built-in WebDAV file manager, so
use a file manager that supports WebDAV.

For example, in a WebDAV-capable file manager:

1.  Add a new remote storage / network location.

2.  Choose **WebDAV**.

3.  Enter:

    -   Server/URL: `https://retrorookie.duckdns.org/dav/`
    -   Username: your shared SFTPGo account
    -   Password: that account's password

4.  Save and connect.

A file manager such as **MiXplorer** or **Solid Explorer** can be used
if its current version supports WebDAV. Availability and feature support
may depend on the app version and licensing.

------------------------------------------------------------------------

## 8. Uploads, large files, and practical notes

-   The Caddy configuration currently sets a maximum request body size
    of 10 GB. This is a configured limit, not a guarantee that every 10
    GB upload will succeed.

-   Large uploads depend on the network connection, available disk
    space, client timeout behaviour, and SFTPGo settings.

-   Check available disk space on the VM:

    ``` bash
    df -h
    ```

-   Check memory:

    ``` bash
    free -h
    ```

-   Check container status:

    ``` bash
    sudo docker ps
    ```

-   Check SFTPGo logs:

    ``` bash
    sudo docker logs --tail 100 sftpgo
    ```

-   Check Caddy status:

    ``` bash
    sudo systemctl status caddy
    ```

-   Validate Caddy configuration before reloading:

    ``` bash
    sudo caddy validate --config /etc/caddy/Caddyfile
    sudo systemctl reload caddy
    ```

-   Validate Compose before applying changes:

    ``` bash
    cd /opt/sftpgo
    sudo docker compose config -q
    ```

------------------------------------------------------------------------

## 9. Useful maintenance commands

### Restart SFTPGo

``` bash
cd /opt/sftpgo
sudo docker compose restart
```

### Recreate SFTPGo after changing Compose configuration

``` bash
cd /opt/sftpgo
sudo docker compose config -q
sudo docker compose up -d --force-recreate
```

### Pull the latest image

``` bash
cd /opt/sftpgo
sudo docker compose pull
sudo docker compose up -d
```

Review release notes and consider backing up before updating production
services.

### View logs

``` bash
sudo docker logs -f sftpgo
```

Press `Ctrl+C` to stop following logs.

### Back up important data

At minimum, back up:

-   `/srv/storage` --- shared photos and videos.
-   `/opt/sftpgo/data` --- SFTPGo database and application data.
-   `/opt/sftpgo/compose.yaml` --- deployment configuration.
-   `/etc/caddy/Caddyfile` --- reverse-proxy configuration.

A backup stored only on the same VM is not sufficient protection against
VM or disk loss. Keep an additional copy somewhere independent.

------------------------------------------------------------------------

## 10. Security checklist

-   Use strong, unique passwords for the SFTPGo administrator and shared
    file account.
-   Do not share the administrator login.
-   Keep SFTPGo's application ports bound to `127.0.0.1` on the VM when
    using Caddy as the reverse proxy.
-   Keep only necessary public ports open. This setup needs SSH (22) for
    administration and HTTP/HTTPS (80/443) for Caddy, subject to your
    cloud and host firewall configuration.
-   Do not expose the Docker API publicly.
-   Keep Ubuntu, Docker, SFTPGo, and Caddy updated.
-   Back up the shared files and SFTPGo database.
-   Do not use `chmod 777` as a general permissions fix.
-   Treat the shared account as shared access: anyone who knows its
    credentials can perform whatever operations its permissions allow.

------------------------------------------------------------------------

## 11. Quick reference

  ----------------------------------------------------------------------------------------------------
  Task                                Command / URL
  ----------------------------------- ----------------------------------------------------------------
  Browser file manager                `https://retrorookie.duckdns.org/web/client`

  Admin interface                     `https://retrorookie.duckdns.org/web/admin`

  WebDAV                              `https://retrorookie.duckdns.org/dav/`

  Storage on VM                       `/srv/storage`

  Storage in container                `/srv/shared`

  SFTPGo Compose directory            `/opt/sftpgo`

  SFTPGo logs                         `sudo docker logs --tail 100 sftpgo`

  Check disk                          `df -h`

  Check memory                        `free -h`

  Check ownership                     `ls -ldn /srv/storage`

  Inspect path permissions            `namei -l /srv/storage`

  Validate Caddy                      `sudo caddy validate --config /etc/caddy/Caddyfile`

  Reload Caddy                        `sudo systemctl reload caddy`

  SSH local tunnel example            `ssh -N -L 8080:127.0.0.1:8080 ubuntu@retrorookie.duckdns.org`
  ----------------------------------------------------------------------------------------------------

------------------------------------------------------------------------

## 12. Mental model to remember

-   **Docker** runs SFTPGo in an isolated container.
-   **Bind mounts** make the VM's `/srv/storage` available inside that
    container as `/srv/shared`.
-   **Linux permissions** decide whether the SFTPGo process can read or
    modify those files.
-   **SFTPGo WebClient** provides the browser-based file manager.
-   **SFTPGo WebDAV** provides protocol-based remote-folder access.
-   **Caddy** is the public HTTPS reverse proxy that routes requests to
    the right SFTPGo listener.
-   **DuckDNS** maps the hostname to the VM's public IP.
-   **SSH local port forwarding** gives your laptop temporary, secure
    access to a service on the VM without publicly exposing that
    service.

That's the whole architecture: one shared directory, two ways to access
it, and a reverse proxy in front.
