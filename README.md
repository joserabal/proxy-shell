# proxy-shell

An isolated shell where **all network traffic is transparently routed through a SOCKS5 proxy** — including DNS, which is automatically resolved through the remote proxy without any manual configuration.

Typical use case: you have a SOCKS5 proxy available (e.g. a corporate proxy, a Tor instance, an SSH tunnel, or any other) and you want every tool you run — `curl`, `nmap`, `git`, custom scripts — to go through it without touching each tool's settings.

## Table of contents

- [How it works](#how-it-works)
- [Advantages over other approaches](#advantages-over-other-approaches)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
- [Notes](#notes)

## How it works

`proxy-shell` uses Linux **network namespaces** and a **TUN interface** powered by [sing-box](https://github.com/SagerNet/sing-box) to create a fully isolated network environment. Inside that environment:

- All network traffic is transparently forwarded through your SOCKS5 proxy.
- DNS queries are intercepted and resolved by the remote proxy using a **fake IP** strategy — the same technique used by tools like Burp Suite's "Use SOCKS proxy for DNS lookups". No DNS leaks, no need to find or configure a remote DNS server.
- The isolation is enforced at the kernel level. There is no per-process configuration, no `LD_PRELOAD` hooking, no `ptrace`. It works with statically linked binaries, Go programs, Rust tools — anything.

When you exit the shell, everything is torn down automatically: the namespace, the TUN interface, and the sing-box process.

## Advantages over other approaches

Most proxy tools — such as **proxychains**, **proxychains-ng**, or **graftcp** — work by intercepting network calls at the process level, either via `LD_PRELOAD` hooks or `ptrace`. This approach has fundamental limitations:

- Statically linked binaries (common in modern Go and Rust tools) bypass `LD_PRELOAD`-based hooks entirely.
- Tools using `ptrace` silently lose control of any subprocess that gains elevated privileges (via `sudo` or binaries with `setcap`). The Linux kernel drops the trace when a setuid binary is exec'd — those processes connect directly through the real IP with no warning.
- Programs that implement their own DNS resolution or networking stack are not captured.
- You must configure each tool individually, or remember to prefix every command.
- DNS leaks are common — the tool is proxified but hostname resolution still happens locally.

A different approach is to set environment variables like `http_proxy` / `https_proxy` / `ALL_PROXY`. This requires no external tools, but only works with programs that explicitly check for them — support is inconsistent and many tools ignore them entirely. DNS leaks are also a problem here, since hostname resolution still happens locally.

`proxy-shell` solves all of these at once. Since the isolation is done at the **network namespace level**, every process running inside — regardless of language, runtime, or how it was compiled — has no choice but to go through the proxy. There is nothing to configure per-tool and nowhere to leak.

## Requirements

- Linux (kernel 4.0+)
- `iproute2`
- [`sing-box`](https://github.com/SagerNet/sing-box)
- `curl`

Missing dependencies are detected automatically at startup, with an option to install them interactively.

## Installation

Make `proxy-shell` available system-wide:

```bash
chmod +x proxy-shell
sudo ./proxy-shell --global-install
```

This copies the script to `/usr/local/bin` and checks for missing dependencies.

## Usage

```
sudo proxy-shell <host:port> [command]

  host:port   SOCKS5 proxy address (e.g. 127.0.0.1:8888)
  command     Optional command to run inside the proxified namespace.
              If omitted, opens an interactive shell — ALL traffic from
              every command you run will be transparently proxified.

  --global-install   Copy this script to /usr/local/bin and check dependencies.
```

### Open a proxified interactive shell

```bash
sudo proxy-shell 127.0.0.1:8888
```

Your prompt will change to indicate you are inside a proxified environment:

```
 proxy-shell ❯ ~ $
```

Every command you run from this shell will have its traffic routed through the proxy. Exit the shell normally (`exit` or Ctrl+D) to tear everything down.

### Run a single command through the proxy

```bash
sudo proxy-shell 127.0.0.1:8888 curl https://ifconfig.me
```

Runs the command inside the proxified namespace and exits immediately after it finishes.

### With an SSH tunnel

A common workflow for penetration testing or accessing internal networks:

```bash
# 1. Open the SSH tunnel in the background
ssh -D 8888 -N user@jumphost &

# 2. Open a proxified shell
sudo proxy-shell 127.0.0.1:8888

# Everything you run inside resolves and connects through the tunnel
 proxy-shell ❯ ~ $ curl https://internal.corp.example
 proxy-shell ❯ ~ $ nmap -sT internal-host
 proxy-shell ❯ ~ $ git clone https://internal-git.corp.example/repo.git
```

### Multiple simultaneous sessions

You can open as many proxified shells as you need — each instance creates its own isolated namespace and TUN device, so they never interfere with each other.

```bash
# Terminal 1
sudo proxy-shell 127.0.0.1:8888

# Terminal 2 — completely independent
sudo proxy-shell 127.0.0.1:9999
```

## Notes

- **UDP** support depends on your proxy. The tool captures all traffic including UDP, but whether it is forwarded depends on the SOCKS5 proxy implementation. `ssh -D` only supports TCP (`CONNECT`), so UDP will be dropped when using an SSH tunnel. Other SOCKS5 proxies that implement `UDP ASSOCIATE` (RFC 1928) will forward UDP traffic without any changes to this tool.
- **`sudo` inside the shell** works as expected — commands run with `sudo` are still fully proxified, since child processes inherit the network namespace regardless of user credentials.
