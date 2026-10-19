# Georama + GeoGirafe vs. Open-Source Geoportals in the Cloud Era

A comparative analysis of the **Georama backend + GeoGirafe frontend** stack against the main open-source geoportal alternatives, focused on architecture, performance, ease of map/OGC publishing, and accessibility to non-GIS specialists.

The georama/geogirafe findings below are grounded in the two codebases in this workspace; competitor facts are from their official documentation and repositories (sources at the end). Note: Georama is at a **Pre-Alpha** development stage per its own packaging classifier — read the maturity column accordingly.

---

## 1. What each stack actually is

| Stack | Backend | Frontend / viewer | Rendering engine | Origin |
|---|---|---|---|---|
| **Georama + GeoGirafe** | Django 5.2 apps (Python ≥3.11), PostGIS, Redis job queue, ASGI (uvicorn/gunicorn) | GeoGirafe (framework-free TypeScript Web Components + OpenLayers/Cesium) | **qgis-server-light** (headless QGIS) + **pygeoapi** for OGC API Features | OPENGIS.ch (QField / QFieldCloud team) |
| **GeoMapFish + GeoGirafe** | c2cgeoportal (Python/Pyramid), PostGIS | GeoGirafe (same viewer) | MapServer (default) or QGIS Server / GeoServer as OGC source | Camptocamp |
| **Lizmap** | Lizmap Web Client (PHP / Jelix), PostGIS | Lizmap JS UI (OpenLayers) | QGIS Server + Lizmap QGIS Server plugin | 3Liz |
| **QGIS Cloud** | Hosted SaaS (managed servers + PostGIS) | QGIS Cloud web viewer | QGIS Server | Sourcepole |
| **GeoServer-based geoportal** (e.g. GeoNode) | GeoServer (Java), often GeoNode (Django) wrapper, PostGIS | GeoNode UI / MapStore (React) | GeoServer (GeoTools) | OSGeo community |
| **Masterportal** | None required — pure client toolkit; consumes any OGC services | Masterportal (OpenLayers + Vue.js) | Any WMS/WFS/WMTS/OGC API source | City of Hamburg (LGV) |

Key structural distinction: **Georama and GeoMapFish are full backend+frontend platforms**; **Masterportal is a frontend-only toolkit**; **GeoServer is a rendering server** that needs a portal layer (GeoNode/MapStore) on top; **QGIS Cloud is a managed hosted service**, not self-hosted software.

---

## 2. Architecture

| Criterion | Georama + GeoGirafe | GeoMapFish + GeoGirafe | Lizmap | QGIS Cloud | GeoServer / GeoNode | Masterportal |
|---|---|---|---|---|---|---|
| **Backend language** | Python / Django (modular apps) | Python / Pyramid | PHP / Jelix | Managed (QGIS Server) | Java (+ Python for GeoNode) | N/A (client only) |
| **Frontend tech** | Framework-free TS Web Components, µhtml, custom Proxy reactivity | Same (GeoGirafe); legacy ngeo = AngularJS/Angular | OpenLayers + custom JS | OpenLayers viewer | React (MapStore) / templated | OpenLayers + Vue.js (+ legacy Backbone) |
| **Rendering approach** | OGC protocol implemented in Django; rendering **offloaded to qgis-server-light workers via Redis queue** (decoupled compute tier) | MapServer/QGIS Server processes | QGIS Server processes | QGIS Server (managed) | GeoServer JVM | Delegated to whatever OGC service is configured |
| **Data / styling source of truth** | **QGIS project files** (native import) | Config + mapfiles / QGIS projects | **QGIS project files** | **QGIS project files** | GeoServer stores + SLD styling | External services |
| **State / config model** | 12-factor env vars (django-configurations, `GEORAMA_*`); runtime `config.json` for viewer | Config files + build step | INI + QGIS project | Managed UI | XML/REST config | JSON config, no build needed to reconfigure |
| **Cloud-native posture** | Container-first, stateless app tier, externalized state (PostGIS/Redis/Keycloak), **independently scalable render workers**; ASGI async. No bundled k8s/Helm | Containerized (Docker); heavier multi-service | Docker images available | SaaS — fully managed | Containerizable; JVM is heavier to scale | Static assets — trivial to host/CDN |
| **Auth model** | Django auth (Argon2) + OIDC/Keycloak via allauth + JWT bearer middleware for APIs/OGC | GeoMapFish auth (OAuth) | Lizmap ACL + QGIS | Account-based (SaaS) | GeoNode/GeoServer security | Delegated to services/proxy |
| **Maturity** | **Pre-Alpha** (new, 2025–) | Mature, long production history | Mature | Mature (SaaS) | Very mature | Mature, widely deployed in Germany |

**Architectural takeaways for Georama+GeoGirafe:**
- The **decoupled rendering tier** (Django emits OGC responses; qgis-server-light does the heavy raster/vector work behind a Redis queue) is the most cloud-native trait — the web tier stays stateless and the render workers scale independently. This is a cleaner separation than a monolithic MapServer/GeoServer process.
- **QGIS project as the single source of truth** removes a whole configuration-translation layer that GeoServer (SLD) and MapServer (mapfiles) impose.
- GeoGirafe being **framework-free Web Components** avoids the React/Angular/Vue runtime that Masterportal (Vue), MapStore (React) and legacy ngeo (Angular) carry. Cesium (3D) is lazily loaded, so a 2D session never pays the 3D cost.

---

## 3. Performance

| Criterion | Georama + GeoGirafe | GeoMapFish + GeoGirafe | Lizmap | QGIS Cloud | GeoServer / GeoNode | Masterportal |
|---|---|---|---|---|---|---|
| **Frontend runtime weight** | Low — no UI framework; aggressive Vite code-splitting; Cesium loaded on demand | Same viewer (low); legacy ngeo heavier | Moderate | Moderate | Higher (React/MapStore) | Moderate (Vue) |
| **Render scaling** | Horizontal — add qgis-server-light replicas behind Redis queue; web tier stateless | Scale MapServer/QGIS Server instances | Scale QGIS Server (FCGI workers) | Managed by provider | Scale JVM instances (memory-heavy) | Depends on backing services |
| **Async I/O** | Yes — ASGI, async OGC views | Mostly WSGI | PHP-FPM | Managed | Servlet threads | N/A |
| **3D handling** | On-demand Cesium chunk; 3D Tiles + terrain | Same | Limited | No | Via external | Extension-based 3D |
| **Typical bottleneck** | qgis-server-light throughput per render job | MapServer / QGIS Server throughput | QGIS Server FCGI capacity | Provider quotas | JVM memory / tuning | Backing OGC service latency |

Notes and caveats:
- No published head-to-head benchmarks exist for Georama at this stage (Pre-Alpha), so performance claims are **architectural**, not measured. The queue-based worker model is well suited to bursty/cloud workloads and to autoscaling, but each render still runs through headless QGIS, so per-tile cost is comparable to any QGIS-Server-based stack (Lizmap, QGIS Cloud, GeoMapFish-on-QGIS).
- GeoServer generally has strong raw throughput and mature caching (GeoWebCache) but a heavier JVM memory footprint to operate.
- Masterportal's performance is essentially the performance of whatever services it points at, plus a light client.

---

## 4. Ease of map / OGC service publishing

| Criterion | Georama + GeoGirafe | GeoMapFish | Lizmap | QGIS Cloud | GeoServer / GeoNode | Masterportal |
|---|---|---|---|---|---|---|
| **Publishing workflow** | Import QGIS project → "Publish from Project" builds layer tree → auto-registers internal OGC server → theme served to viewer | Admin config of layers/themes (more setup) | Configure project in QGIS + Lizmap plugin, copy to server | **One-click from QGIS Desktop plugin** | Create stores/layers, define SLD styles, publish services | No publishing — you attach existing services |
| **Styling fidelity from desktop** | High — QGIS rendering preserved via qgis-server-light | Varies by engine | High (QGIS Server) | High (QGIS Server) | Requires re-styling in SLD | N/A |
| **OGC services emitted** | WMS 1.3.0, WFS 2.0.0, **OGC API Features** (pygeoapi) | WMS/WFS/WMTS via MapServer/QGIS/GeoServer | WMS/WFS/WMTS + Lizmap services | WMS/WFS + OGC | WMS/WFS/WCS/WMTS/WPS + OGC API (broadest) | Consumes, does not emit |
| **Permissions granularity** | Fine-grained, down to attribute level; per-layer read checks on OGC responses | Role/functionality-based | Group-based ACL | Account-based | GeoNode per-resource security | Handled upstream |
| **Metadata management** | Flexible, extensible per-layer metadata | Limited | Some | Some | **GeoNode is metadata/catalog-led** | N/A |
| **Current gaps (Georama)** | WMTS "not yet implemented"; WMS `GetFeatureInfo` not implemented; only WMS 1.3.0 / WFS 2.0.0 accepted; Pre-Alpha | — | — | Vendor lock-in to SaaS | SLD styling learning curve | — |

**Publishing takeaways:**
- **QGIS Cloud is the easiest** for a pure QGIS user (literally one button), but it is a hosted SaaS with vendor lock-in and quotas.
- **Georama's edge** is offering a comparable QGIS-native, styling-preserving workflow **as self-hostable open-source software**, with modern OGC API Features and fine-grained (attribute-level) permissions baked in — while Lizmap requires the QGIS plugin + file sync, and GeoServer requires re-modeling data and SLD styling.
- GeoServer/GeoNode remains the broadest in raw OGC coverage (adds WCS/WPS), and GeoNode is the strongest as a metadata catalog.

---

## 5. Accessibility to non-GIS specialists

| Dimension | Georama + GeoGirafe | GeoMapFish | Lizmap | QGIS Cloud | GeoServer / GeoNode | Masterportal |
|---|---|---|---|---|---|---|
| **Publisher (data owner) skill floor** | QGIS knowledge enough; web publishing via admin UI, no SLD/mapfile authoring | Higher — config + deployment expertise | QGIS + server/plugin setup | **Lowest** — desktop button, no infra | High — GeoServer/SLD concepts | N/A (developer/integrator toolkit) |
| **End-user viewer UX** | Clean, modern GeoGirafe UI; runtime-configurable; per-user config overrides | Same viewer | Functional | Functional | MapStore modern; GeoNode utilitarian | Highly configurable, mature UX |
| **Web accessibility (a11y)** | Explicit design goal in GeoGirafe: `html[lang]` handling, keyboard activation, tooltip/`title` attributes (full WCAG needs manual audit) | Same viewer | Limited | Limited | Varies | Good; widely used in German public sector (a11y-regulated) |
| **Internationalization** | i18n manager, per-language JSON dictionaries, per-deployment string overrides, locale number formatting | Same | Multi-language | Multi-language | Multi-language | Strong (German public sector) |
| **Mobile** | Dedicated mobile interface + components; UA auto-redirect; Cordova APK build | Same viewer | Responsive | Responsive | Responsive | Responsive |
| **Admin experience** | Django admin (jazzmin theme) — familiar CRUD UI | Config-driven | INI/plugin | Web UI | Web admin | Config files |

**Accessibility takeaways:**
- For the **person publishing data**, Georama lowers the bar to "know QGIS + use an admin web UI" — no SLD, no mapfiles, no build step — second only to QGIS Cloud's one-click, but without the SaaS lock-in.
- For **end users and integrators**, GeoGirafe brings a modern, i18n-ready, mobile-capable UI with explicit web-accessibility intent, shared with GeoMapFish.
- Masterportal is strongest where formal public-sector accessibility compliance is a hard requirement, given its heavy German government deployment.

---

## 6. What makes Georama + GeoGirafe attractive

1. **QGIS-native, no translation layer.** Your QGIS project *is* the configuration and styling. qgis-server-light preserves desktop rendering. Contrast with GeoServer's SLD re-styling or MapServer mapfiles.
2. **Cloud-native by construction.** Stateless ASGI web tier + externalized state (PostGIS/Redis/Keycloak) + **independently scalable headless-QGIS render workers behind a queue**. Fits containers and autoscaling cleanly (though it ships no k8s/Helm yet).
3. **Modern, lightweight frontend.** GeoGirafe is framework-free Web Components with lazy-loaded 3D — lighter runtime than Vue (Masterportal), React (MapStore), or legacy Angular (ngeo).
4. **Standards-forward.** Ships OGC API Features (via pygeoapi) alongside classic WMS/WFS, and stays GeoMapFish `themes.json` v2.8 compatible — so you can reuse the GeoGirafe ecosystem.
5. **Low publishing skill floor + fine-grained security.** Web admin publishing (no SLD/mapfile) plus attribute-level permissions and OIDC/Keycloak SSO out of the box.
6. **Pedigree.** Built by OPENGIS.ch (QField / QFieldCloud), aligning naturally with a QGIS field-to-web pipeline.

### Honest trade-offs
- **Maturity: Pre-Alpha.** Not yet production-proven versus Lizmap, GeoMapFish, GeoServer, or Masterportal.
- **Feature gaps today:** WMTS and WMS `GetFeatureInfo` not yet implemented; WMS limited to 1.3.0, WFS to 2.0.0.
- **Narrower OGC breadth** than GeoServer (no WCS/WPS).
- **No bundled orchestration** (k8s/Helm) — cloud-native in design, DIY in deployment.
- **License** is under temporary terms with a permanent open-source license planned for early 2027 — worth confirming for adoption decisions.

---

### Sources
- Georama & GeoGirafe: this workspace's source (README, `docker-compose.yml`, `src/georama/{maps,webgis,core}`, GeoGirafe `package.json`, `src/components/map/component.ts`, `src/tools/...`).
- [Lizmap Web Client docs](https://docs.lizmap.com/) and [3Liz repo](https://github.com/3liz/lizmap-web-client).
- [QGIS Cloud docs](https://docs.qgiscloud.com/en/).
- [Masterportal (OpenLayers + Vue.js, MIT)](https://opencode.de/software/masterportal-413) and [Hamburg Geowerkstatt](https://www.hamburg.de/geowerkstatt/8856396/masterportal/).
- [c2cgeoportal / GeoMapFish](https://github.com/camptocamp/c2cgeoportal) and [MapServer at Camptocamp](https://camptocamp.com/mapserver).
- [GeoServer WMS/WFS docs](https://docs.geoserver.org/) and geodata software comparison notes.

*Content from external sources was rephrased for compliance with licensing restrictions.*
