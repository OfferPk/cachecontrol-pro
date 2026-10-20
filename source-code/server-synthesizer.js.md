/\*\*  
 \* CacheControl Pro \- Server & CDN Synthesizer (v1.0.0)  
 \* Generates production-ready configurations for Nginx, Apache,  
 \* Cloudflare, AWS CloudFront, Caddy, and Next.js/Vercel.  
 \*/

(function (root, factory) {  
  if (typeof module \=== 'object' && module.exports) {  
    module.exports \= factory(require('./cache-engine.js'));  
  } else {  
    root.ServerSynthesizer \= factory(root.CacheEngine);  
  }  
}(typeof self \!== 'undefined' ? self : this, function (CacheEngine) {  
  'use strict';

  function buildHeaderPayload(config) {  
    var ccValue \= CacheEngine.buildCacheControlHeader(config);  
    var headers \= \[{ name: 'Cache-Control', value: ccValue }\];

    if (config.varyHeader && config.varyHeader.trim()) {  
      headers.push({ name: 'Vary', value: config.varyHeader.trim() });  
    }

    if (config.surrogateControl && config.surrogateControl.trim()) {  
      headers.push({ name: 'Surrogate-Control', value: config.surrogateControl.trim() });  
    }

    if (config.cdnCacheControl && config.cdnCacheControl.trim()) {  
      headers.push({ name: 'CDN-Cache-Control', value: config.cdnCacheControl.trim() });  
    }

    return headers;  
  }

  function toNginx(config) {  
    var headers \= buildHeaderPayload(config);  
    var lines \= \['\# \================================================================'\];  
    lines.push('\# CacheControl Pro \- Nginx HTTP Caching Configuration');  
    lines.push('\# RFC 9111 compliant header directives');  
    lines.push('\# \================================================================');  
    lines.push('');  
    headers.forEach(function (h) {  
      lines.push('add\_header ' \+ h.name \+ ' "' \+ h.value \+ '" always;');  
    });  
    return lines.join('\\n');  
  }

  function toApache(config) {  
    var headers \= buildHeaderPayload(config);  
    var lines \= \['\# \================================================================'\];  
    lines.push('\# CacheControl Pro \- Apache 2.4+ (.htaccess / httpd.conf)');  
    lines.push('\# Requires: mod\_headers (LoadModule headers\_module modules/mod\_headers.so)');  
    lines.push('\# \================================================================');  
    lines.push('\<IfModule mod\_headers.c\>');  
    headers.forEach(function (h) {  
      lines.push('    Header always set ' \+ h.name \+ ' "' \+ h.value \+ '"');  
    });  
    lines.push('\</IfModule\>');  
    return lines.join('\\n');  
  }

  function toCloudflarePages(config) {  
    var headers \= buildHeaderPayload(config);  
    var lines \= \['\# \================================================================'\];  
    lines.push('\# CacheControl Pro \- Cloudflare Pages (\_headers file)');  
    lines.push('\# Place in the root of your published output directory');  
    lines.push('\# \================================================================');  
    lines.push('/\*');  
    headers.forEach(function (h) {  
      lines.push('  ' \+ h.name \+ ': ' \+ h.value);  
    });  
    return lines.join('\\n');  
  }

  function toCloudflareWorker(config) {  
    var headers \= buildHeaderPayload(config);  
    var lines \= \['/\*\*'\];  
    lines.push(' \* CacheControl Pro \- Cloudflare Worker / Edge Middleware');  
    lines.push(' \*/');  
    lines.push('export default {');  
    lines.push('  async fetch(request, env, ctx) {');  
    lines.push('    const response \= await fetch(request);');  
    lines.push('    const newHeaders \= new Headers(response.headers);');  
    lines.push('');  
    headers.forEach(function (h) {  
      lines.push('    newHeaders.set("' \+ h.name \+ '", "' \+ h.value \+ '");');  
    });  
    lines.push('');  
    lines.push('    return new Response(response.body, {');  
    lines.push('      status: response.status,');  
    lines.push('      statusText: response.statusText,');  
    lines.push('      headers: newHeaders');  
    lines.push('    });');  
    lines.push('  }');  
    lines.push('};');  
    return lines.join('\\n');  
  }

  function toCloudFront(config) {  
    var ccVal \= CacheEngine.buildCacheControlHeader(config);  
    var policy \= {  
      ResponseHeadersPolicyConfig: {  
        Comment: "CacheControl Pro Generated Security & Cache Policy",  
        Name: "CacheControlProCustomPolicy",  
        CustomHeadersConfig: {  
          Quantity: 1,  
          Items: \[  
            {  
              Header: "Cache-Control",  
              Value: ccVal,  
              Override: true  
            }  
          \]  
        }  
      }  
    };  
    if (config.varyHeader) {  
      policy.ResponseHeadersPolicyConfig.CustomHeadersConfig.Quantity++;  
      policy.ResponseHeadersPolicyConfig.CustomHeadersConfig.Items.push({  
        Header: "Vary",  
        Value: config.varyHeader,  
        Override: true  
      });  
    }  
    return JSON.stringify(policy, null, 2);  
  }

  function toCaddy(config) {  
    var headers \= buildHeaderPayload(config);  
    var lines \= \['\# \================================================================'\];  
    lines.push('\# CacheControl Pro \- Caddy v2 Caddyfile');  
    lines.push('\# \================================================================');  
    lines.push('example.com {');  
    lines.push('    header {');  
    headers.forEach(function (h) {  
      lines.push('        ' \+ h.name \+ ' "' \+ h.value \+ '"');  
    });  
    lines.push('    }');  
    lines.push('    reverse\_proxy localhost:3000');  
    lines.push('}');  
    return lines.join('\\n');  
  }

  function toNextJs(config) {  
    var headers \= buildHeaderPayload(config);  
    var lines \= \['// \================================================================'\];  
    lines.push('// CacheControl Pro \- Next.js next.config.js');  
    lines.push('// \================================================================');  
    lines.push('module.exports \= {');  
    lines.push('  async headers() {');  
    lines.push('    return \[');  
    lines.push('      {');  
    lines.push('        source: "/:path\*",');  
    lines.push('        headers: \[');  
    headers.forEach(function (h, idx) {  
      var comma \= (idx \< headers.length \- 1\) ? ',' : '';  
      lines.push('          { key: "' \+ h.name \+ '", value: "' \+ h.value \+ '" }' \+ comma);  
    });  
    lines.push('        \]');  
    lines.push('      }');  
    lines.push('    \];');  
    lines.push('  }');  
    lines.push('};');  
    return lines.join('\\n');  
  }

  return {  
    buildHeaderPayload: buildHeaderPayload,  
    toNginx: toNginx,  
    toApache: toApache,  
    toCloudflarePages: toCloudflarePages,  
    toCloudflareWorker: toCloudflareWorker,  
    toCloudFront: toCloudFront,  
    toCaddy: toCaddy,  
    toNextJs: toNextJs  
  };  
}));  
