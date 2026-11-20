# NGINX with HTTP/3 and ModSecurity for CentOS 6 (EL6)

This repository provides **unofficial NGINX RPM packaging and build artifacts for Enterprise Linux 6 (CentOS 6 / RHEL 6), x86_64**, for compatibility testing with the ModSecurity v3 NGINX connector and an HTTP/3-capable NGINX build.

**Important:** Install the EL6 YAJL backport and ModSecurity before installing or starting this NGINX build. For the EL6 SysVinit configuration used here, configure NGINX to write its PID file to **`/var/run/nginx.pid`** and keep the init script's PID path consistent.

> **Security warning:** CentOS 6 is end-of-life. These packages do not restore OS, kernel, OpenSSL, or other vendor security support. This project is intended for isolated testing and legacy compatibility verification, not as an endorsement of internet-facing EL6 deployments.

## Related repositories

The following projects are **separate packages**, not automatically installed by cloning this NGINX repository:

| Project | Repository | Role |
| --- | --- | --- |
| YAJL for EL6 | [redadmin-k/yajl-el6](https://github.com/redadmin-k/yajl-el6) | Backported YAJL runtime (`libyajl.so.2`) and development files |
| ModSecurity for EL6 | [redadmin-k/modsecurity-el6](https://github.com/redadmin-k/modsecurity-el6) | ModSecurity v3 engine (`libmodsecurity.so.3`) |
| NGINX for EL6 | [redadmin-k/nginx-http3-modsecurity-el6](https://github.com/redadmin-k/nginx-http3-modsecurity-el6) | NGINX RPM build with ModSecurity integration |

The intended dependency order is:

```text
EL6-compatible runtime libraries
        |
        v
YAJL 2.0.4 (yajl-el6)
        |
        v
ModSecurity v3 (modsecurity-el6)
        |
        v
NGINX + ModSecurity-nginx connector
        |
        v
ModSecurity rules / optional OWASP CRS
```

## Components

The EL6 build environment described by this repository includes the following packages. Check the RPM metadata in the version actually downloaded or built:

| Component | EL6 package version shown in this project | Use |
| --- | --- | --- |
| YAJL | `2.0.4-4.el6` | ModSecurity JSON library dependency |
| ModSecurity v3 | `3.0.17-1.el6` | ModSecurity runtime engine |
| LMDB | `0.9.22-2.el6` | Supporting library |
| PCRE2 | `10.23-2.el6` | Regular-expression library |
| zstd / libzstd | `1.5.4-1.el6` | Compression library |
| Google Test | `1.6.0-2.el6` | Build/test dependency, not normally needed at runtime |

The precise NGINX version, TLS library, and optional modules depend on the NGINX RPM build. Confirm them with `nginx -V` instead of inferring them from the repository name.

## Repository layout

```text
README.md
SPECS/    RPM spec files
SRPMS/    Source RPM packages
RPMS/     Built binary RPM packages
Logs/     Build logs
```

Individual RPMs may be stored in `RPMS/` or in architecture subdirectories. Inspect the files before choosing the installation commands.

## Installation (EL6)

**Install YAJL first, ModSecurity second, and NGINX third.** Use the RPMs built for **EL6**, not binary RPMs for EL7.

The examples below use absolute placeholder paths for separate repository checkouts. Adjust them to the actual RPM locations.

### 1. Install the EL6 YAJL backport

Download or build the `yajl` runtime RPM from:

https://github.com/redadmin-k/yajl-el6

```bash
sudo yum localinstall \
  /path/to/yajl-el6/RPMS/x86_64/yajl-2.0.4-4.el6.x86_64.rpm
```

`yajl-devel` is also needed when **building** ModSecurity from source, but not generally to run an already-built NGINX/ModSecurity RPM:

```bash
sudo yum localinstall \
  /path/to/yajl-el6/RPMS/x86_64/yajl-devel-2.0.4-4.el6.x86_64.rpm
```

Verify that YAJL is installed and its runtime library is visible:

```bash
rpm -q yajl
ldconfig -p | grep libyajl
```

Expected library name: `libyajl.so.2` (often under `/usr/lib64/`).

### 2. Install other required EL6 runtime libraries

Depending on the `modsecurity` and `nginx` RPM builds, supporting packages can include LMDB, PCRE2, and libzstd. For the versions represented in this repository, an example is:

```bash
sudo yum localinstall \
  /path/to/RPMS/lmdb-libs-0.9.22-2.el6.x86_64.rpm \
  /path/to/RPMS/pcre2-10.23-2.el6.x86_64.rpm \
  /path/to/RPMS/libzstd-1.5.4-1.el6.x86_64.rpm
```

Only install packages actually required by your RPM dependency graph. Check dependency requirements before installing:

```bash
rpm -qpR /path/to/modsecurity-3.0.17-1.el6.x86_64.rpm
rpm -qpR /path/to/nginx-*.el6.x86_64.rpm
```

Do not use `rpm --nodeps` to bypass missing libraries.

### 3. Install ModSecurity for EL6

Download or build the EL6 ModSecurity RPM from:

https://github.com/redadmin-k/modsecurity-el6

```bash
sudo yum localinstall \
  /path/to/modsecurity-el6/RPMS/x86_64/modsecurity-3.0.17-1.el6.x86_64.rpm
```

The repository's documented installation prefix is:

```text
/opt/modsecurity
```

An example shared library location is:

```text
/opt/modsecurity/lib64/libmodsecurity.so.3
```

Verify the actual files and resolve transitive runtime dependencies:

```bash
rpm -q yajl modsecurity
rpm -ql modsecurity | grep 'libmodsecurity\.so'
sudo ldconfig
ldconfig -p | grep -E 'libyajl|libmodsecurity'
ldd /opt/modsecurity/lib64/libmodsecurity.so.3 | grep -E 'not found|yajl'
```

If the actual RPM installs its `.so` elsewhere, substitute the path reported by `rpm -ql`. Any `not found` entry must be fixed before starting NGINX. For a nonstandard shared-library directory, verify the corresponding file under `/etc/ld.so.conf.d/` and run `ldconfig`.

### 4. Install the NGINX EL6 RPM

Install the NGINX binary RPM built by this repository (replace the example path with the actual filename):

```bash
sudo yum localinstall /path/to/nginx-VERSION-RELEASE.el6.x86_64.rpm
```

Check the installed binary and compile-time options:

```bash
rpm -q nginx modsecurity yajl
nginx -V 2>&1
nginx -V 2>&1 | grep -i modsecurity
```

For a statically linked ModSecurity connector, `nginx -V` may include `--add-module=...ModSecurity-nginx`. A dynamic connector may instead require a `load_module` directive at the beginning of the main NGINX configuration file.

Check runtime library resolution:

```bash
ldd "$(command -v nginx)" | grep -Ei 'modsecurity|not found'
```

If the connector is a **dynamic** module, examine the module's `.so` with `ldd` as well; `libmodsecurity` need not appear directly in the main NGINX binary's dependencies.

## Required EL6 PID file setting

**For this EL6 NGINX SysVinit setup, use the following directive in `/etc/nginx/nginx.conf`:**

```nginx
pid /var/run/nginx.pid;
```

It must be placed in the **main context**, outside the `events`, `http`, and `server` blocks. Example:

```nginx
user nginx;
worker_processes auto;
pid /var/run/nginx.pid;
error_log /var/log/nginx/error.log warn;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    include /etc/nginx/conf.d/*.conf;
}
```

The relevant `user` and `include` paths should match the installed RPM's configuration. **Do not replace a working NGINX configuration wholesale with this minimal example.**

### Why `/var/run/nginx.pid` matters

The PID file identifies the NGINX master process. In this EL6 packaging/setup, the SysVinit script may look for `/var/run/nginx.pid` when checking status or sending stop/restart signals. If NGINX writes its PID elsewhere, the script may report an incorrect status or fail to stop/restart NGINX reliably. If the chosen PID directory is absent or unwritable, the NGINX master itself may fail to start.

**`/var/run` is not a universal NGINX requirement.** Another PID path works if the NGINX configuration, build defaults, service script, permissions, and management commands all agree. The `/var/run/nginx.pid` setting documents the path needed by this particular EL6 setup.

If rebuilding NGINX, also prefer a matching configure argument:

```text
--pid-path=/var/run/nginx.pid
```

The `pid` directive in `nginx.conf` can override the build-time default. Keep the SysVinit script consistent with either choice.

Check both settings:

```bash
nginx -V 2>&1 | grep -o -- '--pid-path=[^ ]*'
grep -n '^[[:space:]]*pid[[:space:]]' /etc/nginx/nginx.conf
grep -nE 'pidfile|nginx\.pid|PIDFILE' /etc/init.d/nginx
ls -ld /var/run
```

Test NGINX and start it through the EL6 service script:

```bash
sudo nginx -t
sudo service nginx start
sudo service nginx status
ls -l /var/run/nginx.pid
cat /var/run/nginx.pid
ps -ef | grep '[n]ginx: master'
```

If no PID file is created, check the error log and directory permissions:

```bash
tail -n 50 /var/log/nginx/error.log
ls -ld /var/run
```

**Do not create an arbitrary PID file manually:** NGINX must write the PID of its actual master process. Likewise, do not change permissions on all of `/var/run` to make this work. Inspect the service account, startup mode, and error log instead.

CentOS 6 uses **SysVinit**, not systemd. When configuration is valid:

```bash
sudo service nginx restart
sudo chkconfig nginx on  # optional
```

If the installed RPM does not supply `/etc/init.d/nginx`, use the service mechanism actually provided by that package.

## Basic ModSecurity configuration

ModSecurity is **not activated solely by installing its RPM**. Configure the NGINX connector and a rules file.

If the ModSecurity RPM contains the recommended configuration files at the documented paths:

```bash
sudo mkdir -p /etc/nginx/modsec
sudo cp /opt/modsecurity/share/modsecurity.conf-recommended \
  /etc/nginx/modsec/modsecurity.conf
sudo cp /opt/modsecurity/share/unicode.mapping \
  /etc/nginx/modsec/unicode.mapping
```

Check the `SecUnicodeMapFile` setting in `modsecurity.conf` and adjust its path to the copied `unicode.mapping` file if necessary.

For initial verification, edit `/etc/nginx/modsec/modsecurity.conf` to use:

```apache
SecRuleEngine DetectionOnly
```

Create `/etc/nginx/modsec/main.conf`:

```apache
Include /etc/nginx/modsec/modsecurity.conf
```

Example NGINX server configuration (inside the `http` context):

```nginx
server {
    listen 80;
    server_name example.test;

    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
}
```

Test the configuration:

```bash
sudo nginx -t
sudo service nginx restart
```

### Optional OWASP Core Rule Set

OWASP CRS is **not assumed to be bundled**. Once a compatible CRS installation is present, add its paths to `/etc/nginx/modsec/main.conf`, for example:

```apache
Include /etc/nginx/modsec/modsecurity.conf
Include /etc/nginx/modsec/crs-setup.conf
Include /etc/nginx/modsec/rules/*.conf
```

Do not add CRS include paths before the referenced files exist. Test the exact CRS version and rule exclusions with your ModSecurity v3 build. After reviewing detections and false positives, blocking may be enabled:

```apache
SecRuleEngine On
```

A local SQL-injection-like probe is:

```bash
curl -i -H 'Host: example.test' \
  'http://127.0.0.1/?id=1%20UNION%20SELECT%201,2,3'
```

A `403 Forbidden` response depends on active blocking rules and their configured actions. A `200 OK` response does not, by itself, prove the connector is broken; inspect the ModSecurity audit log and loaded CRS rules.

## HTTP/3 / QUIC verification

HTTP/3 capability must be **verified**, not assumed from the repository name.

```bash
nginx -V 2>&1 | grep -Ei 'http_v3|quic|openssl'
```

For an HTTPS/QUIC listener, confirm the actual NGINX version's syntax, TLS library compatibility, UDP port reachability, and certificate configuration. From a separate modern client with HTTP/3 support:

```bash
curl -I --http3-only https://your-test-host.example/
```

The stock CentOS 6 `curl` is too old to be assumed to support `--http3-only`. Verify the negotiated protocol using a capable client and server access logs. An `--with-http_v3_module` build flag alone is not proof of a working HTTP/3 deployment.

## Build considerations

Building these packages on EL6 may require backported libraries, a newer compiler toolchain, custom/archived repositories, and a configured `mock` buildroot.

- For ModSecurity builds, include the EL6 `yajl-devel` package in the buildroot.
- For NGINX builds, use the installed ModSecurity headers and libraries from the actual RPM; `/opt/modsecurity/lib64/pkgconfig` may need to be added to `PKG_CONFIG_PATH`.
- Verify ModSecurity connector compatibility with the NGINX source version.
- The stock CentOS 6 kernel does not offer modern kTLS support; neither an OpenSSL build flag nor an HTTP/3-capable NGINX binary changes that limitation.

Example build environment variable:

```bash
export PKG_CONFIG_PATH=/opt/modsecurity/lib64/pkgconfig${PKG_CONFIG_PATH:+:$PKG_CONFIG_PATH}
```

## Troubleshooting checklist

| Symptom | First checks |
| --- | --- |
| NGINX fails on startup | `nginx -t`, `/var/log/nginx/error.log`, PID location/permissions |
| `service nginx status` disagrees with the running process | `/etc/init.d/nginx` PID path versus `pid /var/run/nginx.pid;` |
| `libmodsecurity.so.3` not found | Install ModSecurity first; inspect `ldconfig -p` and `ldd` |
| `libyajl.so.2` not found | Install EL6 `yajl`; rerun `ldconfig`; check dependencies |
| `unknown directive "modsecurity"` | Verify NGINX connector build and dynamic `load_module` where applicable |
| Unexpected CRS behavior | Verify rule includes, `SecRuleEngine`, audit logs, exclusions |
| HTTP/3 not negotiated | Verify build options, TLS support, UDP listener, network path, HTTP/3 client |

## Production and maintenance notes

Use a maintained OS for internet-facing deployments. On legacy test hosts, keep RPM specs, patches, build logs, exact dependency versions, and validation results available for reproducibility. ModSecurity and OWASP CRS are defense-in-depth components, not substitutes for patching or application security fixes.

## License

NGINX, ModSecurity, YAJL, LMDB, PCRE2, and other dependencies retain their respective upstream licenses. Review the source and RPM metadata for each component. This repository does not alter upstream licensing terms.

## Disclaimer

This repository is **unofficial**. It is not maintained, endorsed, or supported by CentOS, Red Hat, NGINX, OWASP, ModSecurity, or the respective upstream maintainers.

The packages are provided **as is**, without warranty. Test carefully in an isolated environment before deployment.

