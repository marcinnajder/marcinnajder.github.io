---
layout: post
title: Angular http interceptor - caching [TS]
date: 2024-09-27
tags:
  - ts
  - rx
series: angular-interceptor
---
{%- include /_posts/series-toc.md series-name="angular-interceptor" -%}
## CacheInterceptor

We have an Angular application and would like to add simple caching functionality. Our application makes many different web calls; sometimes, the same calls are made from various application regions. From the API perspective, the solution should be transparent and straightforward. We don't want to write redundant and complicated code whenever cache functionality is needed. The built-in HTTP module provides an extension point called interceptors, allowing us to inject additional code before and after a web call. We are going to use it to implement caching functionality. 

In Angular, the HTTP `HttpClient` is used to execute web requests. The typical code calling the REST API looks like this: `httpClient.get<SomeData>("/api/some-resource")` returns `Observable<SomeData>`.  Subscription for an observable object causes an HTTP request. Let's assume that the user would like to cache the response from that endpoint forever. It does not matter how many times and from how many places such a call is made, as long as the HTTP URL and Method (here `GET`) are the same, the cached value will be returned. Only the first call hits the server once; the others get the result from the memory cache.

Built-in `HttpContext` type represents optional parameters that can be passed to each web request. Consumer code will configure cache settings via our `CacheContext` type defined like this:

```typescript
export type CacheModes = "none" | "use" | "reload" | "clear";
export type CacheContext = CacheModes | { mode: CacheModes; } | { mode: "set-expiration"; expiration: number; };
 
export const CACHE_HTTPCONTEXT_TOKEN = new HttpContextToken<CacheContext>(() => "none");
```

The simplest usage of a cache storing the data forever will look like this:

```typescript
const context = new HttpContext().set(CACHE_HTTPCONTEXT_TOKEN, "use"));
const obs = httpClient.get<SomeData>("/api/some-resource", {context})`;
obs.subscribe(data => { });
```

One crucial aspect of interceptors and the HTTP module is that the whole mechanism is based on Rx. Observable objects are lazy, so calling the `get` method does nothing; we must subscribe to them.

We can clear the cache for a specific endpoint by replacing the option `"use"` with `"clear"` in the code above. Once again, we must subscribe to cause any effect! Clearing the cache sends a null value to the lambda passed to the `subscribe` method. The other option is to avoid passing the lambda completely, like `obs.subscribe()`. After the cache is cleared, the next call will hit the server again. Remember that the cache mechanism is used only when `CACHE_HTTPCONTEXT_TOKEN` is set; the interceptor will ignore other calls without a token entirely. 

The next option, `"reload"`, works like `"clear"` and `"use"` combined. It hits the server, refreshing the cache.

The last option, `"set-expiration,"` allows us to set an expiration time. Instead of holding the cache's state forever, the next call after expiration will hit the server again, refreshing the cache with the newest data. The assumption is that the `"set-expiration"` mode is used once at the application startup to configure the expiration time for the specified URL. Then, if we want to make a call using expiration time, the `"use"` mode is used. An example configuration could look like this: 

```typescript
// configuration once at startup
const context = new HttpContext().set(CACHE_HTTPCONTEXT_TOKEN, { mode: "set-expiration", expiration: 60 * 1000 }); // 60 seconds
httpClient.get<SomeData>("/api/some-resource", {context})`.subscribe();
```

Now, let's take a look at the implementation of the interceptor.

```typescript
import { Injectable } from '@angular/core';
import { HttpEvent, HttpInterceptor, HttpHandler, HttpRequest, HttpResponse, HttpErrorResponse, HttpContextToken } from '@angular/common/http';
import { Observable, of, } from 'rxjs';
import { map, shareReplay, } from 'rxjs/operators';

export type CacheModes = "none" | "use" | "reload" | "clear";
export type CacheContext = CacheModes | { mode: CacheModes; } | { mode: "set-expiration"; expiration: number; };

export const CACHE_HTTPCONTEXT_TOKEN = new HttpContextToken<CacheContext>(() => "none"); // default mode

const emptyResponse = of(new HttpResponse<any>({ body: undefined }));

interface CacheEntry {
  readonly expirationTime: number; // ms
  readonly value$: Observable<HttpEvent<any>>;
}

@Injectable()
export class CacheInterceptor implements HttpInterceptor {
  private entries: Dictionary<CacheEntry> = {};
  private expirations: Dictionary<number> = {};

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const cacheKey = req.urlWithParams;
    const cacheContext = req.context.get(CACHE_HTTPCONTEXT_TOKEN); // returns default value if not exists
    const cacheContextObj: Exclude<CacheContext, CacheModes> = typeof cacheContext === "string" ? { mode: cacheContext } : cacheContext;

    if (cacheContextObj.mode !== "none" && req.method !== "GET") {
      throw new HttpErrorResponse({ status: 405, statusText: "Proxy cache is only supported for GET requests" }); //405 Method Not Allowed
    }

    if (cacheContextObj.mode === "clear") {
      this.deleteEntry(cacheKey);
      return CacheInterceptor.clone(emptyResponse);
    }

    if (cacheContextObj.mode === "set-expiration") {
      this.expirations[cacheKey] = cacheContextObj.expiration;
      return CacheInterceptor.clone(emptyResponse);
    }

    if (cacheContextObj.mode === 'use' || cacheContextObj.mode === 'reload') {
      const entry = this.getEntry(cacheKey);

      if (entry) {
        if (cacheContextObj.mode === 'reload' || (Date.now() > entry.expirationTime)) {
          this.deleteEntry(cacheKey);
        } else {
          return CacheInterceptor.clone(entry.value$);
        }
      }

      // thanks to 'refCount: true', http request is aborted when all unsubscribed
      const value$ = next.handle(req).pipe(shareReplay({ refCount: true }));

      this.setEntry(cacheKey, {
        value$,
        expirationTime: Date.now() + (this.expirations[cacheKey] ?? Infinity),
      });

      return value$;
    }

    return next.handle(req);
  }

  private static clone(obs: Observable<HttpEvent<any>>): Observable<HttpEvent<any>> {
    return obs.pipe(map(event => event instanceof HttpResponse ? event.clone() : event));
  }
  private deleteEntry(key: string) {
    const keysToDelete = Object.keys(this.entries)
	    .filter(k => k === key || k.startsWith(key + '?')); // includes querystrings
    keysToDelete.forEach(key => delete this.entries[key]);
  }
  private getEntry(key: string) {
    return this.entries[key];
  }
  private setEntry(key: string, entry: CacheEntry) {
    this.entries[key] = entry;
  }
}

```

What can I say about the implementation? It's Rx, so no one really knows what is happening :) The implementation mixes imperative with declarative style. I guess it would be possible to realize the same functionality using a single big Rx query. It's not perfect for sure, but at least it works handling many corner cases like:
- Many calls with the same URL can be made simultaneously, and only one HTTP request will be sent to the server.
- We can subscribe many times to the same instance of an observable object returned from the call or create new instances each time; the behavior should be the same.
- Requests are cancelled once all observers are unsubscribed.
- Errors returned from the server are not cached.