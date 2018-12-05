# Outline ss-server

This repository has the Shadowsocks service soon to be used by Outline servers. It uses components from [go-shadowsocks2](https://github.com/shadowsocks/go-shadowsocks2), and adds a number of improvements to meet the needs of the Outline users.

The Outline Shadowsocks service allows for:
- Multiple users on a single port.
  - Does so by trying all the different credentials until one succeeds.
- Multiple ports
- Whitebox monitoring of the service using [prometheus.io](https://prometheus.io)
  - Includes traffic measurements and other health indicators.
- Live updates via config change + SIGHUP

![Graphana Dashboard](https://user-images.githubusercontent.com/113565/44177062-419d7700-a0ba-11e8-9621-db519692ff6c.png "Graphana Dashboard")

## Try it!

Fetch dependencies and build:
```
go get github.com/Jigsaw-Code/outline-ss-server github.com/shadowsocks/go-shadowsocks2 github.com/prometheus/prometheus/cmd/...
```

On Terminal 1, start the SS server:
```
$(go env GOPATH)/bin/outline-ss-server -config config_example.yml -metrics localhost:8080
```

On Terminal 2, start prometheus scraper for metrics collection:
```
$(go env GOPATH)/bin/prometheus --config.file=prometheus_example.yml
```

On Terminal 3, start the SS client:
```
$(go env GOPATH)/bin/go-shadowsocks2 -c ss://chacha20-ietf-poly1305:Secret0@:9000 -verbose  -socks :1080
```

On Terminal 4, fetch a page using the SS client:
```
curl --proxy socks5h://localhost:1080 example.com
```

Stop and restart the client on Terminal 3 with "Secret1" as the password and try to fetch the page again on Terminal 4.

Open http://localhost:8080/metrics and see the exported Prometheus variables.

Open http://localhost:9090/ and see the Prometheus server dashboard.


## Performance Testing

Start the iperf3 server (runs on port 5201 by default):
```
iperf3 -s
```

Start the SS server (listening on port 9000):
```
go build github.com/Jigsaw-Code/outline-ss-server && \
./outline-ss-server -config config_example.yml
```

Start the SS tunnel to redirect port 8000 -> localhost:5201 via the proxy on 9000:
```
$(go env GOPATH)/bin/go-shadowsocks2 -c ss://chacha20-ietf-poly1305:Secret0@:9000 -tcptun ":8000=localhost:5201" -udptun ":8000=localhost:5201" -verbose
```

Test TCP upload (client -> server):
```
iperf3 -c localhost -p 8000
```

Test TCP download (server -> client):
```
iperf3 -c localhost -p 8000 --reverse
```

Test UDP upload:
```
iperf3 -c localhost -p 8000 --udp -b 0
```

Test UDP download:
```
iperf3 -c localhost -p 8000 --udp -b 0 --reverse
```

### Compare to go-shadowsocks2

Run the commands above, but start the SS server with
```
$(go env GOPATH)/bin/go-shadowsocks2 -s ss://chacha20-ietf-poly1305:Secret0@:9000 -verbose
```


### Compare to shadowsocks-libev 

Start the SS server (listening on port 10001):
```
ss-server -s localhost -p 10001 -m chacha20-ietf-poly1305 -k Secret1 -u -v
```

Start the SS tunnel to redirect port 10002 -> localhost:5201 via the proxy on 10001:
```
ss-tunnel -s localhost -p 10001 -m chacha20-ietf-poly1305 -k Secret1 -l 10002 -L localhost:5201 -u -v
```

Run the iperf3 client tests listed above on port 10002.

You can mix and match the libev and go servers and clients.


## Development

For development you may want to use `git clone` over SSH instead of `go get`:

```
git clone git@github.com:Jigsaw-Code/outline-ss-server.git $(go env GOPATH)/src/github.com/Jigsaw-Code/outline-ss-server
```

## Release

We use [GoReleaser](https://goreleaser.com/) to build and upload binaries to our [GitHub releases](https://github.com/Jigsaw-Code/outline-ss-server/releases).

Summary:
- [Install GoReleaser](https://goreleaser.com/install/).
- Export an environment variable named `GITHUB_TOKEN` with a repo-scoped GitHub token ([create one here](https://github.com/settings/tokens/new)):
  ```bash
  export GITHUB_TOKEN=yournewtoken
  ```
- `cd` to your clone, most likely:
  ```bash
  cd ~/go/src/github.com/Jigsaw-Code/outline-ss-server
  ```
- Create a new tag and push it to GitHub e.g.:
  ```bash
  git tag v1.0.0
  git push origin v1.0.0
  ```
- Build and upload:
  ```bash
  goreleaser
  ```

To test locally without tagging/uploading , use `--skip-publish`:
```bash
goreleaser --skip-publish
```

Full instructions in [GoReleaser's Quick Start](https://goreleaser.com/quick-start) (jump to the section starting "You’ll need to export a GITHUB_TOKEN environment variable").
