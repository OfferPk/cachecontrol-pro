/\*\*  
 \* CacheControl Pro \- 10-Point PII Leakage & Caching Risk Auditor (v1.0.0)  
 \* Evaluates cache visibility, sensitive data leakage, RFC 9111 compliance,  
 \* stale deployment drift, and CDN edge optimization.  
 \*/

(function (root, factory) {  
  if (typeof module \=== 'object' && module.exports) {  
    module.exports \= factory(require('./cache-engine.js'));  
  } else {  
    root.CacheAuditor \= factory(root.CacheEngine);  
  }  
}(typeof self \!== 'undefined' ? self : this, function (CacheEngine) {  
  'use strict';

  function auditPolicy(config) {  
    config \= config || {};  
    var isSensitiveData \= \!\!config.isSensitiveData;  
    var isStaticAsset \= \!\!config.isStaticAsset;  
    var isHtmlDocument \= \!\!config.isHtmlDocument;  
    var varyHeader \= config.varyHeader ? config.varyHeader.trim().toLowerCase() : '';  
    var hasSurrogateControl \= \!\!(config.surrogateControl && config.surrogateControl.trim());

    var posture \= CacheEngine.evaluateCachingPosture(config);  
    var checks \= \[\];  
    var totalScore \= 0;

    // 1\. Authenticated Session & PII Leakage Prevention (Weight: 20\)  
    var c1 \= {  
      id: 'pii-leakage',  
      title: 'Authenticated Session & PII Cache Leakage Defense',  
      weight: 20,  
      score: 0,  
      status: 'FAIL',  
      description: 'Prevents authenticated profile data, billing records, and session tokens from being cached in shared CDN edge nodes.'  
    };  
    if (isSensitiveData) {  
      if (config.noStore) {  
        c1.score \= 20;  
        c1.status \= 'PASS';  
        c1.detail \= 'Strict no-store active. Neither browser disk nor intermediate CDN caches will retain sensitive user payloads.';  
      } else if (config.visibility \=== 'public' || (typeof config.sMaxage \=== 'number' && config.sMaxage \> 0)) {  
        c1.score \= 0;  
        c1.status \= 'FAIL';  
        c1.detail \= 'CRITICAL VULNERABILITY: Sensitive user endpoint is configured as public or has s-maxage. CDNs will cache User A\\'s session and serve it to User B (Steam/ChatGPT-style breach).';  
      } else if (config.noCache) {  
        c1.score \= 6;  
        c1.status \= 'WARN';  
        c1.detail \= 'no-cache permits storage on client disk and intermediate caches. Sensitive PII requires strict no-store.';  
      } else {  
        c1.score \= 0;  
        c1.status \= 'FAIL';  
        c1.detail \= 'Sensitive authenticated endpoint lacks no-store. High risk of cross-user session data leakage in shared caches.';  
      }  
    } else {  
      c1.score \= 20;  
      c1.status \= 'PASS';  
      c1.detail \= 'Public/non-sensitive asset; shared caching does not pose a PII leakage hazard.';  
    }  
    checks.push(c1);

    // 2\. The "no-cache" Misconception Trap (Weight: 10\)  
    var c2 \= {  
      id: 'no-cache-trap',  
      title: 'The "no-cache" vs "no-store" Semantics Alignment',  
      weight: 10,  
      score: 0,  
      status: 'PASS',  
      description: 'Ensures developers do not confuse no-cache (conditional validation) with no-store (zero storage).'  
    };  
    if (isSensitiveData && config.noCache && \!config.noStore) {  
      c2.score \= 2;  
      c2.status \= 'FAIL';  
      c2.detail \= 'MISCONCEPTION: "no-cache" does NOT mean "do not cache". It tells caches to store the response and revalidate. Use "no-store" for sensitive data.';  
    } else if (isHtmlDocument && config.noCache && \!config.noStore) {  
      c2.score \= 10;  
      c2.status \= 'PASS';  
      c2.detail \= 'Optimal HTML strategy: "no-cache" ensures browsers revalidate dynamic HTML against origin ETags on every navigation.';  
    } else {  
      c2.score \= 10;  
      c2.status \= 'PASS';  
      c2.detail \= 'Directives correctly reflect intended retention and validation semantics.';  
    }  
    checks.push(c2);

    // 3\. Shared Cache (CDN) Isolation (Weight: 15\)  
    var c3 \= {  
      id: 'shared-cache-isolation',  
      title: 'Shared Cache & CDN Edge Boundary Scoping',  
      weight: 15,  
      score: 0,  
      status: 'FAIL',  
      description: 'Guarantees shared caches respect user-private vs public caching boundaries.'  
    };  
    if (config.visibility \=== 'private') {  
      if (config.sMaxage && config.sMaxage \> 0\) {  
        c3.score \= 4;  
        c3.status \= 'WARN';  
        c3.detail \= 'Conflict: "private" contradicts "s-maxage". Shared caches should not process s-maxage when private is set.';  
      } else {  
        c3.score \= 15;  
        c3.status \= 'PASS';  
        c3.detail \= 'Correctly scoped to "private". Intermediate CDNs and corporate proxies will never store this response.';  
      }  
    } else if (config.visibility \=== 'public' || (config.sMaxage && config.sMaxage \> 0)) {  
      if (isSensitiveData) {  
        c3.score \= 0;  
        c3.status \= 'FAIL';  
        c3.detail \= 'Shared cache enabled on sensitive data. Must be set to private or no-store.';  
      } else {  
        c3.score \= 15;  
        c3.status \= 'PASS';  
        c3.detail \= 'Public caching enabled for non-sensitive resource; CDNs can cache and absorb origin traffic.';  
      }  
    } else {  
      c3.score \= 8;  
      c3.status \= 'WARN';  
      c3.detail \= 'Cache visibility unspecified. Some CDNs default to caching 200 OK responses with query parameters.';  
    }  
    checks.push(c3);

    // 4\. Static Asset Zero-Roundtrip Optimization (Weight: 10\)  
    var c4 \= {  
      id: 'static-immutable',  
      title: 'Static Asset Invalidation & Immutable Optimization',  
      weight: 10,  
      score: 0,  
      status: 'INFO',  
      description: 'Maximizes browser cache hits on hashed static bundles (JS, CSS, images) using immutable.'  
    };  
    if (isStaticAsset) {  
      if (config.immutable && config.maxAge \>= 31536000\) {  
        c4.score \= 10;  
        c4.status \= 'PASS';  
        c4.detail \= 'Optimal static asset configuration: "max-age=31536000, immutable" eliminates 304 revalidation round-trips.';  
      } else if (config.maxAge \>= 86400\) {  
        c4.score \= 7;  
        c4.status \= 'WARN';  
        c4.detail \= 'Long max-age detected without "immutable". Browser reloads may still trigger conditional 304 validation requests.';  
      } else {  
        c4.score \= 3;  
        c4.status \= 'FAIL';  
        c4.detail \= 'Static asset has low max-age or is uncached, causing high origin egress bandwidth and slow page reloads.';  
      }  
    } else {  
      c4.score \= 10;  
      c4.status \= 'PASS';  
      c4.detail \= 'Non-static resource; immutable directive not applicable.';  
    }  
    checks.push(c4);

    // 5\. Stale-While-Revalidate Resiliency (Weight: 10\)  
    var c5 \= {  
      id: 'stale-while-revalidate',  
      title: 'Stale-While-Revalidate (RFC 5861\) Asynchronous Freshness',  
      weight: 10,  
      score: 0,  
      status: 'INFO',  
      description: 'Delivers instantaneous cached responses while asynchronously refreshing content in the background.'  
    };  
    if (config.staleWhileRevalidate && config.staleWhileRevalidate \> 0\) {  
      if (isSensitiveData) {  
        c5.score \= 0;  
        c5.status \= 'FAIL';  
        c5.detail \= 'stale-while-revalidate must NOT be used on sensitive user-specific data.';  
      } else {  
        c5.score \= 10;  
        c5.status \= 'PASS';  
        c5.detail \= 'stale-while-revalidate active (' \+ config.staleWhileRevalidate \+ 's). Eliminates latency spikes on cache expiration.';  
      }  
    } else if (\!isSensitiveData && \!isStaticAsset && \!isHtmlDocument) {  
      c5.score \= 5;  
      c5.status \= 'INFO';  
      c5.detail \= 'Consider adding stale-while-revalidate to dynamic API responses for sub-10ms edge response times.';  
    } else {  
      c5.score \= 10;  
      c5.status \= 'PASS';  
      c5.detail \= 'stale-while-revalidate configuration matches resource archetype.';  
    }  
    checks.push(c5);

    // 6\. Stale-If-Error Availability Defense (Weight: 5\)  
    var c6 \= {  
      id: 'stale-if-error',  
      title: 'Stale-If-Error (RFC 5861\) Outage Resilience',  
      weight: 5,  
      score: 0,  
      status: 'INFO',  
      description: 'Allows CDNs to serve stale content during origin 5xx server downtime.'  
    };  
    if (config.staleIfError && config.staleIfError \> 0\) {  
      c6.score \= 5;  
      c6.status \= 'PASS';  
      c6.detail \= 'stale-if-error configured (' \+ config.staleIfError \+ 's). CDNs will absorb origin outages gracefully.';  
    } else {  
      c6.score \= 3;  
      c6.status \= 'INFO';  
      c6.detail \= 'stale-if-error omitted. Origin outages will immediately propagate 502/504 errors to end users.';  
    }  
    checks.push(c6);

    // 7\. Vary Header Cache Poisoning Protection (Weight: 10\)  
    var c7 \= {  
      id: 'vary-header',  
      title: 'Vary Header & Cache Partitioning Integrity',  
      weight: 10,  
      score: 0,  
      status: 'FAIL',  
      description: 'Prevents cache poisoning across compression encodings, authentication tokens, and user cookies.'  
    };  
    if (varyHeader.indexOf('accept-encoding') \!== \-1) {  
      c7.score \= 10;  
      c7.status \= 'PASS';  
      c7.detail \= 'Vary: Accept-Encoding is configured, preventing gzip/brotli decompression poisoning across clients.';  
    } else if (config.visibility \=== 'public') {  
      c7.score \= 4;  
      c7.status \= 'WARN';  
      c7.detail \= 'Public cached responses should include "Vary: Accept-Encoding" to prevent uncompressed payloads served to brotli clients.';  
    } else {  
      c7.score \= 8;  
      c7.status \= 'PASS';  
      c7.detail \= 'Vary header configuration acceptable for resource scope.';  
    }  
    checks.push(c7);

    // 8\. HTML Document Deployment Invalidation (Weight: 10\)  
    var c8 \= {  
      id: 'html-invalidation',  
      title: 'HTML Document Invalidation & Deployment Drift Defense',  
      weight: 10,  
      score: 0,  
      status: 'FAIL',  
      description: 'Prevents users from getting stuck on outdated HTML referencing obsolete JavaScript bundle hashes.'  
    };  
    if (isHtmlDocument) {  
      if (config.noStore || (config.noCache && (\!config.maxAge || config.maxAge \=== 0))) {  
        c8.score \= 10;  
        c8.status \= 'PASS';  
        c8.detail \= 'HTML document is configured with no-cache / no-store. Ensures immediate rollout visibility upon new deployments.';  
      } else if (config.maxAge && config.maxAge \> 3600\) {  
        c8.score \= 2;  
        c8.status \= 'FAIL';  
        c8.detail \= 'High max-age on HTML document. Users will retain old HTML pointing to deleted asset hashes, crashing the UI.';  
      } else {  
        c8.score \= 6;  
        c8.status \= 'WARN';  
        c8.detail \= 'HTML document has moderate caching; recommend no-cache with ETag validation.';  
      }  
    } else {  
      c8.score \= 10;  
      c8.status \= 'PASS';  
      c8.detail \= 'Non-HTML asset; deployment drift check passed.';  
    }  
    checks.push(c8);

    // 9\. Downstream Cache Revalidation Enforcement (Weight: 5\)  
    var c9 \= {  
      id: 'must-revalidate',  
      title: 'Downstream Proxy & Browser Revalidation Enforcement',  
      weight: 5,  
      score: 0,  
      status: 'INFO',  
      description: 'Guarantees caches strictly obey expiration without serving stale content.'  
    };  
    if (config.mustRevalidate || config.proxyRevalidate) {  
      c9.score \= 5;  
      c9.status \= 'PASS';  
      c9.detail \= 'must-revalidate or proxy-revalidate enforced. Expired content will never be served without origin verification.';  
    } else {  
      c9.score \= 3;  
      c9.status \= 'INFO';  
      c9.detail \= 'Caches may serve disconnected stale responses under certain network failure conditions.';  
    }  
    checks.push(c9);

    // 10\. Surrogate & CDN-Specific Header Scoping (Weight: 5\)  
    var c10 \= {  
      id: 'surrogate-scoping',  
      title: 'Surrogate-Control & Multi-Tier CDN Differentiation',  
      weight: 5,  
      score: 0,  
      status: 'INFO',  
      description: 'Enables distinct cache TTLs between edge CDN servers and client desktop/mobile browsers.'  
    };  
    if (hasSurrogateControl || (typeof config.sMaxage \=== 'number' && config.sMaxage \!== config.maxAge)) {  
      c10.score \= 5;  
      c10.status \= 'PASS';  
      c10.detail \= 'Multi-tier cache hierarchy active: CDN edge nodes and client browsers operate under decoupled TTL lifecycles.';  
    } else {  
      c10.score \= 3;  
      c10.status \= 'INFO';  
      c10.detail \= 'Single-tier caching: Browser and CDN share identical TTL lifetimes.';  
    }  
    checks.push(c10);

    // Calculate Total Score  
    checks.forEach(function (c) {  
      totalScore \+= c.score;  
    });

    var grade \= 'F';  
    if (totalScore \>= 97\) grade \= 'A+';  
    else if (totalScore \>= 90\) grade \= 'A';  
    else if (totalScore \>= 80\) grade \= 'B';  
    else if (totalScore \>= 70\) grade \= 'C';  
    else if (totalScore \>= 60\) grade \= 'D';

    var criticalCount \= checks.filter(function (c) { return c.status \=== 'FAIL'; }).length;  
    var warningCount \= checks.filter(function (c) { return c.status \=== 'WARN'; }).length;  
    var passCount \= checks.filter(function (c) { return c.status \=== 'PASS'; }).length;

    return {  
      score: totalScore,  
      grade: grade,  
      posture: posture,  
      checks: checks,  
      summary: {  
        total: checks.length,  
        passed: passCount,  
        warnings: warningCount,  
        failures: criticalCount  
      }  
    };  
  }

  return {  
    auditPolicy: auditPolicy  
  };  
}));  
