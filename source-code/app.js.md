/\*\*  
 \* CacheControl Pro \- Studio UI Controller & State Manager (v1.0.0)  
 \*/

(function () {  
  'use strict';

  var PRESETS \= {  
    'authenticated-fintech-no-store': {  
      name: 'Authenticated FinTech & User Account API (Strict Zero-Retention)',  
      visibility: 'private',  
      noStore: true,  
      noCache: true,  
      mustRevalidate: true,  
      proxyRevalidate: false,  
      immutable: false,  
      noTransform: true,  
      maxAge: 0,  
      sMaxage: 0,  
      staleWhileRevalidate: 0,  
      staleIfError: 0,  
      isSensitiveData: true,  
      isStaticAsset: false,  
      isHtmlDocument: false,  
      varyHeader: 'Authorization, Cookie, Accept-Encoding',  
      surrogateControl: 'no-store',  
      cdnCacheControl: 'no-store'  
    },  
    'static-assets-immutable-cdn': {  
      name: 'High-Performance Content-Hashed Static Assets (Immutable CDN)',  
      visibility: 'public',  
      noStore: false,  
      noCache: false,  
      mustRevalidate: false,  
      proxyRevalidate: false,  
      immutable: true,  
      noTransform: false,  
      maxAge: 31536000,  
      sMaxage: 31536000,  
      staleWhileRevalidate: 0,  
      staleIfError: 0,  
      isSensitiveData: false,  
      isStaticAsset: true,  
      isHtmlDocument: false,  
      varyHeader: 'Accept-Encoding',  
      surrogateControl: 'max-age=31536000',  
      cdnCacheControl: 'max-age=31536000'  
    },  
    'leaking-public-api-vulnerable': {  
      name: 'Vulnerable Authenticated API (Catastrophic PII Shared Cache Leak)',  
      visibility: 'public',  
      noStore: false,  
      noCache: false,  
      mustRevalidate: false,  
      proxyRevalidate: false,  
      immutable: false,  
      noTransform: false,  
      maxAge: 3600,  
      sMaxage: 7200,  
      staleWhileRevalidate: 0,  
      staleIfError: 0,  
      isSensitiveData: true,  
      isStaticAsset: false,  
      isHtmlDocument: false,  
      varyHeader: '',  
      surrogateControl: '',  
      cdnCacheControl: ''  
    }  
  };

  var state \= {  
    visibility: 'private',  
    noStore: true,  
    noCache: true,  
    mustRevalidate: true,  
    proxyRevalidate: false,  
    immutable: false,  
    noTransform: true,  
    maxAge: 0,  
    sMaxage: 0,  
    staleWhileRevalidate: 0,  
    staleIfError: 0,  
    isSensitiveData: true,  
    isStaticAsset: false,  
    isHtmlDocument: false,  
    varyHeader: 'Authorization, Cookie, Accept-Encoding',  
    surrogateControl: 'no-store',  
    cdnCacheControl: 'no-store',  
    activeTab: 'nginx',  
    activeScenario: 'pii\_cache\_poisoning'  
  };

  // DOM Elements  
  var postureBadge \= document.getElementById('posture-badge');  
  var postureBadgeText \= document.getElementById('posture-badge-text');  
  var presetSelect \= document.getElementById('preset-select');

  var noStoreToggle \= document.getElementById('no-store-toggle');  
  var noCacheToggle \= document.getElementById('no-cache-toggle');  
  var mustRevalidateToggle \= document.getElementById('must-revalidate-toggle');  
  var proxyRevalidateToggle \= document.getElementById('proxy-revalidate-toggle');  
  var immutableToggle \= document.getElementById('immutable-toggle');  
  var noTransformToggle \= document.getElementById('no-transform-toggle');

  var maxAgeInput \= document.getElementById('max-age-input');  
  var sMaxageInput \= document.getElementById('s-maxage-input');  
  var swrInput \= document.getElementById('swr-input');  
  var sieInput \= document.getElementById('sie-input');

  var sensitiveToggle \= document.getElementById('sensitive-toggle');  
  var staticToggle \= document.getElementById('static-toggle');  
  var htmlToggle \= document.getElementById('html-toggle');

  var varyInput \= document.getElementById('vary-input');  
  var surrogateInput \= document.getElementById('surrogate-input');

  var auditScoreCircle \= document.getElementById('audit-score-circle');  
  var auditGrade \= document.getElementById('audit-grade');  
  var auditScore \= document.getElementById('audit-score');  
  var auditHeadline \= document.getElementById('audit-headline');  
  var auditSummaryText \= document.getElementById('audit-summary-text');  
  var auditList \= document.getElementById('audit-list');

  var scenarioTimeline \= document.getElementById('scenario-timeline');  
  var serverCodeBlock \= document.getElementById('server-code-block');  
  var copyBtn \= document.getElementById('copy-code-btn');

  function init() {  
    setupEventListeners();  
    applyStateToUI();  
    renderAll();  
  }

  function setupEventListeners() {  
    // Preset dropdown  
    if (presetSelect) {  
      presetSelect.addEventListener('change', function () {  
        var p \= PRESETS\[presetSelect.value\];  
        if (p) {  
          Object.keys(p).forEach(function (k) {  
            if (k in state) state\[k\] \= p\[k\];  
          });  
          applyStateToUI();  
          renderAll();  
        }  
      });  
    }

    // Visibility segments  
    setupSegmentGroup('visibility-segments', 'visibility', function (val) {  
      state.visibility \= val;  
      renderAll();  
    });

    // Checkboxes  
    setupCheckbox(noStoreToggle, 'noStore');  
    setupCheckbox(noCacheToggle, 'noCache');  
    setupCheckbox(mustRevalidateToggle, 'mustRevalidate');  
    setupCheckbox(proxyRevalidateToggle, 'proxyRevalidate');  
    setupCheckbox(immutableToggle, 'immutable');  
    setupCheckbox(noTransformToggle, 'noTransform');

    setupCheckbox(sensitiveToggle, 'isSensitiveData');  
    setupCheckbox(staticToggle, 'isStaticAsset');  
    setupCheckbox(htmlToggle, 'isHtmlDocument');

    // Number Inputs  
    setupNumberInput(maxAgeInput, 'maxAge');  
    setupNumberInput(sMaxageInput, 'sMaxage');  
    setupNumberInput(swrInput, 'staleWhileRevalidate');  
    setupNumberInput(sieInput, 'staleIfError');

    // Text Inputs  
    if (varyInput) {  
      varyInput.addEventListener('input', function () {  
        state.varyHeader \= varyInput.value;  
        renderAll();  
      });  
    }  
    if (surrogateInput) {  
      surrogateInput.addEventListener('input', function () {  
        state.surrogateControl \= surrogateInput.value;  
        renderAll();  
      });  
    }

    // Scenario Pills  
    var scenarioPills \= document.querySelectorAll('.scenario-pill');  
    scenarioPills.forEach(function (pill) {  
      pill.addEventListener('click', function () {  
        scenarioPills.forEach(function (btn) { btn.classList.remove('active'); });  
        pill.classList.add('active');  
        state.activeScenario \= pill.getAttribute('data-scenario');  
        renderSimulator();  
      });  
    });

    // Server Tabs  
    var serverTabs \= document.querySelectorAll('.server-tab');  
    serverTabs.forEach(function (tab) {  
      tab.addEventListener('click', function () {  
        serverTabs.forEach(function (btn) { btn.classList.remove('active'); });  
        tab.classList.add('active');  
        state.activeTab \= tab.getAttribute('data-tab');  
        renderServerCode();  
      });  
    });

    // Copy Button  
    if (copyBtn) {  
      copyBtn.addEventListener('click', function () {  
        var text \= serverCodeBlock.textContent;  
        navigator.clipboard.writeText(text).then(function () {  
          copyBtn.textContent \= 'Copied\!';  
          setTimeout(function () {  
            copyBtn.textContent \= 'Copy Config';  
          }, 2000);  
        });  
      });  
    }  
  }

  function setupCheckbox(el, stateKey) {  
    if (\!el) return;  
    el.addEventListener('change', function () {  
      state\[stateKey\] \= el.checked;  
      renderAll();  
    });  
  }

  function setupNumberInput(el, stateKey) {  
    if (\!el) return;  
    el.addEventListener('input', function () {  
      var val \= parseInt(el.value, 10);  
      state\[stateKey\] \= isNaN(val) ? 0 : val;  
      renderAll();  
    });  
  }

  function setupSegmentGroup(containerId, stateKey, onSelect) {  
    var container \= document.getElementById(containerId);  
    if (\!container) return;  
    var buttons \= container.querySelectorAll('.segment-btn');  
    buttons.forEach(function (btn) {  
      btn.addEventListener('click', function () {  
        buttons.forEach(function (b) { b.className \= 'segment-btn'; });  
        var val \= btn.getAttribute('data-val');  
        btn.classList.add('active');  
        if (val \=== 'private') btn.classList.add('emerald');  
        else if (val \=== 'public') btn.classList.add('primary');  
        else btn.classList.add('amber');  
        onSelect(val);  
      });  
    });  
  }

  function applyStateToUI() {  
    updateSegmentActive('visibility-segments', state.visibility);

    if (noStoreToggle) noStoreToggle.checked \= state.noStore;  
    if (noCacheToggle) noCacheToggle.checked \= state.noCache;  
    if (mustRevalidateToggle) mustRevalidateToggle.checked \= state.mustRevalidate;  
    if (proxyRevalidateToggle) proxyRevalidateToggle.checked \= state.proxyRevalidate;  
    if (immutableToggle) immutableToggle.checked \= state.immutable;  
    if (noTransformToggle) noTransformToggle.checked \= state.noTransform;

    if (maxAgeInput) maxAgeInput.value \= state.maxAge;  
    if (sMaxageInput) sMaxageInput.value \= state.sMaxage;  
    if (swrInput) swrInput.value \= state.staleWhileRevalidate;  
    if (sieInput) sieInput.value \= state.staleIfError;

    if (sensitiveToggle) sensitiveToggle.checked \= state.isSensitiveData;  
    if (staticToggle) staticToggle.checked \= state.isStaticAsset;  
    if (htmlToggle) htmlToggle.checked \= state.isHtmlDocument;

    if (varyInput) varyInput.value \= state.varyHeader;  
    if (surrogateInput) surrogateInput.value \= state.surrogateControl;  
  }

  function updateSegmentActive(containerId, activeVal) {  
    var container \= document.getElementById(containerId);  
    if (\!container) return;  
    var buttons \= container.querySelectorAll('.segment-btn');  
    buttons.forEach(function (btn) {  
      var val \= btn.getAttribute('data-val');  
      btn.className \= 'segment-btn';  
      if (val \=== activeVal) {  
        btn.classList.add('active');  
        if (val \=== 'private') btn.classList.add('emerald');  
        else if (val \=== 'public') btn.classList.add('primary');  
        else btn.classList.add('amber');  
      }  
    });  
  }

  function renderAll() {  
    renderPostureBadge();  
    renderAuditor();  
    renderSimulator();  
    renderServerCode();  
  }

  function renderPostureBadge() {  
    var posture \= window.CacheEngine.evaluateCachingPosture(state);  
    if (posture.classification \=== 'UNCACHED\_SENSITIVE') {  
      postureBadge.className \= 'posture-badge secure';  
      postureBadgeText.textContent \= 'UNCACHED\_SENSITIVE (Zero-Retention)';  
    } else if (posture.classification \=== 'IMMUTABLE\_STATIC') {  
      postureBadge.className \= 'posture-badge optimal';  
      postureBadgeText.textContent \= 'IMMUTABLE\_STATIC (0ms Revalidation)';  
    } else if (state.isSensitiveData && \!state.noStore && (state.visibility \=== 'public' || state.sMaxage \> 0)) {  
      postureBadge.className \= 'posture-badge hazard';  
      postureBadgeText.textContent \= 'CRITICAL PII CACHE LEAK HAZARD';  
    } else if (posture.classification \=== 'SHARED\_CDN\_CACHED') {  
      postureBadge.className \= 'posture-badge optimal';  
      postureBadgeText.textContent \= 'SHARED\_CDN\_CACHED (Public Edge)';  
    } else {  
      postureBadge.className \= 'posture-badge secure';  
      postureBadgeText.textContent \= posture.classification;  
    }  
  }

  function renderAuditor() {  
    var audit \= window.CacheAuditor.auditPolicy(state);

    auditScore.textContent \= audit.score \+ '/100';  
    auditGrade.textContent \= audit.grade;

    var gradeClass \= 'grade-F';  
    if (audit.grade \=== 'A+' || audit.grade \=== 'A') gradeClass \= 'grade-A';  
    else if (audit.grade \=== 'B') gradeClass \= 'grade-B';  
    else if (audit.grade \=== 'C') gradeClass \= 'grade-C';  
    else if (audit.grade \=== 'D') gradeClass \= 'grade-D';

    auditScoreCircle.className \= 'score-badge-circle ' \+ gradeClass;

    if (audit.score \>= 95\) {  
      auditHeadline.textContent \= 'Zero-Retention Fortified Posture';  
      auditSummaryText.textContent \= 'No PII leakage surface. Responses completely prevented from intermediate caching.';  
    } else if (audit.score \>= 80\) {  
      auditHeadline.textContent \= 'Compliant Production Posture';  
      auditSummaryText.textContent \= 'Caching rules aligned with minor optimization or header tuning opportunities.';  
    } else if (audit.score \>= 60\) {  
      auditHeadline.textContent \= 'Suboptimal Caching Configuration';  
      auditSummaryText.textContent \= 'Risk of stale deployments, high egress costs, or incomplete proxy revalidation.';  
    } else {  
      auditHeadline.textContent \= 'High-Risk Vulnerable Caching Posture';  
      auditSummaryText.textContent \= 'Severe risk of personal data exposure or cross-user session pollution in shared caches.';  
    }

    var listHtml \= '';  
    audit.checks.forEach(function (c) {  
      listHtml \+= '\<div class="audit-item status-' \+ c.status \+ '"\>';  
      listHtml \+= '  \<div class="audit-item-top"\>';  
      listHtml \+= '    \<span class="audit-item-title"\>' \+ escapeHtml(c.title) \+ '\</span\>';  
      listHtml \+= '    \<span class="audit-item-badge ' \+ c.status \+ '"\>' \+ c.status \+ ' (' \+ c.score \+ '/' \+ c.weight \+ ')\</span\>';  
      listHtml \+= '  \</div\>';  
      listHtml \+= '  \<p class="audit-item-detail"\>' \+ escapeHtml(c.detail) \+ '\</p\>';  
      listHtml \+= '\</div\>';  
    });

    auditList.innerHTML \= listHtml;  
  }

  function renderSimulator() {  
    var scenario \= window.CacheSimulator.SCENARIOS\[state.activeScenario\];  
    if (\!scenario) return;

    var html \= '';  
    scenario.steps.forEach(function (step) {  
      html \+= '\<div class="timeline-step"\>';  
      html \+= '  \<div\>';  
      html \+= '    \<div class="timeline-phase"\>' \+ escapeHtml(step.phase) \+ '\</div\>';  
      html \+= '    \<div class="timeline-actor"\>' \+ escapeHtml(step.actor) \+ '\</div\>';  
      html \+= '  \</div\>';  
      html \+= '  \<div\>';  
      html \+= '    \<div class="timeline-action"\>' \+ escapeHtml(step.action) \+ '\</div\>';  
      html \+= '    \<div class="timeline-detail"\>' \+ escapeHtml(step.detail) \+ '\</div\>';  
      html \+= '  \</div\>';  
      html \+= '  \<div class="timeline-badge"\>';  
      html \+= '    \<span class="' \+ step.badgeClass \+ '"\>' \+ escapeHtml(step.badge) \+ '\</span\>';  
      html \+= '  \</div\>';  
      html \+= '\</div\>';  
    });

    scenarioTimeline.innerHTML \= html;  
  }

  function renderServerCode() {  
    var code \= '';  
    switch (state.activeTab) {  
      case 'nginx':  
        code \= window.ServerSynthesizer.toNginx(state);  
        break;  
      case 'apache':  
        code \= window.ServerSynthesizer.toApache(state);  
        break;  
      case 'cf-pages':  
        code \= window.ServerSynthesizer.toCloudflarePages(state);  
        break;  
      case 'cf-worker':  
        code \= window.ServerSynthesizer.toCloudflareWorker(state);  
        break;  
      case 'cloudfront':  
        code \= window.ServerSynthesizer.toCloudFront(state);  
        break;  
      case 'caddy':  
        code \= window.ServerSynthesizer.toCaddy(state);  
        break;  
      case 'nextjs':  
        code \= window.ServerSynthesizer.toNextJs(state);  
        break;  
      default:  
        code \= window.ServerSynthesizer.toNginx(state);  
    }

    serverCodeBlock.textContent \= code;  
  }

  function escapeHtml(str) {  
    if (\!str) return '';  
    return String(str)  
      .replace(/&/g, '\&amp;')  
      .replace(/\</g, '\&lt;')  
      .replace(/\>/g, '\&gt;')  
      .replace(/"/g, '\&quot;')  
      .replace(/'/g, '&\#039;');  
  }

  if (document.readyState \=== 'loading') {  
    document.addEventListener('DOMContentLoaded', init);  
  } else {  
    init();  
  }  
})();  
