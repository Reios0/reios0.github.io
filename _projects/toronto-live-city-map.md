---
title: Toronto Live City Map
description: A full-stack geospatial web application that serves as a real-time
  operations dashboard for the city of Toronto. Designed as a centralized
  platform for exploring public transit, emergency presence, and civic
  infrastructure.
date: 2026-07-02
skills:
  - React
  - JavaScript
  - TypeScript
  - Leaflet.js
  - Node.js
  - Express
  - Vite
  - ESLint
  - Prettier
  - Git & GitHub
  - GitHub Actions
  - Porkbun
  - Railway
link: https://torontomap.ca/
screenshot: /assets/images/torontomap-screenshot.png
---
# Table of contents

1. [Overview](#overview)
2. [Key features](#key-features)
3. [Architecture](#architecture)
4. [Tech stack](#tech-stack)
  - [Frontend](#frontend)
  - [Backend](#backend)
  - [Communication](#communication)
  - [Testing](#testing)
  - [Code quality](#code-quality)
  - [CI/CD](#cicd)
  - [Hosting & SSL](#hosting--ssl)
  - [Domain & DNS](#domain--dns)
5. [Technical challenges](#technical-challenges)
  - [Normalizing seven data sources into one](#normalizing-seven-data-sources-into-one)
  - [Rendering thousands of markers without slowing down](#rendering-thousands-of-markers-without-slowing-down)
  - [Fixing a memory spike in a 4-million-row file](#fixing-a-memory-spike-in-a-4-million-row-file)
  - [Making an interactive map accessible](#making-an-interactive-map-accessible)
6. [Other engineering decisions](#other-engineering-decisions)
7. [Data sources](#data-sources)
8. [What's next](#whats-next)

## Overview

Toronto publishes a wealth of public data, but it's scattered across separate systems. Toronto Live City Map brings together live transit positions, emergency incidents, Bike Share availability, and public amenities into a single interactive map, eliminating the need to check multiple websites.

The application visualizes approximately 9,000 TTC stops, more than 1,000 Bike Share stations, over 300 public washrooms, several hundred live TTC vehicles, and live emergency incidents within a single interface.

Behind that unified view is a significant data integration challenge. The backend ingests seven public data sources spanning six different formats (JSON, XML, GTFS, GTFS-Realtime, GBFS, and plain text), normalizing each into a shared internal model before serving it to the frontend. Supporting the transit layer also requires preprocessing GTFS datasets containing more than 126,000 trips and over 4 million stop-time records.

## Key features

- **Live TTC vehicle tracking**: bus and streetcar positions update every 10 seconds; clicking a vehicle shows route, vehicle ID, bearing, speed, and occupancy. Train positions aren't publicly available, so only buses and streetcars are tracked.
- **Police and fire activity**: active incidents refresh every 20 minutes for police and every 5 minutes for fire, with call type, cross streets, dispatch time, alarm level, and units sent.
- **Bike Share Toronto stations**: live dock and bike availability every 10 seconds, static station details refresh daily.
- **Public washroom locations**: status, hours, address, and accessibility details, refreshed daily.
- **TTC stops and stations**: static stop data refreshed biweekly, with arrivals every 30 seconds and service alerts every minute.

## Architecture

The frontend is a React single-page app built with Vite. The backend is a persistent Node/Express process rather than a serverless deployment, because Server-Sent Events need a long-lived connection to push updates, and serverless functions are built to terminate after a request finishes. Railway hosts both, with automatic SSL and a straightforward deploy path: a push to GitHub triggers linting and tests through GitHub Actions, and Railway deploys and health-checks the new version once those pass.

The backend serves data in two ways. Frequently changing information is normalized and pushed to connected clients over SSE as it updates. SSE was chosen over WebSockets because communication is entirely server-to-client, making it simpler to implement while still providing automatic reconnection and efficient one-way updates. Less volatile information is updated on a schedule, and fetched over REST on demand when a user clicks a marker for more detail.

Live feeds are polled by scheduled background jobs, normalized once on the server, then broadcast to all connected clients over SSE. This avoids every browser independently polling each upstream data source while ensuring every client receives the same view of the data.

## Tech stack

### Frontend

- React (with CSS modules)
- JavaScript
- TypeScript
- Leaflet (with MarkerCluster and LocateControl plugins)

### Backend

- Node.js
- Express
- TypeScript

### Communication

- REST (static and on-demand details)
- Server-sent events (live updates)

### Testing

- Vitest

### Code quality

- ESLint
- Prettier

### CI/CD

- GitHub Actions (linting and tests)
- Railway (automatic deploy)

### Hosting & SSL

- Railway

### Domain & DNS

- Porkbun

## Technical challenges

### Normalizing seven data sources into one

Each source has its own schema and its own idea of what a location looks like, so the backend converts everything into a shared internal structure. For example, one source may expose coordinates as `lat` and `lon`, another as `latitude` and `longitude`, while GTFS-Realtime represents them differently again. Regardless of the source, every location is normalized into the same internal model. A map component that renders a vehicle doesn't need to know whether its data came from a GTFS-RT feed or a JSON endpoint; it always receives the same fields in the same shape. Scheduled background tasks handle the refresh side of this: fast-changing data is pulled frequently, static data is refreshed less often, and the frontend doesn't need to understand each source's individual update schedule.

### Rendering thousands of markers without slowing down

Leaflet creates a DOM element per marker, which becomes a real performance problem once thousands of vehicles, incidents, and stations are on screen at once, especially during pan and zoom. Three changes addressed it:

1. Marker clustering (via Leaflet MarkerCluster plugin) groups nearby points when zoomed out and splits them apart on zoom in.
2. Viewport-based rendering keeps markers outside the visible area out of the DOM entirely, with pan/zoom recalculation debounced so continuous dragging doesn't trigger it on every frame.
3. Marker reuse matches incoming data to existing markers by ID, updating in place instead of tearing down and rebuilding the whole set on every refresh.

Marker clustering introduced an unexpected complication: when a marker moves between clusters, the plugin removes and recreates its DOM element, which resets any CSS transition applied to it. Smooth position interpolation through CSS wasn't reliable as a result, so marker movement is animated manually with `requestAnimationFrame`, interpolating between the previous and new coordinates on every frame regardless of what MarkerCluster does to the underlying DOM.

### Fixing a memory spike in a 4-million-row file

The TTC's GTFS stop time data is a text file with over 4 million rows. The original naive implementation loaded it into an array with one element per row, which meant millions of strings held in memory simultaneously before any of them were used, a worst-case space complexity of `O(mn)` for `m` columns and `n` rows. The issue didn't surface in local development on a machine with 16GB of RAM; it only became visible once the app was deployed to a server with a tighter memory budget. 

To make the numbers concrete: RSS (resident set size) is the total RAM the operating system has given the process at any moment. Both approaches start at around 94-98 MB. Downloading the zip file is the same for both, bringing RSS to roughly 341-343 MB as the raw bytes sit in a C++ buffer. 

From there the two approaches diverge. The naive version decompresses all three GTFS files into strings at the same time before parsing any of them. That simultaneous decompression is the single biggest spike: RSS hits 1262 MB with all three file strings in memory at once. The optimized version never does this as it decompresses one file, parses it, frees the buffer, then moves to the next. After parsing stops, RSS actually drops to 268 MB because the buffer was already released. 

The optimized version's worst moment comes later, during the stop routes step, which processes the 4-million-row stop_times file. That spike reaches 1301 MB which is slightly higher than the naive version's peak because the raw buffer for that one large file still has to be in memory while rows are being read. The generator helps with what gets kept after processing, not with the buffer that has to exist during it. 

After both finish and GC (garbage collection, the runtime's automatic memory cleanup) runs, the difference becomes clear. The naive version settles at 754 MB, a 660 MB increase from baseline. The optimized version settles at 541 MB, a 443 MB increase (a saving of around 213 MB). Notably, heapUsed after GC is identical in both cases (34.62 MB), meaning the actual useful data retained is the same size either way. The saving comes entirely from the optimized version's sequential approach leaving less unreachable memory for GC to deal with.

### Making an interactive map accessible

Interactive maps present accessibility challenges that lists and forms don't. Spatial relationships between markers can't be conveyed in a meaningful reading order, so traditional accessibility patterns don't translate directly. The application addresses those challenges in several ways.

Markers carry ARIA labels describing what they are, whether more detail is available, and what action is possible, so a screen reader user has context before interacting. Popups move focus in on open and return it to the originating marker on close, so keyboard users don't lose their place.

The harder problem was structural: Leaflet appends its control container (zoom, locate, attribution) after the map pane in the DOM, and since that pane can contain thousands of interactive markers, a keyboard user has to tab through all of them before reaching any control. The fix was a skip-navigation button that moves focus directly to the control container. It's implemented as a button rather than an anchor link because an anchor would append a fragment identifier to the URL, and the URL is already used for layer state management, so changing it during accessibility navigation would interfere with the application's routing. Since the app requires JavaScript to function, using a JavaScript-powered button doesn't remove functionality for any users.

Layer controls are custom-built rather than relying on Leaflet's defaults, which aren't keyboard-friendly out of the box. Animations are reduced or disabled when the user has `prefers-reduced-motion` set.

## Other engineering decisions

- **URL-based layer state**: active map layers are encoded in the URL rather than kept only in client state, so a configuration can be bookmarked or shared, and the frontend knows what to fetch before making any request.
- **In-memory data storage**: processed datasets are kept in server memory for speed and simplicity rather than an external cache. The tradeoff is that a restart or crash means re-downloading and re-parsing everything, causing a temporary memory spike. The current setup also only works correctly with a single server instance. If a second instance were added for load balancing, each would maintain its own isolated copy of the data, so clients routed to different instances could see different state. Redis would solve this because it runs as a separate process that all instances share, so a write from one server is immediately available to the others. It is still in-memory and fast, just no longer tied to a single process. The backend is structured so that replacing the in-memory store with Redis later would be a contained change.
- **Native** `<dialog>` **for data attribution**: the attribution panel uses the HTML `dialog` element for built-in focus management, keyboard support, and escape-to-close, while staying mounted in the DOM so search engines can still index it.
- `**useRef` for the Leaflet instance**: the map object and layer groups are stored in refs rather than component state, since they don't need to trigger React re-renders.
- `**divIcon` markers**: lightweight HTML-based icons instead of image assets, since the map can have a large number of markers on screen at once.
- **Obfuscated contact email**: the contact address is constructed dynamically on user interaction rather than placed in the HTML as plain text, to reduce automated scraping while satisfying OpenStreetMap's tile usage attribution requirement.

## Data sources

All data is used under the Toronto and Ontario open government licenses, from seven public endpoints:

- [Surface Routes and Schedules for BusTime](https://open.toronto.ca/dataset/surface-routes-and-schedules-for-bustime/)
- [TTC BusTime Real-Time Next Vehicle Arrival (NVAS)](https://open.toronto.ca/dataset/ttc-bustime-real-time-next-vehicle-arrival-nvas/)
- [TTC GTFS-Realtime (GTFS-RT)](https://open.toronto.ca/dataset/ttc-gtfs-realtime-gtfs-rt/)
- [Bike Share Toronto](https://open.toronto.ca/dataset/bike-share-toronto/)
- [Park Washroom Facilities](https://open.toronto.ca/dataset/washroom-facilities/)
- [Toronto Fire Active Incidents](https://www.toronto.ca/community-people/public-safety-alerts/alerts-notifications/toronto-fire-active-incidents/)
- [Toronto Police Service Calls for Service](https://experience.arcgis.com/experience/a22f5295933e48a5b0a4c90cd3c4cae1)

## What's next

A congestion layer built from live traffic flow data would make the map more useful for trip planning. Future work also includes incorporating temporary city events such as street closures and festivals so the map reflects not just permanent infrastructure, but the city's changing state throughout the day.