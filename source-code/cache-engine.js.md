/\*\*  
 \* CacheControl Pro \- RFC 9111 HTTP Caching Engine (v1.0.0)  
 \* Evaluates RFC 9111 and RFC 5861 directives, computes browser & CDN TTLs,  
 \* validates cache visibility, and synthesizes compliant headers.  
 \*/

(function (root, factory) {  
  if (typeof module \=== 'object' && module.exports) {  
    module.exports \= factory();  
  } else {  
    root.CacheEngine \= factory();  
  }  
}(typeof self \!== 'undefined' ? self : this, function () {  
  'use strict';

  var DIRECTIVES\_META \= {  
    // Visibility  
    'private': {  
      label: 'private',  
      category: 'visibility',  
      description: 'Response is intended for a single user; shared caches (CDNs, proxies) must NOT store it.'  
    },  
    'public': {  
      label: 'public',  
      category: 'visibility',  
      description: 'Response may be stored by any cache (browsers, CDNs, proxies), even if authenticated.'  
    },  
    // Storage & Revalidation  
    'no-store': {  
      label: 'no-store',  
      category: 'storage',  
      description: 'Strict zero-retention. Caches must not store any part of the request or response in non-volatile or shared memory.'  
    },  
    'no-cache': {  
      label: 'no-cache',  
      category: 'revalidation',  
      description: 'Caches may store the response, but MUST revalidate with origin (ETag/304) before serving.'  
    },  
    'must-revalidate': {  
      label: 'must-revalidate',  
      category: 'revalidation',  
      description: 'Once stale, caches cannot serve the response without successful revalidation, even if disconnected.'  
    },  
    'proxy-revalidate': {  
      label: 'proxy-revalidate',  
      category: 'revalidation',  
      description: 'Like must-revalidate, but applies strictly to shared/CDN caches, exempting private browser caches.'  
    },  
    'immutable': {  
      label: 'immutable',  
      category: 'optimization',  
      description: 'Response body will never change during max-age. Browser will not send conditional 304 requests on refresh.'  
    },  
    'no-transform': {  
      label: 'no-transform',  
      category: 'security',  
      description: 'Intermediary proxies must not transform response body or headers (e.g. image transcoding/minification).'  
    }  
  };

  function buildCacheControlHeader(config) {  
    config \= config || {};  
    var parts \= \[\];

    // If no-store is set, standard security practice mandates private, no-cache, no-store, must-revalidate  
    if (config.noStore) {  
      parts.push('no-store');  
      if (config.noCache) parts.push('no-cache');  
      if (config.mustRevalidate) parts.push('must-revalidate');  
      if (config.visibility \=== 'private') parts.push('private');  
      return parts.join(', ');  
    }

    // Visibility  
    if (config.visibility \=== 'private') {  
      parts.push('private');  
    } else if (config.visibility \=== 'public') {  
      parts.push('public');  
    }

    // Revalidation directives  
    if (config.noCache) {  
      parts.push('no-cache');  
    }

    // Browser max-age  
    if (typeof config.maxAge \=== 'number' && config.maxAge \>= 0\) {  
      parts.push('max-age=' \+ config.maxAge);  
    }

    // Shared Cache s-maxage  
    if (typeof config.sMaxage \=== 'number' && config.sMaxage \>= 0 && config.visibility \!== 'private') {  
      parts.push('s-maxage=' \+ config.sMaxage);  
    }

    // Stale-While-Revalidate (RFC 5861\)  
    if (typeof config.staleWhileRevalidate \=== 'number' && config.staleWhileRevalidate \> 0\) {  
      parts.push('stale-while-revalidate=' \+ config.staleWhileRevalidate);  
    }

    // Stale-If-Error (RFC 5861\)  
    if (typeof config.staleIfError \=== 'number' && config.staleIfError \> 0\) {  
      parts.push('stale-if-error=' \+ config.staleIfError);  
    }

    // Revalidation enforcements  
    if (config.mustRevalidate) {  
      parts.push('must-revalidate');  
    }  
    if (config.proxyRevalidate && config.visibility \!== 'private') {  
      parts.push('proxy-revalidate');  
    }  
    if (config.immutable) {  
      parts.push('immutable');  
    }  
    if (config.noTransform) {  
      parts.push('no-transform');  
    }

    return parts.length ? parts.join(', ') : 'no-cache';  
  }

  function parseCacheControlHeader(headerStr) {  
    if (\!headerStr || typeof headerStr \!== 'string') return {};  
    var clean \= headerStr.trim().toLowerCase();  
    var tokens \= clean.split(',').map(function (t) { return t.trim(); });

    var result \= {  
      visibility: 'unspecified',  
      noStore: false,  
      noCache: false,  
      mustRevalidate: false,  
      proxyRevalidate: false,  
      immutable: false,  
      noTransform: false,  
      maxAge: null,  
      sMaxage: null,  
      staleWhileRevalidate: null,  
      staleIfError: null  
    };

    tokens.forEach(function (token) {  
      if (token \=== 'private') result.visibility \= 'private';  
      else if (token \=== 'public') result.visibility \= 'public';  
      else if (token \=== 'no-store') result.noStore \= true;  
      else if (token \=== 'no-cache') result.noCache \= true;  
      else if (token \=== 'must-revalidate') result.mustRevalidate \= true;  
      else if (token \=== 'proxy-revalidate') result.proxyRevalidate \= true;  
      else if (token \=== 'immutable') result.immutable \= true;  
      else if (token \=== 'no-transform') result.noTransform \= true;  
      else if (token.indexOf('max-age=') \=== 0\) {  
        result.maxAge \= parseInt(token.replace('max-age=', ''), 10\) || 0;  
      } else if (token.indexOf('s-maxage=') \=== 0\) {  
        result.sMaxage \= parseInt(token.replace('s-maxage=', ''), 10\) || 0;  
      } else if (token.indexOf('stale-while-revalidate=') \=== 0\) {  
        result.staleWhileRevalidate \= parseInt(token.replace('stale-while-revalidate=', ''), 10\) || 0;  
      } else if (token.indexOf('stale-if-error=') \=== 0\) {  
        result.staleIfError \= parseInt(token.replace('stale-if-error=', ''), 10\) || 0;  
      }  
    });

    return result;  
  }

  function formatDuration(seconds) {  
    if (typeof seconds \!== 'number' || seconds \<= 0\) return '0 seconds';  
    if (seconds \=== 31536000\) return '1 Year (365 days)';  
    if (seconds \>= 86400\) {  
      var days \= Math.floor(seconds / 86400);  
      return days \+ (days \=== 1 ? ' day' : ' days');  
    }  
    if (seconds \>= 3600\) {  
      var hours \= Math.floor(seconds / 3600);  
      return hours \+ (hours \=== 1 ? ' hour' : ' hours');  
    }  
    if (seconds \>= 60\) {  
      var mins \= Math.floor(seconds / 60);  
      return mins \+ (mins \=== 1 ? ' minute' : ' minutes');  
    }  
    return seconds \+ ' seconds';  
  }

  function evaluateCachingPosture(config) {  
    config \= config || {};  
    var header \= buildCacheControlHeader(config);

    var posture \= {  
      header: header,  
      isNoStore: \!\!config.noStore,  
      isPrivate: config.visibility \=== 'private',  
      isPublic: config.visibility \=== 'public',  
      isSharedCacheable: config.visibility \=== 'public' || (typeof config.sMaxage \=== 'number' && config.sMaxage \> 0),  
      isImmutable: \!\!config.immutable,  
      browserTTL: (typeof config.maxAge \=== 'number') ? config.maxAge : 0,  
      cdnTTL: (typeof config.sMaxage \=== 'number') ? config.sMaxage : ((typeof config.maxAge \=== 'number') ? config.maxAge : 0),  
      classification: 'DYNAMIC\_CUSTOM'  
    };

    if (config.noStore) {  
      posture.classification \= 'UNCACHED\_SENSITIVE';  
    } else if (config.immutable && config.maxAge \>= 86400\) {  
      posture.classification \= 'IMMUTABLE\_STATIC';  
    } else if (config.noCache && (\!config.maxAge || config.maxAge \=== 0)) {  
      posture.classification \= 'REVALIDATED\_DYNAMIC';  
    } else if (posture.isSharedCacheable && posture.cdnTTL \> 0\) {  
      posture.classification \= 'SHARED\_CDN\_CACHED';  
    } else if (posture.isPrivate && posture.browserTTL \> 0\) {  
      posture.classification \= 'PRIVATE\_CLIENT\_CACHED';  
    }

    return posture;  
  }

  return {  
    DIRECTIVES\_META: DIRECTIVES\_META,  
    buildCacheControlHeader: buildCacheControlHeader,  
    parseCacheControlHeader: parseCacheControlHeader,  
    formatDuration: formatDuration,  
    evaluateCachingPosture: evaluateCachingPosture  
  };  
}));  
