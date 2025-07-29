---
date:
  created: 2025-07-29 
  updated: 2025-07-29
draft: false
tags:
  - nginx
title: Nginx dynamic hidden redirect based on query args
icon: simple/nginx
---

# Nginx Dynamic Redirect Based on Query Args  

Recently, I faced the following scenario at work:  
- A client was preparing to launch their new site  
- 40,000+ indexed pages ea/domain, 6 domains in total  
- There were third-party developers involved in the project    
  

For backup purposes, someone ended up creating a rewrite engine like `/seo/product/<product_number>/` and wanted dynamic rewrites to it based on query args, specifically `/old-product/?id=12341-12`, which then needed to redirect to the engine, which would then rewrite further... see the problem?

When changing URLs, it's important to tell Google through redirects. However, if we do a rewrite:  
  
`/old-product/?id=12341-12` → `/seo/product/<product_number>/` → `/new-product/cool-url-12341/`  
  
Then we have a problem because the odds are that Google will think the middle point is the new URL for that page, e.g., `/seo/product/<product_number>/`, and not the correct address: `/new-product/cool-url-12341/`.  

Instead of the rewrites, I created a hidden rewrite layer such that, as far as the user or Google is aware, you are taken from `/old-product/?id=12341-12` → `/new-product/cool-url-12341/`. This is how I achieved it.

## Nginx Maps

Put in nginx.conf:
```Nginx
map $arg_id $seo_redirect_arg {
  default "";

  ~^([0-9]+)- 0;
}

map $request_uri $seo_redirect_uri {
  default 0;

  ~^/se/cs-seo/product-redirect/ 1;
  ~^/en/cs-seo/product-redirect/ 1;
  ~^/cs-seo/product-redirect/    1;
}

map $seo_redirect_uri:$seo_redirect_arg $seo_redirect_map {
  default 1;

  ~^0:([0-9]+) 0;
}
```
We use maps as a starter because they are faster than if statements.  

## Named Location

Then we put a named location wherever within the server block or an included file:
```Nginx
# custom error code to handle redirect from where ever needed
error_page 379 = @internal_redirect;

location @internal_redirect {
    internal;
    set $product_id "";
    set $dir "/";

    if ($arg_id ~ "^([0-9]+)-") {
        set $product_id $1;
    }

    if ($product_id = "") {
      return 404;
    }

    if ($request_uri ~ "^/se/") {
      set $dir "/se/";
    }

    if ($request_uri ~ "^/en/") {
      set $dir "/en/";
    }

    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
    proxy_intercept_errors on;
    proxy_pass http://127.0.0.1:8080${dir}cs-seo/product-redirect/$product_id;
    proxy_set_header Host $host;
}
```
In case you are wondering, the variables defined in the maps are not referenceable within a named location.

We create an error page for the return of HTTP 379 (can be whatever—just stay clear of the usual ones). Doing it this way, we are able to call the named location within an if statement, which is normally a hassle. For example, if you want `try_files` within an if, that's not possible.

## Final Integration

Finally, the part that glues it all together:
```Nginx
location / {
   [...]
   if ($seo_redirect_map = 0) {
     # Redirects to internal_redirect
     return 379;
   }
   
   index <index_files>.php;
   try_files $uri $uri/ /index.php$is_args$args;
}
```