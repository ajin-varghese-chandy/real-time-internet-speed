# Chrome Extension Permissions Explanation

## Host Permissions

### 1. `https://speed.cloudflare.com/*`
**Justification**: Primary speed testing endpoint
- **Specific Use**: Downloads test files (1 MB – 25 MB) to measure download throughput; uploads random payloads (256 KB – 512 KB) to measure upload speed; sends 1-byte requests for latency (ping/jitter) measurement
- **Why Essential**: Cloudflare's speed endpoint is purpose-built for bandwidth testing — it serves unthrottled, uncached, randomly-sized responses and accepts POST uploads, making it the most accurate freely-available speed testing service
- **Data Handling**: No personal data transmitted; requests contain only random binary payloads for upload testing and standard HTTP headers
- **User Benefit**: Provides accurate, real-time download speed, upload speed, ping, and jitter measurements

### 2. `https://www.google.com/*`
**Justification**: Latency measurement and fallback
- **Specific Use**: Sends requests to `/generate_204` (returns an empty 204 response with no body) to measure pure network round-trip latency for ping calculations
- **Why Essential**: The `generate_204` endpoint is the standard lightweight probe used by operating systems and browsers for connectivity checks — its empty response ensures measured latency reflects network conditions rather than transfer time
- **Limited Scope**: Only accesses the `/generate_204` endpoint; does not access Search, Gmail, or any other Google service
- **Data Handling**: Standard HTTP requests with no personal data; no response body is read
- **User Benefit**: Delivers accurate, steady-state ping measurements unaffected by file transfer overhead

### 3. `https://httpbin.org/*`
**Justification**: Secondary fallback speed testing source
- **Specific Use**: Downloads random byte payloads (`/bytes/{n}`) as a fallback when Cloudflare endpoints are unreachable
- **Why Essential**: Ensures the extension continues functioning on networks that block or throttle Cloudflare; HTTPBin is an open-source, widely-available HTTP testing service
- **Data Handling**: No personal data transmitted; only standard HTTP download requests
- **User Benefit**: Maintains reliable speed measurements across restricted or corporate network environments

### 4. `https://upload.wikimedia.org/*`
**Justification**: Legacy fallback (retained for compatibility)
- **Specific Use**: Was previously used as a tertiary fallback download source; retained in host permissions for backward compatibility
- **Data Handling**: Standard HTTP image requests with no user tracking
- **User Benefit**: Provides an additional safety net on networks with unusual routing

---

## Standard Permissions

### `activeTab`
- **Use**: Allows the extension popup to open and interact with the current tab context
- **Scope**: Only active when the user explicitly opens the extension popup
- **No Data Access**: Does not read tab content, URLs, or browsing history

### `background`
- **Use**: Runs a persistent service worker that performs speed tests on a schedule and keeps the badge text up to date
- **Scope**: Executes speed tests periodically in the background; no interaction with page content
- **No Data Access**: Only performs outbound network requests to the speed testing endpoints listed above

### `storage`
**Justification**: Persisting speed data and user preferences
- **Specific Use**:
  - `speedData` — stores the latest speed, upload, ping, jitter, and history so the page overlay (content script) can read live results without requiring an open popup
  - `overlayEnabled` — persists the user's choice to show or hide the floating speed badge across browser sessions
  - `installTime` — records the timestamp of first installation to enforce the 10-minute delay before showing the rating prompt, preventing it from appearing immediately on install
- **Why Essential**: The content script runs in a separate context from the background service worker and cannot share in-memory variables directly; `chrome.storage.local` is the standard bridge for this cross-context communication in Manifest V3 extensions
- **Data Handling**: All data is stored locally on the user's device only; nothing is synced to remote servers or shared with third parties
- **User Benefit**: Enables the floating overlay to show live speed on every page; remembers overlay preference between sessions; ensures the rating prompt is shown only after meaningful usage

### `tabs`
**Justification**: Propagating overlay toggle changes to open tabs
- **Specific Use**: When the user toggles the "Speed badge on every tab" switch in the popup, `chrome.tabs.query({})` retrieves all open tabs and `chrome.tabs.sendMessage()` sends an `overlayToggled` message to each content script so the overlay appears or disappears immediately without requiring a page reload
- **Why Essential**: Without this permission, toggling the overlay off would leave it visible on already-open tabs until each page was manually refreshed; the permission allows instant, consistent behaviour across all tabs
- **Scope**: Only reads tab IDs for message routing; does not access tab URLs, titles, or any page content
- **Data Handling**: No tab data is stored, transmitted, or logged; IDs are used solely to deliver the toggle message and are discarded immediately after
- **User Benefit**: Toggling the overlay on or off takes effect instantly across every open tab without any page refreshes

---

## Privacy Commitment

- **No Personal Data Collection**: The extension does not collect, store, or transmit any personally identifiable information
- **Local Processing Only**: All speed calculations are performed locally on the user's device
- **No User Tracking**: No analytics, tracking scripts, or behavioural monitoring of any kind
- **Transparent Operation**: Every network request is made solely for speed testing or latency measurement purposes
- **Minimal Data Retention**: Only the last 10 speed measurements and basic preferences are stored locally in the browser

---

## Chrome Web Store Compliance

This extension fully complies with Chrome Web Store policies:
- **Single Purpose**: Exclusively measures internet connection speed and displays results
- **Limited Data Use**: Only requests permissions and accesses data strictly necessary for core functionality
- **Transparent Permissions**: Every permission is directly tied to a specific, documented feature
- **User Control**: Users can disable the page overlay at any time via the popup toggle
- **No Deceptive Practices**: Honest, straightforward functionality with no hidden behaviour

---

## Technical Implementation Summary

| Feature | Permissions Used |
|---|---|
| Download speed measurement | `speed.cloudflare.com` |
| Upload speed measurement | `speed.cloudflare.com` |
| Ping / jitter measurement | `google.com`, `speed.cloudflare.com` |
| Fallback speed tests | `httpbin.org` |
| Page overlay (floating badge) | `storage`, `tabs`, `activeTab` |
| Overlay toggle persistence | `storage` |
| Rating prompt timing gate | `storage` |
| Background scheduling & badge | `background` |
