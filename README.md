* Patch is after: https://github.com/tianon/gosu/commit/e845c46b6bedb4457cda8c88814c30a53cc6c9e7
* Latest binaries at https://hub.docker.com/repository/docker/prapl4group/better-bases/tags?name=gosu&page=1
* How to use to fix other docker images:
  * https://internetworking.dev/mitigating-gosu-security-concerns/ (https://archive.is/wip/98TVx)

For example, I used this to build better mongo:
```
FROM prapl4group/better-bases:gosu-1.17-go-1.24-20250707 AS gosu
FROM mongo:4.4.29-rfhardened
COPY --from=gosu /go/bin/gosu-amd64 /usr/local/bin/gosu
RUN chmod +x /usr/local/bin/gosu
```

```diff
From 7b119c01586a470ebc4d7bca99b77b22d72581a6 Mon Sep 17 00:00:00 2001
From: Yoni Water Man 
Date: Mon, 7 Jul 2025 16:40:38 +0000
Subject: [PATCH 1/3] start new

---
 README.md             | 90 ++-----------------------------------------
 go.sum => _old_go.sum |  0
 2 files changed, 4 insertions(+), 86 deletions(-)
 rename go.sum => _old_go.sum (100%)

diff --git a/README.md b/README.md
index 079c4c7..8dfb660 100644
--- a/README.md
+++ b/README.md
@@ -1,89 +1,7 @@
-# gosu
-
-This is a simple tool grown out of the simple fact that `su` and `sudo` have very strange and often annoying TTY and signal-forwarding behavior.  They're also somewhat complex to setup and use (especially in the case of `sudo`), which allows for a great deal of expressivity, but falls flat if all you need is "run this specific application as this specific user and get out of the pipeline".
-
-The core of how `gosu` works is stolen directly from how Docker/libcontainer itself starts an application inside a container (and in fact, is using the `/etc/passwd` processing code directly from libcontainer's codebase).
-
-console
-$ gosu
-Usage: ./gosu user-spec command [args]
-   eg: ./gosu tianon bash
-       ./gosu nobody:root bash -c 'whoami && id'
-       ./gosu 1000:1 id
-
-./gosu version: 1.1 (go1.3.1 on linux/amd64; gc)
-
-
-Once the user/group is processed, we switch to that user, then we `exec` the specified process and `gosu` itself is no longer resident or involved in the process lifecycle at all.  This avoids all the issues of signal passing and TTY, and punts them to the process invoking `gosu` and the process being invoked by `gosu`, where they belong.
-
-## Warning
-
-The core use case for `gosu` is to step _down_ from `root` to a non-privileged user during container startup (specifically in the `ENTRYPOINT`, usually).
-
-Uses of `gosu` beyond that could very well suffer from vulnerabilities such as CVE-2016-2779 (from which the Docker use case naturally shields us); see [`tianon/gosu#37`](https://github.com/tianon/gosu/issues/37) for some discussion around this point.
-
-## Installation
-
-High-level steps:
-
-1. download `gosu-$(dpkg --print-architecture | awk -F- '{ print $NF }')` as `gosu`
-2. download `gosu-$(dpkg --print-architecture | awk -F- '{ print $NF }').asc` as `gosu.asc`
-3. fetch my public key (to verify your download): `gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4`
-4. `gpg --batch --verify gosu.asc gosu`
-5. `chmod +x gosu`
-
-For explicit `Dockerfile` instructions, see [`INSTALL.md`](INSTALL.md).
-
-## Why?
-
-console
-$ docker run -it --rm ubuntu:trusty su -c 'exec ps aux'
-USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
-root         1  0.0  0.0  46636  2688 ?        Ss+  02:22   0:00 su -c exec ps a
-root         6  0.0  0.0  15576  2220 ?        Rs   02:22   0:00 ps aux
-$ docker run -it --rm ubuntu:trusty sudo ps aux
-USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
-root         1  3.0  0.0  46020  3144 ?        Ss+  02:22   0:00 sudo ps aux
-root         7  0.0  0.0  15576  2172 ?        R+   02:22   0:00 ps aux
-$ docker run -it --rm -v $PWD/gosu-amd64:/usr/local/bin/gosu:ro ubuntu:trusty gosu root ps aux
-USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
-root         1  0.0  0.0   7140   768 ?        Rs+  02:22   0:00 ps aux
 
+https://internetworking.dev/mitigating-gosu-security-concerns/
+    https://archive.is/wip/98TVx
 
-Additionally, due to the fact that `gosu` is using Docker's own code for processing these `user:group`, it has exact 1:1 parity with Docker's own `--user` flag.
-
-If you're curious about the edge cases that `gosu` handles, see [`Dockerfile.test-alpine`](Dockerfile.test-alpine) for the "test suite" (and the associated [`test.sh`](test.sh) script that wraps this up for testing arbitrary binaries).
-
-(Note that `sudo` has different goals from this project, and it is *not* intended to be a `sudo` replacement; for example, see [this Stack Overflow answer](https://stackoverflow.com/a/48105623/433558) for a short explanation of why `sudo` does `fork`+`exec` instead of just `exec`.)
-
-## Alternatives
-
-### `setpriv`
-
-Available in newer `util-linux` (`>= 2.32.1-0.2`, in Debian; https://manpages.debian.org/buster/util-linux/setpriv.1.en.html):
-
-console
-$ docker run -it --rm buildpack-deps:buster-scm setpriv --reuid=nobody --regid=nogroup --init-groups ps faux
-USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
-nobody       1  5.0  0.0   9592  1252 pts/0    RNs+ 23:21   0:00 ps faux
-
-
-### `chroot`
-
-With the `--userspec` flag, `chroot` can provide similar benefits/behavior:
-
-console
-$ docker run -it --rm ubuntu:trusty chroot --userspec=nobody / ps aux
-USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
-nobody       1  5.0  0.0   7136   756 ?        Rs+  17:04   0:00 ps aux
-
-
-### `su-exec`
-
-In the Alpine Linux ecosystem, [`su-exec`](https://github.com/ncopa/su-exec) is a minimal re-write of `gosu` in C, making for a much smaller binary, and is available in the `main` Alpine package repository.  However, as of version 0.2 it has [a pretty severe parser bug](https://github.com/ncopa/su-exec/pull/26) that hasn't been in a release for many years (and which the buggy behavior is that typos lead to running code as root unexpectedly 😬).
-
-### Others
-
-I'm not terribly familiar with them, but a few other alternatives I'm aware of include:
+https://github.com/tianon/gosu/pull/154/files
 
-- `chpst` (part of `runit`)
+
\ No newline at end of file
diff --git a/go.sum b/_old_go.sum
similarity index 100%
rename from go.sum
rename to _old_go.sum
-- 
2.49.0


From 6039cc1eebcf21bb1114b3b380650063184b3da5 Mon Sep 17 00:00:00 2001
From: Yoni Water Man 
Date: Mon, 7 Jul 2025 16:59:18 +0000
Subject: [PATCH 2/3] upgraded versions of go and libs

---
 Dockerfile |  5 ++++-
 go.mod     | 11 ++++++++---
 go.sum     |  4 ++++
 3 files changed, 16 insertions(+), 4 deletions(-)
 create mode 100644 go.sum

diff --git a/Dockerfile b/Dockerfile
index df6ed0a..886c865 100644
--- a/Dockerfile
+++ b/Dockerfile
@@ -1,4 +1,4 @@
-FROM golang:1.20.5-bookworm
+FROM golang:1.24-bookworm as builder
 
 RUN set -eux; \
 	apt-get update; \
@@ -50,3 +50,6 @@ RUN ARCH=riscv64  GOARCH=riscv64     gosu-build-and-test.sh
 RUN ARCH=s390x    GOARCH=s390x       gosu-build-and-test.sh
 
 RUN set -eux; ls -lAFh /go/bin/gosu-*; file /go/bin/gosu-*
+
+FROM scratch
+COPY --from=builder /go/bin/* /go/bin/
\ No newline at end of file
diff --git a/go.mod b/go.mod
index 9f8577f..6b09a76 100644
--- a/go.mod
+++ b/go.mod
@@ -1,8 +1,13 @@
 module github.com/tianon/gosu
 
-go 1.20
+go 1.24
 
 require (
-	github.com/moby/sys/user v0.1.0
-	golang.org/x/sys v0.1.0
+	// Get latest versions:
+	// https://pkg.go.dev/github.com/moby/sys/user
+	github.com/moby/sys/user v0.4.0
+	// https://pkg.go.dev/golang.org/x/sys
+	golang.org/x/sys v0.33.0
 )
+
+// Regenerate go.usm with go mod tidy
diff --git a/go.sum b/go.sum
new file mode 100644
index 0000000..67f6786
--- /dev/null
+++ b/go.sum
@@ -0,0 +1,4 @@
+github.com/moby/sys/user v0.4.0 h1:jhcMKit7SA80hivmFJcbB1vqmw//wU61Zdui2eQXuMs=
+github.com/moby/sys/user v0.4.0/go.mod h1:bG+tYYYJgaMtRKgEmuueC0hJEAZWwtIbZTB+85uoHjs=
+golang.org/x/sys v0.33.0 h1:q3i8TbbEz+JRD9ywIRlyRAQbM0qF7hu24q3teo2hbuw=
+golang.org/x/sys v0.33.0/go.mod h1:BJP2sWEmIv4KK5OTEluFJCKSidICx8ciO85XgH3Ak8k=
-- 
2.49.0


From 9a7dfac5208c0db8d918f8caf0972786278e8ae9 Mon Sep 17 00:00:00 2001
From: Yoni Water Man 
Date: Mon, 7 Jul 2025 17:01:49 +0000
Subject: [PATCH 3/3] readme

---
 README.md | 2 ++
 1 file changed, 2 insertions(+)

diff --git a/README.md b/README.md
index 8dfb660..a4de924 100644
--- a/README.md
+++ b/README.md
@@ -1,4 +1,6 @@
 
+https://hub.docker.com/repository/docker/prapl4group/better-bases/tags?name=gosu&page=1
+
 https://internetworking.dev/mitigating-gosu-security-concerns/
     https://archive.is/wip/98TVx
 
-- 
2.49.0

```

