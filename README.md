# NGINX with HTTP/3 and ModSecurity for CentOS 6 (EL6)

This repository provides **unofficial NGINX RPM packaging and build artifacts for CentOS 6 / Enterprise Linux 6 (EL6) x86_64**.

It is intended for legacy-system compatibility experiments and RPM build verification. The repository includes a ModSecurity v3 build and the supporting RPMs needed for this EL6 build environment. Availability of HTTP/3 or other optional features must be confirmed against the actual NGINX build and runtime configuration.

> **Security warning:** CentOS 6 is end-of-life. These packages do not restore vendor security support for the operating system, kernel, or other dependencies. Use an isolated test environment, and do not expose an EL6 host to untrusted networks without a separate, carefully reviewed risk assessment.

## Components and versions

The following packages have been prepared for the EL6 build environment:

| Component | Version | Purpose |
| --- | --- | --- |
| ModSecurity (libmodsecurity v3) | 3.0.17-1.el6 | WAF engine; install before NGINX when the NGINX build links against it |
| YAJL | 2.0.4-4.el6 | JSON library used by the ModSecurity build |
| LMDB | 0.9.22-2.el6 | Supporting library |
| PCRE2 | 10.23-2.el6 | Regular-expression library |
| zstd / libzstd | 1.5.4-1.el6 | Compression library and development files |
| Google Test (gtest) | 1.6.0-2.el6 | Build/test dependency |

Check the actual NGINX version and compile-time options with `nginx -V`. This README does not assume a particular NGINX or OpenSSL version because those package details have not been verified here.

## Repository layout

A suggested repository layout is:

```text
RPMS/           Binary RPM packages
SRPMS/          Source RPM packages
SPECS/          RPM spec files
Logs/           Build logs
rpmbuild_mod/   ModSecurity build workspace (if retained)
rpmbuild_nginx/ NGINX build workspace (if retained)
README.md       Documentation
```

The actual directory structure may differ. The RPM listing provided for this project includes individual package files as well as the `rpmbuild_mod` and `rpmbuild_nginx` directories.

## Installation order (important)

**Install supporting runtime libraries first, then ModSecurity, then NGINX.**

For the build represented by the listed RPMs, **YAJL must be available for ModSecurity**, and **ModSecurity must be installed before an NGINX RPM that dynamically links to `libmodsecurity.so.3`**.

The exact RPM dependencies are determined by the binaries and the RPM metadata. Before installation, inspect the package requirements:

```bash
rpm -qpR modsecurity-3.0.17-1.el6.x86_64.rpm
rpm -qpR /path/to/nginx-VERSION-RELEASE.el6.x86_64.rpm
```

### 1. Install runtime libraries

Run these commands from the directory containing the downloaded or locally built RPM files:

```bash
sudo yum localinstall \
  ./yajl-2.0.4-4.el6.x86_64.rpm \
  ./lmdb-libs-0.9.22-2.el6.x86_64.rpm \
  ./pcre2-10.23-2.el6.x86_64.rpm \
  ./libzstd-1.5.4-1.el6.x86_64.rpm
```

Other packages may be required depending on how ModSecurity and NGINX were built. Confirm and resolve those dependencies rather than forcing installation with `rpm --nodeps`.

### 2. Install ModSecurity before NGINX

```bash
sudo yum localinstall ./modsecurity-3.0.17-1.el6.x86_64.rpm
```

Locate the shared library supplied by the package:

```bash
rpm -ql modsecurity | grep 'libmodsecurity\.so'
ldconfig -p | grep modsecurity
```

The installation path of `libmodsecurity.so.3` is determined by the RPM build. **Do not assume the AlmaLinux 9 path `/opt/modsecurity/lib64/` also applies to EL6.** If a nonstandard library directory is used, configure the dynamic linker appropriately and run `ldconfig`.

### 3. Install NGINX

Install the **actual binary NGINX RPM** from the build output; replace the example filename with the one produced by your build:

```bash
sudo yum localinstall /path/to/nginx-VERSION-RELEASE.el6.x86_64.rpm
```

Verify the installed package and linked libraries:

```bash
rpm -q nginx modsecurity yajl
nginx -V 2>&1
ldd /usr/sbin/nginx | grep -Ei 'modsecurity|not found'
```

If NGINX is linked dynamically against ModSecurity, the `ldd` output should resolve `libmodsecurity.so.3` without `not found`.

CentOS 6 uses SysV service commands, not `systemctl`:

```bash
sudo service nginx configtest
sudo service nginx restart
# Optional: enable startup at boot
sudo chkconfig nginx on
```

If `service nginx configtest` is not supported by the installed init script, use `sudo nginx -t` instead.

## Build dependencies

The following development packages are present in the supplied EL6 RPM set and may be needed to build ModSecurity, its dependencies, or the NGINX module integration:

```text
gtest-1.6.0-2.el6.x86_64.rpm
gtest-devel-1.6.0-2.el6.x86_64.rpm
libzstd-1.5.4-1.el6.x86_64.rpm
libzstd-devel-1.5.4-1.el6.x86_64.rpm
libzstd-static-1.5.4-1.el6.x86_64.rpm
lmdb-0.9.22-2.el6.x86_64.rpm
lmdb-devel-0.9.22-2.el6.x86_64.rpm
lmdb-libs-0.9.22-2.el6.x86_64.rpm
pcre2-10.23-2.el6.x86_64.rpm
pcre2-devel-10.23-2.el6.x86_64.rpm
pcre2-static-10.23-2.el6.x86_64.rpm
yajl-2.0.4-4.el6.x86_64.rpm
yajl-devel-2.0.4-4.el6.x86_64.rpm
zstd-1.5.4-1.el6.x86_64.rpm
```

**Not all packages in the build output are mandatory runtime or compile-time dependencies.** `*-debuginfo` packages are for debugging, `*-static` packages are only required when performing relevant static links, and documentation packages such as `lmdb-doc` are optional. `modsecurity-3.0.17-1.el6.src.rpm` is a source RPM, not an installable runtime dependency.

For source builds, inspect the SPEC files for `BuildRequires` and use `rpm -qpR` for the packaged runtime requirements. Do not bulk-install every RPM in the output directory, especially when it contains source, debuginfo, or alternate binary packages.

## Verify ModSecurity integration

Check whether the NGINX build includes the ModSecurity connector:

```bash
nginx -V 2>&1 | grep -i modsecurity
```

Depending on the SPEC, it may have been built as a static module or a dynamic module. A static build may show `--add-module=...ModSecurity-nginx`; a dynamic build may show `--add-dynamic-module=...` and require an NGINX `load_module` directive.

Check the dependency and any embedded runtime library paths:

```bash
ldd /usr/sbin/nginx | grep -Ei 'modsecurity|not found'
readelf -d /usr/sbin/nginx | grep -E 'RPATH|RUNPATH'
```

No `RPATH` or `RUNPATH` is required when ModSecurity is installed in a standard linker path or its directory is registered with the dynamic linker.

## ModSecurity and OWASP CRS

**OWASP Core Rule Set (CRS) is not included unless explicitly bundled in the RPM.** Install a separately verified CRS release and adjust the include paths to match the files on the host.

Example NGINX server configuration:

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

Example `/etc/nginx/modsec/main.conf`:

```apache
Include /etc/nginx/modsec/modsecurity.conf
Include /etc/nginx/modsec/crs-setup.conf
Include /etc/nginx/modsec/rules/*.conf
```

Begin testing with:

```apache
SecRuleEngine DetectionOnly
```

After reviewing audit logs, validating rule behavior, and tuning false positives, blocking mode can be enabled:

```apache
SecRuleEngine On
```

Test the configuration:

```bash
sudo nginx -t
sudo service nginx restart
```

Once CRS and blocking rules are configured, a basic local WAF probe is:

```bash
curl -i -H 'Host: example.test' \
  'http://127.0.0.1/?id=1%20UNION%20SELECT%201,2,3'
```

A `403 Forbidden` response is **possible** when blocking is enabled and the configured CRS rules and anomaly threshold match the request. A `200` response does not by itself establish that the connector failed: check the ModSecurity audit logs and relevant CRS rules.

## HTTP/3 / QUIC verification

HTTP/3 requires explicit NGINX build and listener configuration, appropriate TLS support, UDP reachability, and a client capable of HTTP/3. **A repository name or build option alone does not demonstrate working HTTP/3.**

Check the configured build options:

```bash
nginx -V 2>&1 | grep -E 'http_v3|OpenSSL|quic'
```

From a modern client that supports HTTP/3:

```bash
curl -I --http3-only https://your-test-host.example/
```

Confirm the protocol reported by the client and, when possible, by the NGINX access log. Older EL6-provided `curl` versions should not be assumed to support `--http3-only`.

## kTLS compatibility note

kTLS requires kernel and TLS-library support. **The stock CentOS 6 kernel does not provide modern Linux kTLS support.** An OpenSSL build flag by itself cannot make kTLS available on that kernel. Document kTLS as enabled only after testing on a suitable kernel with appropriate cipher suite, TLS library, and NGINX settings.

## Production and maintenance notes

- Prefer a supported operating system for internet-facing deployments.
- Keep build-time dependencies, patches, compiler versions, SPEC files, and build logs under version control where licensing and size permit.
- Validate ModSecurity and CRS against false positives before enabling blocking.
- Verify all dynamic libraries and NGINX modules after every RPM update.
- Treat any EL6 deployment as a legacy exception with an explicit security and migration plan.

## Licensing

NGINX, ModSecurity, YAJL, LMDB, PCRE2, zstd, and other dependencies are distributed under their own respective licenses. Review the license notices in each upstream source package and the RPM metadata. This repository does not change upstream licensing terms.

## Disclaimer

This repository is unofficial. It is not maintained, endorsed, or supported by the CentOS Project, Red Hat, NGINX, OWASP, ModSecurity, or their upstream maintainers.

These packages are provided **as is**, without warranty. Use them at your own risk and test thoroughly in an isolated environment before any deployment.

