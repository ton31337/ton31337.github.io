---
layout: post
title: HTTP request validation using hashing
categories:
- blog
---

A few days ago we saw that some HTTP requests are coming directly from browsers instead of being proxied by intermediate load balancers. It looks kinda dangerous because it's possible to DoS single container(s) so bypassing our logical layer (load balancers). So we started to dig more into this. 

Since we are using IPv6 between containers and load balancers, the first idea was to block incoming HTTP traffic for containers originated using IPv4 address family. It's more than worse while IPv6 is getting a fast pace. Thus, this idea will stop working shortly.

Hence, I started to do some research what and how we can simply achieve this request validation.

We are using some external community modules for [Apache](http://httpd.apache.org/), thus first looked at [mod_rpaf](https://github.com/gnif/mod_rpaf) module. It has nicely written `RPAF_ForbidIfNotProxy` directive, which literally doesn't work in some cases (e.g.: IP address is rewritten to client's after mod_rewrite phase). As you might guess, this module was rejected. 

Second candidate was [mod_reset](https://github.com/ton31337/mod_reset). I asked myself: "Why do not validate requests using arbitrary header?". Hence, I began to brainstorm how to best implement this validation using the header. Some ideas come to my mind:

#### Timestamps (REJECTED)

Send timestamp as an arbitrary header from load balancer and compare it on the backend. It's fine, but it's very trivial to work around (by setting this header manually from the client).

#### Sequence numbers (REJECTED)

Send sequence number as arbitrary header and compare received with previous one. This is better than timestamping, but requires additional efforts, like introducing mapping between sequence number and backends.

#### proxy_pass with Authorization header (REJECTED)

Use backends with enabled basic authorization and send `Authorization` header to backends. One of the biggest drawbacks is that header is sent using base64 encoded string, which is nothing else, but plain text.

#### AES-256 (REJECTED)

Encrypt hostname or something else with [AES-256](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) and send [base64](https://en.wikipedia.org/wiki/Base64) encoded string as header to backends. Backends, in turn, decrypt this string and if they match, all good. 

Again, this requires additional efforts to implement this, especially on Apache side, because [apr crypto](https://apr.apache.org/docs/apr-util/1.6/group___a_p_r___util___crypto.html) routines just suck and Openresty is not able to do AES encryption natively.

#### MD5/SHA1/Bcrypt (ACCEPTED)

Why only those three? Because [apr md5](https://apr.apache.org/docs/apr/2.0/group___a_p_r___m_d5.html) routines support only those types. But it's good enough. 

We set the arbitrary header to value of generated hash (Bcrypt):

```
set $hash '';
access_by_lua_file '
  local bcrypt = require("bcrypt")
  local hash = string.gsub(bcrypt.digest("secret"), "^($2b$)", "$2a$")
';
proxy_set_header X-Arbitrary-Header $hash;
```

And backends do the following validations:

```
			             |----------------|
                        request ---->| header_exists? |-- NO --> 403
                                     |----------------|
                                               |
                                              YES
                                               |
                                     |----------------|
                                     | hash is valid? |-- NO --> 403
                                     |----------------|
                                               |
                                              YES
                                               |
                                     |----------------|
                                     |      uniq?     |-- NO --> 403
                                     |----------------|
                                               |
                                               |
                                               |--------- YES --> 200
```

I added additional trivial hook for Apache which does this aforementioned validation:

```
static int reset_check_handler(request_rec *r)
{
        const char *header = NULL;
        reset_config *conf = (reset_config *) ap_get_module_config(r->server->module_config, &reset_module);
        if (conf->enable) {
                if (conf->deny_header) {
                        if (!r->hostname)
                                goto req_ok;

                        /* Check if request contains arbitrary header */
                        header = (char *) apr_table_get(r->headers_in, conf->deny_header);
                        if (!header)
                                return HTTP_FORBIDDEN;

#ifdef MOD_RESET_AUTH_KEY
                        /* Do some validation by secret */
                        if (apr_password_validate(MOD_RESET_AUTH_KEY, header) != APR_SUCCESS)
                                return HTTP_FORBIDDEN;

                        if (!conf->hash)
                                goto req_ok;

                        if (apr_strnatcmp(conf->hash, header) == 0)
                                return HTTP_FORBIDDEN;
#endif
                }
        }
req_ok:
        conf->hash = (char *)header;
        return OK;
}
```

### Conclusion

* When someone says "pretty simple" regarding cryptography, it's often neither pretty nor simple;
* The challenge is to keep simple things simple while allowing complex things to be possible;
* Benchmarks shown no performance penalty.
