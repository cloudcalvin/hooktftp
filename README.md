[![Build Status](https://travis-ci.org/epeli/hooktftp.png?branch=master)](https://travis-ci.org/epeli/hooktftp)

# hooktftp

Hooktftp is a dynamic read-only TFTP server. It's dynamic in a sense it is
executes hooks matched on read requests (RRQ) instead of reading files from
the file system. Hooks are matched with regular expressions and on match
hooktftp will execute a script, issue a HTTP GET request or just reads the file
from the filesystem.

This is server inspired by [puavo-tftp]. It's written in Go in the hope of
being faster and more stable.

It's intented to be used with [PXELINUX][] for dynamic mac address based boot configurations.
This can be very useful in [LTSP][] enviroments.


## Usage

    hooktftp [config]

Config will be read from `/etc/hooktftp.yml` by default.

## Config

Config file is in yaml format and it can contain following keys:

  - `port`: Port to listen (required)
  - `user`: User to drop privileges to
  - `hooks`: Array of hooks. One or more is required

### Hooks

Hooks consists of following keys:

  - `type`: Type of the hook
    - `file`: Get response from the file system
    - `http`: Get response from a HTTP server
    - `shell`: Get response from external application
  - `regexp`: Regexp matcher
    - Hook is executed when this regexp matches the requested path
  - `template`: A template where the regexp is expanded

Regexp can be expanded to the template using the `$0`, `$1`, `$1` etc.
variables. `$0` is the full matched regexp and rest are the matched regexp
groups.

### Example

To share files from `/var/lib/tftpboot` add following hook:

```yaml
type: file
regexp: ^.*$
template: /var/lib/tftpboot/$0
```

Share custom boot configurations for PXELINUX from a custom http server:

```yaml
type: http
regexp: pxelinux.cfg\/01-(([0-9A-Fa-f]{2}[:-]){5}[0-9A-Fa-f]{2})
template: http://localhost:8080/boot/$1
```

The order of the hooks matter. The first one matched is used.

To put it all together:

```yaml
port: 69
user: hooktftp
hooks:
  - type: http
    regexp: pxelinux.cfg\/01-(([0-9A-Fa-f]{2}[:-]){5}[0-9A-Fa-f]{2})
    template: http://localhost:8080/boot/$1

  - type: file
    regexp: ^.*$
    template: /var/lib/tftpboot/$0
```

## Silly benchmarks

On AMD Phenom II X4 965 and Samsung SSD

    time tools/speed.sh

server            | size | concurrency | count | blocksize | time
----------------- |------|-------------|-------|---------- |-----
puavo-tftp (ruby) | 11M  | 1           | 10    | 512       | 0m27.012s
hooktftp   (Go)   | 11M  | 1           | 10    | 512       | 0m16.126s
tftp-hpa   (C)    | 11M  | 1           | 10    | 512       | 0m14.409s


    time tools/speed-concurrent.sh

server            | size | concurrency | count | blocksize | time
----------------- |------|-------------|-------|---------- |-----
puavo-tftp (ruby) | 11M  | 10          | 10    | 512       | 0m59.869s
hooktftp   (Go)   | 11M  | 10          | 10    | 512       | 0m24.531s
tftp-hpa   (C)    | 11M  | 10          | 10    | 512       | 0m10.326s Broken test?

# Install

## apt-get

Add

    deb http://archive.opinsys.fi/hooktftp precise main

to `/etc/apt/sources.list` and install `hooktftp` package

    sudo apt-get update
    sudo apt-get install hooktftp

or just pick up .deb package from <http://archive.opinsys.fi/hooktftp/pool/precise/main/h/hooktftp/>.

After installing you should have `hooktftp` executable in your PATH.

There are currently only 64bit Ubuntu Precise packages. But the package is so
simple it will likely work just fine on latter Ubuntu 64bit versions too and
probably on 64bit debian also.

## Compiling from sources

[Install Go][] (1.1 or later). Assuming you've successfully set up GOPATH and have GOPATH/bin on your path, simply:
    
    go get github.com/epeli/hooktftp
    
Now you should have a standalone hooktftp binary on your path.

    hooktftp -h
    Usage: hooktftp [config]

To implement changes, run `go install` from `$GOPATH/src/github.com/epeli/hooktftp` and the binary on your path will reflect your most recent changes.

[puavo-tftp]: https://github.com/opinsys/puavo-tftp
[PXELINUX]: http://www.syslinux.org/wiki/index.php/PXELINUX
[LTSP]: http://www.ltsp.org/
[Install Go]: http://golang.org/doc/install
