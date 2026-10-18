# drop. — GitHub + Vercel frontend + Render signalling backend

This project uses WebRTC DataChannels for peer-to-peer file transfer. GitHub stores the source code, Vercel serves the static frontend, and a small Node.js service on Render handles WebSocket signalling and the `/ice-config` endpoint.

## Important architecture note

Do **not** deploy only the static `public/` folder to Vercel and expect room codes to work by themselves. The browser also needs a reachable WebSocket signalling server. The current Node server keeps room membership in process memory, so run one backend instance (no horizontal scaling) unless you add a shared room store/pub-sub layer. Vercel's current WebSocket features still require careful shared-state architecture for multi-instance realtime rooms.

## What is and isn't stored

- Files are intended to travel over an encrypted WebRTC DataChannel between browsers when a direct connection succeeds.
- The signalling server exchanges room and connection metadata; it does not accept file-upload HTTP requests or save file contents to disk.
- Browsers temporarily hold files/chunks in memory for transfer and session-history sharing. Refreshing/closing a browser can lose that copy.
- If a TURN relay is configured and used, file traffic passes through that relay while the connection is active. TURN is not permanent file storage by itself.
- This prototype is not security-audited and should not be marketed as vulnerability-free.

## 1. Put the code on GitHub

1. Download/extract the project ZIP.
2. Create a new GitHub repository, e.g. `drop`.
3. Upload the **contents of the `drop-app` folder** to the repository root (so `server.js`, `package.json`, `public/`, and `vercel.json` are at the top level).
4. Do not commit `.env` files, TURN credentials, or secrets. `.gitignore` excludes them.

## 2. Deploy the signalling backend on Render

1. Sign in to Render and choose **New → Web Service**.
2. Connect the GitHub `drop` repository.
3. Set **Runtime** to Node, **Build Command** to `npm install`, and **Start Command** to `npm start`.
4. Deploy. Render gives you a backend URL like `https://drop-backend.onrender.com`.
5. In Render → service → **Environment**, add:
   - `ALLOWED_ORIGIN` = your exact Vercel origin once you have it, e.g. `https://drop.vercel.app` (no trailing slash)
   - Optional `TURN_URLS`, `TURN_USERNAME`, `TURN_CREDENTIAL` from your TURN provider
6. Keep the service to a **single instance** for this in-memory prototype. Multiple instances won't share room codes/presence.

## 3. Point the Vercel frontend to the backend

1. In Vercel, choose **Add New → Project** and import the same GitHub repository.
2. In project settings, set the framework preset to **Other** and output directory to `public`. No build step is required.
3. Deploy. Vercel gives you an origin like `https://drop-abc123.vercel.app`.
4. Open `public/config.js` in GitHub and set the backend URL exactly, e.g.:

   ```js
   window.DROP_CONFIG = { signalingUrl: 'https://drop-backend.onrender.com' };
   ```

   Use the backend's HTTPS URL, not `wss://`; the app converts it to `wss://` for WebSockets.
5. Commit that change. Vercel will redeploy automatically.
6. Back in Render, set `ALLOWED_ORIGIN` to the exact Vercel origin, without a trailing slash. Save and redeploy/restart the backend.

If you add a custom domain later, include that exact origin in `ALLOWED_ORIGIN` too (comma-separated origins are supported).

## 4. Test before sharing

1. Open the Vercel URL on your laptop and phone.
2. Create a room on one device and join with the six-character code on the other.
3. Send one small non-sensitive file first.
4. Confirm the recipient can download and open it.
5. Test leaving/rejoining, then try multiple files and a larger file.
6. Check browser DevTools Console and the Render logs if the socket fails.

A Vercel static page without the backend configured will not be able to create/join rooms. A Render service that is asleep/cold-starting may take a little time to wake.

## 5. TURN for restrictive networks

STUN-only connectivity cannot work on every carrier, corporate, or restrictive network. If a browser reports that the network requires a relay, get TURN credentials from a trusted provider and set all three `TURN_*` variables in Render. Never put TURN credentials in `public/config.js` or frontend code. TURN improves connectivity, but relayed transfers are no longer direct end-to-end network paths (WebRTC media/data remains encrypted in transit).

## Local development

```bash
npm install
npm start
```

Open `http://localhost:3000`. For cross-device testing over ngrok, expose port 3000. Do not use the ngrok URL as the permanent production URL.

## Current limits and cautions

- 6-character case-insensitive alphanumeric room codes; 15-minute room lifetime; max 12 peers per room.
- 50 MB maximum per file; browser memory and connection conditions affect performance.
- No resumable/chunk-persistence storage. If a tab closes or a peer disconnects, a transfer may need to restart.
- This is an experimental prototype. Basic input checks and rate limits are present, but production use still requires real multi-device tests, dependency updates, abuse monitoring, TURN configuration, and a security review.
