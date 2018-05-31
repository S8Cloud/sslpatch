# Usefull SSL (OpenSSL & BoringSSL) & Nginx Patch Bundle

# For OpenSSL

### OpenSSL 1.1.0h

```
wget https://www.openssl.org/source/openssl-1.1.0h.tar.gz && tar zxf openssl-1.1.0h.tar.gz && cd openssl-1.1.0h

# "double" ecdhx25519 performance on 64-bit platforms. Upstream openssl 1.1.1-dev
patch -p1 < /path/to/sslpatch/OpenSSL1.1.0h-double-performance-ecdhx-25519.patch

# improve ECDSA sign by 30-40%. Upstream openssl 1.1.1-dev
patch -p1 < /path/to/sslpatch/OpenSSL1.1.0h-improve-ECDSA-sign-30-40%.patch

# fix CVE-2018-0737 openssl RSA key generation cache timing vulnerability
patch -p1 < /path/to/sslpatch/OpenSSL1.1.0h-cache-timing-rsa-key-gen.patch

# Support OpenSSL Equal preference cipher groups (BoringSSL feature)
patch -p1 < /path/to/sslpatch/OpenSSL1.1.0h-equal-preference-cipher-groups.patch

# Thanks much to https://gitlab.com/buik/ 
```

Sample nginx configuration:

```
ssl_protocols TLSv1.2 TLSv1.1 TLSv1;
ssl_ciphers '[ECDHE-ECDSA-AES128-GCM-SHA256|ECDHE-ECDSA-CHACHA20-POLY1305]:[ECDHE-RSA-AES128-GCM-SHA256|ECDHE-RSA-CHACHA20-POLY1305]:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES128-SHA';
ssl_ecdh_curve X25519:prime256v1:secp384r1;
ssl_prefer_server_ciphers on;
```

### OpenSSL 1.0.2o

```
wget https://www.openssl.org/source/openssl-1.0.2o.tar.gz && tar zxf openssl-1.0.2o.tar.gz && cd openssl-1.0.2o

# Cloudflare patch to add draft-chacha-cipher and chacha-prefered features
patch -p1 < /path/to/sslpatch/OpenSSL1.0.2o-chacha20-poly1305-draft.patch

# fix CVE-2018-0737 openssl RSA key generation cache timing vulnerability
patch -p1 < /path/to/sslpatch/OpenSSL1.0.2o-cache-timing-rsa-key-gen.patch
```

Sample nginx configuration:

```
ssl_protocols TLSv1.2 TLSv1.1 TLSv1;
ssl_ciphers 'EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:RSA+3DES:!MD5;
ssl_ecdh_curve prime256v1:secp384r1;
ssl_prefer_server_ciphers on;
```

### OpenSSL 1.1.1-dev

Here I forked OpenSSL 1.1.1 pre2 pre7 and pre8(master) patchs form [Hakase](https://github.com/hakasenyang/openssl-patch).

These patches can enable BoringSSL's Equal Preference feature for OpenSSL 1.1.1 dev version and let users define TLS1.3 ciphers in the nginx config file.

Noting that OpenSSL 1.1.1-dev has many different preview versions and **none of them should be used at production environment**.

| OpenSSL pre version | Supported TLSv1.3 version |
| :--- |  ---: |
| pre2 | draft 23 (the only pre version to support this draft) |
| pre3 to 6 | rare used darft version (like draft 26,27) |
| pre7 and higher | draft 28 (the final accepted tlsv1.3 version) |

Till now, latest stable version of modern browsers (Chrome & Firefox) support TLSv1.3 draft 23 and ubstable version support draft 28.

Detailed configuration guide can be found at [Hakase](https://github.com/hakasenyang/openssl-patch/blob/master/README.md)

### BoringSSL Patch

BoringSSL is a fork of OpenSSL that is designed to meet Google's needs.

BoringSSL's TLSv1.3 (especially draft 23) was production-used by google and  Cloudflare for a long time.

I myself am using BoringSSL and I forked BoringSSL and made TLSv1.3 enabled by default (using the `BoringSSL-enable-TLS1.3.patch`)

* to enable tlsv1.3 draft 23 (stable & recommended)

```
# Note that golang and cmake should be installed first
git clone -b chromium-stable https://github.com/S8Cloud/boringssl.git && cd boringssl
cmake -DCMAKE_BUILD_TYPE=Release && make -j2
cd .. && mkdir -p .openssl/lib && cd .openssl && ln -s ../include .
cd .. && cp build/crypto/libcrypto.* build/ssl/libssl.* .openssl/lib && cd ..
```

And then complie nginx using `--with-openssl=/path/to/boringssl`

* to enable tlsv1.3 draft 23 & draft 28 (master and unstable)

```
git clone -b master https://boringssl.googlesource.com/boringssl && cd boringssl
patch -p1 < /path/to/sslpatch/BoringSSL-enable-TLS1.3.patch
```

building process is the same with above.

* or you need tlsv1.3 draft 18

Just clone `2987` branch of my boringssl fork and build it.

```
git clone -b 2987 https://github.com/S8Cloud/boringssl.git
```

# For Nginx

Cloudflare made there 3 excellent patches open source: `dynamic_tls_records patch` , `http2_spdy patch` and `http2_hpack patch` but did not upgarde for quite a long time.

My frient [kn007](http://kn007.net/) combined theme together and adapted it to the latest nginx version.

```
wget -c https://nginx.org/download/nginx-1.14.0.tar.gz && tar zxf nginx-1.14.0.tar.gz && cd nginx-1.14.0
patch -p1 < /path/to/sslpatch/Nginx_cloudflare_patch_combined_kn007.patch
```

* if you are using BoringSSL and you want to enable OCSP Staping for your server patch the following patch to `Nginx`

```
patch -p1 < /path/to/sslpatch/patch BoringSSL-enable-OCSP.patch
```

activated it by:

```
openssl ocsp -no_nonce \
    -respout /path/to/ocsp.resp \
    -issuer  intermediate.pem \
    -cert    website.pem \
    -CAfile  ca-bundle.pem \
    -VAfile  ca-bundle.pem \
    -url     http://ocsp.yourcaserver.com/ 
```
and add these to nginx config:

```
ssl_stapling on;
ssl_stapling_file /path/to/ocsp.resp;
```

## Special Thanks

**Hakase, kn007, CarterLi, buik**