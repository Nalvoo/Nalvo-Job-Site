# whoami.html – Personal Portfolio Page Showcase

> **Note:** Technical showcase of selected frontend patterns. Personal content, contact details, and backend endpoints are intentionally omitted.

Live: **[nalvo.hu/whoami](https://nalvo.hu/whoami)**  
Stack: Vanilla HTML/CSS/JS · No frameworks · No build step

---

## Table of Contents

- [Design System](#design-system)
- [Canvas Terminal Background](#canvas-terminal-background)
- [Mouse Spotlight Effect](#mouse-spotlight-effect)
- [Scroll Reveal – IntersectionObserver](#scroll-reveal--intersectionobserver)
- [Language Switcher – View Transitions API](#language-switcher--view-transitions-api)
- [Social Buttons – JS-measured Width Animation](#social-buttons--js-measured-width-animation)
- [Spotify Now Playing + Progress Bar](#spotify-now-playing--progress-bar)
- [FAQ Accordion – CSS Grid Trick](#faq-accordion--css-grid-trick)
- [Cloudflare Turnstile – Lazy Init](#cloudflare-turnstile--lazy-init)

---

## Design System

Single-file CSS variables — no preprocessor, no framework.

```css
:root {
  --bg: #030014;
  --surface: rgba(255,255,255,0.028);
  --surface-hover: rgba(255,255,255,0.05);
  --border: rgba(255,255,255,0.06);
  --text: #e8e9f0;
  --text-muted: rgba(255,255,255,0.38);
  --accent: #8b5cf6;
  --accent2: #a78bfa;
  --accent3: #c4b5fd;
  --radius: 18px;
  --mono: 'JetBrains Mono', monospace;
}
```

Animated gradient text — CSS only, no JS:

```css
.grad {
  background: linear-gradient(135deg, #c4b5fd 0%, #a78bfa 45%, #8b5cf6 100%);
  background-size: 200% 200%;
  animation: gradShift 6s ease infinite;
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

@keyframes gradShift {
  0%,100% { background-position: 0% 50%; }
  50%      { background-position: 100% 50%; }
}
```

---

## Canvas Terminal Background

Fixed full-screen canvas with scrolling terminal lines. Multiple independent "streams" at different speeds, positions, and opacities. Prompt lines (`$`) are brighter and colored differently.

```js
(function() {
  const canvas = document.getElementById('bg-canvas');
  const ctx    = canvas.getContext('2d');
  let W, H, streams = [];

  const LINES = [
    '$ systemctl status nginx',
    'Active: active (running)',
    '$ tail -f /var/log/nginx/access.log',
    '127.0.0.1 - - [GET /api/content HTTP/1.1] 200',
    '$ git log --oneline',
    'a3f9c12 fix: nginx rate limit headers',
    'b7e1d04 feat: HLS streaming pipeline',
    '$ df -h',
    'Filesystem  Size  Used Avail Use%',
    '/dev/sda1    80G   31G   49G  39%',
    // ...
  ];

  function initStreams() {
    const count = Math.floor(W / 340) + 2;
    streams = Array.from({ length: count }, (_, i) => ({
      x:            (W / count) * i + Math.random() * 60 - 30,
      y:            Math.random() * H * 1.5 - H * 0.5,
      speed:        0.6 + Math.random() * 0.7,
      lineIndex:    Math.floor(Math.random() * LINES.length),
      lineSpacing:  22 + Math.random() * 4,
      alpha:        0.07 + Math.random() * 0.05,
      linesVisible: 8 + Math.floor(Math.random() * 6),
    }));
  }

  function draw() {
    ctx.clearRect(0, 0, W, H);
    ctx.font = '11px "JetBrains Mono", monospace';

    streams.forEach(s => {
      s.y += s.speed;
      if (s.y > H + 200) {
        s.y = -s.linesVisible * s.lineSpacing - Math.random() * 200;
        s.lineIndex = Math.floor(Math.random() * LINES.length);
      }

      for (let i = 0; i < s.linesVisible; i++) {
        const lineY = s.y + i * s.lineSpacing;
        if (lineY < -20 || lineY > H + 20) continue;

        const line = LINES[(s.lineIndex + i) % LINES.length];

        // Fade in/out at viewport edges
        const fade = Math.min(
          Math.min(1, lineY / 80),
          Math.min(1, (H - lineY) / 80)
        );

        const isPrompt = line.startsWith('$');
        const alpha    = s.alpha * fade * (isPrompt ? 1.6 : 1);

        ctx.fillStyle = isPrompt
          ? `rgba(99,102,241,${alpha})`   // indigo for prompts
          : `rgba(200,210,230,${alpha})`; // dim white for output

        ctx.fillText(line, s.x, lineY);
      }
    });

    requestAnimationFrame(draw);
  }

  function resize() {
    W = canvas.width  = window.innerWidth;
    H = canvas.height = window.innerHeight;
    initStreams();
  }

  window.addEventListener('resize', resize);
  resize();
  draw();
})();
```

---

## Mouse Spotlight Effect

Radial gradient that follows the cursor — pure CSS + 2 lines of JS:

```css
#spotlight {
  position: fixed;
  width: 600px;
  height: 600px;
  border-radius: 50%;
  pointer-events: none;
  z-index: 2;
  transform: translate(-50%, -50%);
  background: radial-gradient(circle, rgba(139,92,246,0.09) 0%, transparent 70%);
  transition: opacity 0.3s;
}
```

```js
const spotlight = document.getElementById('spotlight');
document.addEventListener('mousemove', e => {
  spotlight.style.left = e.clientX + 'px';
  spotlight.style.top  = e.clientY + 'px';
});
```

---

## Scroll Reveal – IntersectionObserver

Elements fade + slide in as they enter the viewport. Staggered delay on list items via inline `transitionDelay`.

```js
const revealObs = new IntersectionObserver((entries) => {
  entries.forEach(e => {
    if (e.isIntersecting) {
      e.target.classList.add('visible');
      revealObs.unobserve(e.target); // observe once, then stop
    }
  });
}, { threshold: 0.06, rootMargin: '0px 0px -40px 0px' });

// Staggered delay on FAQ items
document.querySelectorAll('.faq-list').forEach(list => {
  list.querySelectorAll('.faq-item').forEach((item, i) => {
    item.style.transitionDelay = (i * 0.07) + 's';
  });
});

// Observe everything
document.querySelectorAll('.reveal').forEach(el => revealObs.observe(el));
```

```css
.reveal {
  opacity: 0;
  transform: translateY(18px);
  transition: opacity 0.55s cubic-bezier(0.4,0,0.2,1),
              transform 0.55s cubic-bezier(0.4,0,0.2,1);
}
.reveal.visible {
  opacity: 1;
  transform: none;
}
```

---

## Language Switcher – View Transitions API

HU/EN toggle with slide animation. Uses the View Transitions API when available, falls back to manual `requestAnimationFrame` double-buffering.

```js
function doSwitchLang() {
  const scrollY = window.scrollY; // preserve scroll position
  isEN = !isEN;

  // Update all [data-hu][data-en] elements
  document.querySelectorAll('[data-hu][data-en]').forEach(el => {
    el.innerHTML = isEN ? el.dataset.en : el.dataset.hu;
  });

  const leaving  = document.getElementById(isEN ? 'panelHU' : 'panelEN');
  const entering = document.getElementById(isEN ? 'panelEN' : 'panelHU');

  // Slide out
  leaving.style.transition = 'opacity 0.2s ease, transform 0.2s ease';
  leaving.style.opacity    = '0';
  leaving.style.transform  = isEN ? 'translateX(-24px)' : 'translateX(24px)';

  setTimeout(() => {
    leaving.style.display = 'none';
    entering.style.display = 'flex';
    entering.style.opacity   = '0';
    entering.style.transform = isEN ? 'translateX(24px)' : 'translateX(-24px)';

    // Double rAF ensures the browser has painted the initial state
    requestAnimationFrame(() => requestAnimationFrame(() => {
      entering.style.transition = 'opacity 0.38s ease, transform 0.38s cubic-bezier(0.34,1.2,0.64,1)';
      entering.style.opacity    = '1';
      entering.style.transform  = 'translateX(0)';
      window.scrollTo(0, scrollY);
    }));
  }, 200);
}

function switchLang() {
  // View Transitions API — smooth cross-fade if supported
  if (document.startViewTransition) {
    document.startViewTransition(() => doSwitchLang());
  } else {
    doSwitchLang();
  }
}
```

---

## Social Buttons – JS-measured Width Animation

Buttons start as 46×46px squares (icon only). On hover they expand to show a label. The expanded width is measured once via JS (not hardcoded) to handle any label length.

```css
.social-btn {
  width: 46px;
  height: 46px;
  overflow: hidden;
  transition: width 0.45s cubic-bezier(0.22,1,0.36,1);
}
.social-btn .sb-label {
  opacity: 0;
  transition: opacity 0.25s ease;
}
.social-btn:hover .sb-label { opacity: 1; }
```

```js
document.querySelectorAll('.social-btn').forEach(btn => {
  let expandedW = null;

  function getExpandedWidth() {
    btn.style.transition = 'none';
    btn.style.width = 'max-content';
    const w = btn.scrollWidth;
    btn.style.width = '46px';
    btn.offsetWidth; // force reflow
    btn.style.transition = '';
    return w;
  }

  btn.addEventListener('mouseenter', () => {
    if (expandedW === null) expandedW = getExpandedWidth(); // measure once
    btn.style.width = expandedW + 'px';
  });
  btn.addEventListener('mouseleave', () => {
    btn.style.width = '46px';
  });

  // Touch / keyboard support
  btn.addEventListener('focus', () => {
    if (expandedW === null) expandedW = getExpandedWidth();
    btn.style.width = expandedW + 'px';
  });
  btn.addEventListener('blur', () => { btn.style.width = '46px'; });
});
```

---

## Spotify Now Playing + Progress Bar

Polls `/api/spotify/now-playing` every 15 seconds. When a track is playing, a progress bar is drawn client-side and ticked every second using the elapsed time since the last API response — no extra API calls needed between polls.

```js
let currentProgress = 0;
let currentDuration = 0;
let trackFetchedAt  = 0;
let progressInterval = null;

function startProgressTick() {
  if (progressInterval) clearInterval(progressInterval);
  progressInterval = setInterval(() => {
    const elapsed   = Date.now() - trackFetchedAt;
    const estimated = Math.min(currentProgress + elapsed, currentDuration);
    const pct       = currentDuration > 0 ? (estimated / currentDuration) * 100 : 0;
    progressBar.style.width  = pct + '%';
    progressTime.textContent = fmtTime(estimated);
  }, 1000);
}

async function loadTrack() {
  const data = await fetch('/api/spotify/now-playing').then(r => r.json());

  if (data.is_playing && data.track?.progress_ms != null) {
    currentProgress = data.track.progress_ms;
    currentDuration = data.track.duration_ms;
    trackFetchedAt  = Date.now();

    // Set bar without transition first, then re-enable
    progressBar.style.transition = 'none';
    progressBar.style.width = (currentProgress / currentDuration * 100) + '%';
    setTimeout(() => { progressBar.style.transition = 'width 1s linear'; }, 50);

    startProgressTick();
  }
}

loadTrack();
setInterval(loadTrack, 15000);
```

Album art blur background — the album cover is set as a blurred, darkened background behind the player card:

```css
.sp-bg-blur {
  position: absolute;
  top: 0; left: 0; right: 0;
  height: 160px;
  z-index: 0;
  opacity: 0;
  transition: opacity 0.8s ease;
  pointer-events: none;
}
.sp-bg-blur img {
  width: 100%; height: 100%;
  object-fit: cover;
  filter: blur(60px) saturate(1.8) brightness(0.25);
  transform: scale(1.2);
}
.sp-bg-blur.visible { opacity: 1; }
```

---

## FAQ Accordion – CSS Grid Trick

Smooth height animation without `max-height` hacks. Uses `grid-template-rows` transition from `0fr` to `1fr` — no JS height calculation needed.

```css
.faq-body {
  display: grid;
  grid-template-rows: 0fr;
  transition: grid-template-rows 0.35s cubic-bezier(0.4,0,0.2,1);
}
.faq-item.open .faq-body {
  grid-template-rows: 1fr;
}
.faq-body-inner {
  overflow: hidden; /* required for the 0fr trick to work */
}
```

```js
function faqBtnHandler() {
  const item   = this.closest('.faq-item');
  const isOpen = item.classList.contains('open');

  // Close all open items first
  document.querySelectorAll('.faq-item.open').forEach(el => {
    el.classList.remove('open');
    el.querySelector('.faq-btn').setAttribute('aria-expanded', 'false');
  });

  // Toggle clicked item
  if (!isOpen) {
    item.classList.add('open');
    this.setAttribute('aria-expanded', 'true');
  }
}
```

---

## Cloudflare Turnstile – Lazy Init

The Turnstile site key is fetched from the backend at runtime — not hardcoded in the HTML. The widget renders only after the script is loaded (polling with `setTimeout`).

```js
(async function initTurnstile() {
  try {
    const { siteKey } = await fetch('/api/config/turnstile').then(r => r.json());
    if (!siteKey) return;

    const container = document.getElementById('turnstile-container');
    if (!container) return;

    function renderWhenReady() {
      if (window.turnstile) {
        window.turnstile.render(container, {
          sitekey: siteKey,
          theme: 'dark',
          size: 'flexible',
        });
      } else {
        setTimeout(renderWhenReady, 100); // wait for script to load
      }
    }
    renderWhenReady();
  } catch(e) { /* silent fail — page works without captcha */ }
})();
```

---

## What's NOT here

Intentionally omitted:

- Contact details and personal information
- Backend API implementations (`/api/spotify/now-playing`, `/api/health`, etc.)
- Server stats widget data source
- FAQ analytics tracking endpoint
- Full HTML structure and content

---

*Single HTML file. Zero dependencies. Zero build step.*  
*All CSS and JS inline — loads in one request.*
