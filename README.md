# Moby ECH + ClientHello fragmentation test

Unofficial Linux x86_64 test build of Moby `dockerd`, based on Moby commit `9aa9edf` plus a local ECH and TLS ClientHello/SNI-fragmentation prototype. It is not an official Docker/Moby release.

## What it tests

- Optional TLS ClientHello record fragmentation for registry connections, enabled with `MOBY_TLS_FRAGMENT_CLIENT_HELLO=1`.
- ECH discovery when the registry hostname itself advertises an ECH configuration in HTTPS/SVCB DNS records.
- A separate test daemon and socket for a `hello-world:latest` image pull.

The fragmentation feature is opt-in. It splits the first ClientHello into valid TLS handshake records and preserves normal certificate verification. It does not guarantee that the records become separate TCP packets, and it cannot guarantee bypassing a middlebox that reassembles TCP streams. Docker Hub's `registry-1.docker.io` and its blob hosts may not advertise ECH configuration, so this build does not force Cloudflare ECH onto those hosts.

## Download

Download `moby-ech-test-linux-amd64.tar.gz` from the repository's Releases page. The archive includes `dockerd`, the install/start/test/stop scripts, README, checksums, and Moby `LICENSE`/`NOTICE` files.

SHA-256 of the current archive:

```text
0ccd3194d2f0b9d818ee578bb4385e8a648e54ee18819b7defa7a300a49825ab
```

## VPS requirements

- Linux x86_64/amd64, root or sudo.
- Existing Docker CLI (`docker`) and `runc` available in root's `PATH`.
- DNS, outbound HTTPS, and a CA certificate bundle.
- No external `containerd` is needed; the test daemon uses Moby's embedded containerd.

## Install and run

Copy the archive to `/tmp/moby-ech-test-linux-amd64.tar.gz` on the VPS, then run:

```sh
mkdir -p /tmp/moby-ech-test
tar -xzf /tmp/moby-ech-test-linux-amd64.tar.gz -C /tmp/moby-ech-test --strip-components=1
cd /tmp/moby-ech-test
sha256sum -c SHA256SUMS
sudo ./install.sh
sudo env MOBY_TLS_FRAGMENT_CLIENT_HELLO=1 /opt/moby-ech-test/start.sh
sudo /opt/moby-ech-test/test.sh
sudo /opt/moby-ech-test/stop.sh
```

The test script uses `unix:///run/moby-ech-test/docker.sock` and pulls `hello-world:latest`. It does not contact the existing daemon's default socket and does not run the pulled container.

The test daemon has a separate data root, exec root, PID file, and embedded-containerd state. It uses the `vfs` storage driver, disables the default bridge and iptables handling, and is intended only for daemon startup and image-pull tests. Do not remove `--bridge=none` or `--iptables=false` for side-by-side testing.

## Runtime files

- Binary: `/opt/moby-ech-test/dockerd`
- Socket: `/run/moby-ech-test/docker.sock`
- Data: `/var/lib/moby-ech-test`
- Log: `/var/log/moby-ech-test.log`

The package does not replace or modify the system Docker daemon. Review the included `README.md` inside the archive for cleanup commands and further detail.

## Source and license

`moby-ech-sni-fragmentation.patch` contains the local source diff against Moby. The binary is built from that modified source tree for testing. Moby is distributed under Apache-2.0; corresponding `LICENSE` and `NOTICE` are included in the repository and archive.
