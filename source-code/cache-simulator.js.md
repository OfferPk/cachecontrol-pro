/\*\*  
 \* CacheControl Pro \- Interactive Cache Poisoning & Stale Asset Simulator (v1.0.0)  
 \* Visualizes shared cache PII data breaches, no-cache disk leaks,  
 \* deployment drift crashes, and zero-trust multi-tier caching architectures.  
 \*/

(function (root, factory) {  
  if (typeof module \=== 'object' && module.exports) {  
    module.exports \= factory();  
  } else {  
    root.CacheSimulator \= factory();  
  }  
}(typeof self \!== 'undefined' ? self : this, function () {  
  'use strict';

  var SCENARIOS \= {  
    pii\_cache\_poisoning: {  
      id: 'pii\_cache\_poisoning',  
      title: 'Authenticated Session PII Leak in Shared CDN Cache',  
      threatType: 'Shared Cache Data Exposure (Steam/ChatGPT Glitch)',  
      endpoint: '/api/v1/user/billing-profile',  
      description: 'Demonstrates how omitting "no-store" or using "public" on authenticated endpoints causes CDNs to cache and serve User A\\'s personal data to User B.',  
      steps: \[  
        {  
          phase: '1. User A Authenticated Request',  
          actor: 'Alice (User A)',  
          action: 'GET /api/v1/user/billing-profile with Authorization: Bearer token\_alice.',  
          detail: 'Alice accesses her dashboard to update her registered credit card and home address.',  
          badge: 'AUTH REQUEST',  
          badgeClass: 'badge-info'  
        },  
        {  
          phase: '2. Misconfigured Origin Response',  
          actor: 'Origin Server',  
          action: 'HTTP 200 OK with Cache-Control: public, max-age=300 (or omitted without no-store).',  
          detail: 'Developer intended 5-minute browser caching, but mistakenly declared the response "public" or allowed default CDN caching.',  
          badge: 'MISCONFIGURATION',  
          badgeClass: 'badge-danger'  
        },  
        {  
          phase: '3. CDN Edge Node Cache Poisoning',  
          actor: 'CDN Edge PoP (Frankfurt)',  
          action: 'CDN caches Alice\\'s full JSON payload under cache key "GET /api/v1/user/billing-profile".',  
          detail: 'Intermediate shared cache stores sensitive PII: name, billing address, last 4 digits of credit card.',  
          badge: 'SHARED CACHE POISONED',  
          badgeClass: 'badge-danger'  
        },  
        {  
          phase: '4. User B Data Breach (Cross-Account Leak)',  
          actor: 'Bob (User B)',  
          action: 'Bob logs in and visits his billing page: GET /api/v1/user/billing-profile.',  
          detail: 'CDN intercepts request, matches cache key, and returns HTTP 200 (CF-Cache-Status: HIT) containing ALICE\\'s confidential billing profile\!',  
          badge: 'CRITICAL DATA LEAK',  
          badgeClass: 'badge-danger'  
        }  
      \]  
    },

    no\_cache\_misconception: {  
      id: 'no\_cache\_misconception',  
      title: 'The "no-cache" Misconception & Local Disk Forensic Exposure',  
      threatType: 'Client-Side Persistent Storage Vulnerability',  
      endpoint: '/patient/records/diagnostic-summary',  
      description: 'Demonstrates why "no-cache" does NOT prevent sensitive records from being written to unencrypted local disk caches.',  
      steps: \[  
        {  
          phase: '1. Patient Medical Session',  
          actor: 'Patient (Shared Workstation)',  
          action: 'Patient views confidential lab results at a hospital kiosk or public library computer.',  
          detail: 'Web server returns response with header: Cache-Control: no-cache.',  
          badge: 'INTENDED NO-CACHE',  
          badgeClass: 'badge-info'  
        },  
        {  
          phase: '2. Browser Disk Storage Write',  
          actor: 'Browser Cache Engine',  
          action: 'Browser parses "no-cache". RFC 9111 permits writing the response to local disk cache.',  
          detail: '"no-cache" only mandates revalidation before serving; it does NOT prohibit disk persistence.',  
          badge: 'WRITTEN TO DISK',  
          badgeClass: 'badge-warning'  
        },  
        {  
          phase: '3. Patient Logout',  
          actor: 'User Session Manager',  
          action: 'Patient clicks "Log Out" and leaves the public computer.',  
          detail: 'Session cookie is cleared, but persistent browser disk cache files remain intact in the local filesystem.',  
          badge: 'SESSION TERMINATED',  
          badgeClass: 'badge-info'  
        },  
        {  
          phase: '4. Forensic Cache Extraction',  
          actor: 'Next Computer User',  
          action: 'Subsequent user opens DevTools or examines OS cache folder: %LOCALAPPDATA%\\\\Google\\\\Chrome\\\\User Data\\\\Default\\\\Cache.',  
          detail: 'Patient diagnostic lab report, full name, and medical diagnosis extracted from unencrypted disk cache\! Fix: Use Cache-Control: no-store.',  
          badge: 'FORENSIC LEAK',  
          badgeClass: 'badge-danger'  
        }  
      \]  
    },

    stale\_deployment\_drift: {  
      id: 'stale\_deployment\_drift',  
      title: 'Stale HTML Caching & Webpack/Vite ChunkLoadError Crash',  
      threatType: 'Deployment Desynchronization & Outage',  
      endpoint: '/index.html',  
      description: 'Demonstrates how caching dynamic HTML causes clients to request deleted JavaScript chunks, resulting in application crashes.',  
      steps: \[  
        {  
          phase: '1. Over-Aggressive HTML Caching',  
          actor: 'Web Server / S3 Bucket',  
          action: 'Origin serves index.html with Cache-Control: max-age=86400 (24 hours).',  
          detail: 'HTML references JavaScript entry point: \<script src="/assets/main-a1b2c3.js"\>.',  
          badge: 'AGGRESSIVE HTML CACHE',  
          badgeClass: 'badge-warning'  
        },  
        {  
          phase: '2. Production Deployment & Build Invalidation',  
          actor: 'CI/CD Pipeline',  
          action: 'Engineering deploys Version 2.0.0. Vite/Webpack generates new bundle: main-d4e5f6.js.',  
          detail: 'Old bundle main-a1b2c3.js is purged from the origin server.',  
          badge: 'RELEASE DEPLOYED',  
          badgeClass: 'badge-info'  
        },  
        {  
          phase: '3. Returning User Navigation',  
          actor: 'Returning Customer',  
          action: 'User visits website. Browser serves index.html directly from memory/disk cache without checking origin.',  
          detail: 'Cached HTML still requests the deleted bundle: /assets/main-a1b2c3.js.',  
          badge: 'STALE HTML SERVED',  
          badgeClass: 'badge-danger'  
        },  
        {  
          phase: '4. Frontend Application Crash',  
          actor: 'Browser Runtime',  
          action: 'HTTP 404 Not Found on main-a1b2c3.js. App crashes with Uncaught ChunkLoadError.',  
          detail: 'Users see a blank white screen. Fix: Always serve HTML with Cache-Control: no-cache, and hashed assets with immutable.',  
          badge: 'APP CRASHED',  
          badgeClass: 'badge-danger'  
        }  
      \]  
    },

    cachecontrol\_hardened\_tier: {  
      id: 'cachecontrol\_hardened\_tier',  
      title: 'CacheControl Pro Multi-Tier Zero-Trust Architecture',  
      threatType: 'Fortified Multi-Tier Caching Strategy',  
      endpoint: 'Full Stack Orchestration (APIs, HTML & Static Assets)',  
      description: 'Demonstrates mathematically optimal, leak-proof caching across authenticated APIs, dynamic HTML, and immutable assets.',  
      steps: \[  
        {  
          phase: '1. Sensitive Authenticated Endpoints',  
          actor: 'API Gateway',  
          action: 'Cache-Control: private, no-cache, no-store, must-revalidate & Vary: Authorization, Cookie.',  
          detail: 'Zero bytes stored in browser memory, disk, or CDN edge nodes. Complete immunity to cross-user PII leakage.',  
          badge: 'ZERO RETENTION',  
          badgeClass: 'badge-success'  
        },  
        {  
          phase: '2. Dynamic HTML Documents',  
          actor: 'Edge CDN / Nginx',  
          action: 'Cache-Control: no-cache & ETag: "v2.0.0-hash".',  
          detail: 'Browser checks with origin in 15ms. If unchanged, returns 304; if new deployment exists, immediately downloads new HTML.',  
          badge: 'INSTANT DEPLOYMENT',  
          badgeClass: 'badge-success'  
        },  
        {  
          phase: '3. Content-Hashed Static Assets',  
          actor: 'Asset CDN Storage',  
          action: 'Cache-Control: public, max-age=31536000, immutable.',  
          detail: 'Zero round-trips for 1 full year. Browser never sends 304 validation requests, yielding 0ms asset loads and 90% bandwidth savings.',  
          badge: 'IMMUTABLE 0MS LOAD',  
          badgeClass: 'badge-success'  
        },  
        {  
          phase: '4. Public Dynamic Content APIs',  
          actor: 'Cloudflare / Fastly Edge',  
          action: 'Cache-Control: public, max-age=60, s-maxage=300, stale-while-revalidate=600, stale-if-error=86400.',  
          detail: 'Instant edge responses under 10ms. Asynchronous background revalidation. 100% uptime even during origin database outages.',  
          badge: 'MAX RESILIENCE',  
          badgeClass: 'badge-success'  
        }  
      \]  
    }  
  };

  return {  
    SCENARIOS: SCENARIOS  
  };  
}));  
