---
layout: post
title: Limit bandwidth by Openresty
categories:
- blog
---

Nginx has two cool features to limit bandwidth for responses:
* [limit_rate](http://nginx.org/en/docs/http/ngx_http_core_module.html#limit_rate)
* [limit_rate_after](http://nginx.org/en/docs/http/ngx_http_core_module.html#limit_rate_after)

For instance, this works well, but absolutely not well for video streamings:

```
location / {
    limit_rate_after 500k;
    limit_rate 50k;
    ...
}
```

This means, that you can buffer the stream for 500k then rate drops until 50k, but if you watch the video using the browser/player/whatever you will notice that with every new buffer chunk the browser will issue a new request which means new limits.

What if I want to push those limits only for video streamings and not for the poor `wget`? I mean - with every next request to the same URI lower limits in reverse exponentially manner, like reverse TCP congestion behavior.

#### Everything Should Be Made as Simple as Possible, But Not Simpler

TL;DR; - create `SHA256` hash for full URI with 5 minutes expiration time and increment the counter with every new request. This counter is used for exponentially decreasing the bandwidth for responses (bytes per second):
```
  26214400
  6553600
  1638400
  409600
  102400
  25600
  6400
  ...
```
router.conf:
```
    location / {
        set $limit_rate '';
        set $limit_rate_after '';
        access_by_lua_file conf/router.lua;
        proxy_buffering on;
        ...
    }
```
NOTE: If you use `proxy_pass` you must enable `proxy_buffering on`. 

router.lua:
```
local request_count = function ()
  local resty_sha256 = require "resty.sha256"
  local str = require "resty.string"
  local sha256 = resty_sha256:new()
  sha256:update(ngx.var.http_host .. ngx.var.request)
  local digest = sha256:final()
  local key = "request:" .. str.to_hex(digest)

  local ok, err = redis:connect("127.0.0.1")
  if not ok then
    return nil, err
  end

  local ok, err = redis:auth("something")
  if not ok then
    return nil, err
  end

  local data, _ = redis:get(key)
  redis:incr(key)
  redis:expire(key, 300)

  return data
end

local bw_limit = 104857600
if request_count() ~= ngx.null then
  bw_limit = math.max(10240, math.floor(bw_limit / 2 ^ request_count()))
end

ngx.var.limit_rate = bw_limit
ngx.var.limit_rate_after = 8388608
```


What is your biggest wish? To eject this shit out of my ass. 
