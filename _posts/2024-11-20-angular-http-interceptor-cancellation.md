---
layout: post
title: Angular http interceptor - cancellation [TS]
date: 2024-11-20
tags:
  - ts
series: angular-interceptor
---

{%- include /_posts/series-toc.md series-name="angular-interceptor" -%}
## RaceInterceptor

Last time, we implemented `CacheInterceptor`, which is responsible for caching data returned from the server. Angular interceptors give us an infrastructure for implementing "cross-cutting concerns." Every web call can potentially use caching functionality, and we don't want to spread code responsible for that in many places. The interceptor keeps the code in one place, which can be shared between many SPA applications. 

Handling race conditions is another functionality that is potentially duplicated in many places in our application. Let's say we must build UI for the list of products with additional filters above. The user can change the filter settings and press the "Search" button many times. The HTTP responses will come back asynchronously, not necessarily in the same order they were initially sent. A typical race condition occurs when users search `productType=TV`, but the UI presents iPads. The problem is that the response for searching iPads came last, overriding all previous responses. At least in an Angular application, where Rx is used, the solution to this problem requires using appropriate Rx operators like `switchMap` or others, depending on the specific scenarios.

Our interceptor's usage is simple. We can optionally pass an `HttpContext` object to each web call. This object is a dictionary with key-value pairs. Setting the `CANCELLATION_KEY_HTTPCONTEXT_TOKEN` key activates the interceptor logic. The value is a string representing the request's unique ID. The following request with the same ID cancels the previous request that is still running. If you check the Network tab in Chrome Developer Tools, you will see that the request is physically aborted. That's it. The mechanism is simple but still powerful. It does not matter what the URL is, as long as the ID is the same, the running request with the same ID is canceled. ID is just a string and can always be generated dynamically. 

```typescript
const context = new HttpContext().set(CANCELLATION_KEY_HTTPCONTEXT_TOKEN, "products-list"));
const obs = httpClient.get<Product>("/api/products", {context})`;
obs.subscribe(data => { });
```

From the Rx perspective, the cancellation means that the `OnNext` is not emitted on the `obs`  object, only `OnCompleted`. Sometimes it's worth wrapping an observable object like `lastValueFrom(obs, { defaultValue: ... })`to provide a default value in case of cancellation. It is especially important when an Observable object is converted into a Promise. 

Now, let's take a look at the implementation of the interceptor.

```typescript
import { Injectable } from '@angular/core';
import { HttpEvent, HttpInterceptor, HttpHandler, HttpRequest, HttpContextToken } from '@angular/common/http';
import { Observable, Subject } from 'rxjs';
import { filter, takeUntil } from 'rxjs/operators';

export const CANCELLATION_KEY_HTTPCONTEXT_TOKEN = new HttpContextToken<string | undefined>(() => undefined);

@Injectable()
export class RaceInterceptor implements HttpInterceptor {

  private cancellationKyes$ = new Subject<string>();

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const cancellationContext = req.context.get(CANCELLATION_KEY_HTTPCONTEXT_TOKEN); // returns default value if not exists

    if (typeof cancellationContext !== "undefined") {
      const cancellationKey = cancellationContext;

      this.cancellationKyes$.next(cancellationKey);
      return next.handle(req).pipe(takeUntil(this.cancellationKyes$.pipe(filter(ck => ck === cancellationKey))));
    }

    return next.handle(req);
  }
}
```
