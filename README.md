# Planning Poker (peer-to-peer, no backend)

A version of the planning poker app with no server component at all. One
person's browser tab acts as the "host" — everyone else connects to it
directly over WebRTC. Votes, names, and reveals never touch a server; they
flow browser-to-browser.

## What "no backend" actually means here

WebRTC still needs a tiny bit of outside help to get two browsers talking:
a *signaling* step (so peers can find each other and exchange connection
details) and often a *TURN* relay (for browsers behind strict NATs/firewalls
that can't connect directly). This app uses PeerJS's free, publicly hosted
broker (`0.peerjs.com`) for both. That means:

- **You don't run or pay for anything.** There's no server to deploy, no
  account to create, no infrastructure of your own.
- **You're trusting a third party's free service for connection setup.**
  PeerJS's broker only ever sees enough metadata to connect two peers (peer
  IDs, ICE candidates) — never your room state, names, or votes, which are
  end-to-end between the browsers. But it's still an external dependency
  you don't control, and if PeerJS's free service has an outage, joining a
  room would fail until it's back.

If you'd rather not depend on that at all, the only way to remove it
entirely is a manual signaling exchange (copy-paste connection codes
between browsers) or running your own signaling server — both real options
if this dependency ever becomes a problem, just clunkier or back to needing
something to host.

## How to use it

Just open `index.html` in a browser — directly from disk, or hosted
anywhere that can serve a static file (GitHub Pages, Cloudflare Pages,
S3 with static hosting, or literally just emailing the file). There's no
build step.

One person clicks "Create a new room," which gives them a room code/link.
Everyone else opens the same file (or visits the same hosted URL) and joins
with that code. Anyone in the room can reveal or reset votes.

**The catch:** since the host's browser tab is what holds the room
together, the room only exists while that tab stays open. If the host
closes their tab or loses their connection, the room ends for everyone
else too. For a quick standup-style estimation session this is rarely an
issue — for an "always-on" room you'd return to over days, the small
Cloudflare Worker version (with real server-side state) is the better fit.

## Reconnecting after a refresh or dropped connection

If your connection drops (closed laptop, flaky wifi) and you rejoin with
the same name, you reclaim your seat — your existing vote, if any, is
preserved rather than wiped. This is deliberate: the alternative (treating
every reconnect as a brand-new person) would mean a single network hiccup
mid-vote forces you to vote again and clutters the roster with duplicates.

The trade-off: there's no real identity system here beyond a typed name, so
if two different people on your team happen to type the exact same name,
the second one to connect will take over the first one's spot rather than
appearing as a separate row. For a small team this is unlikely to matter,
but it's worth knowing about if your team has, say, two Tylers.

## Deploying via GitHub Pages + GitHub Actions

This repo includes `.github/workflows/deploy-pages.yml`, which deploys
`index.html` to GitHub Pages automatically on every push to `main`. To turn
this on (one-time setup, a few clicks, not part of the workflow file):

1. Push this repo to GitHub if you haven't already.
2. In the repo, go to **Settings → Pages**.
3. Under **Build and deployment → Source**, choose **GitHub Actions**
   (not "Deploy from a branch" — that's the older method and doesn't use
   this workflow).
4. Push any commit to `main` (or go to the **Actions** tab and run the
   "Deploy to GitHub Pages" workflow manually via **Run workflow**).
5. The Actions tab will show the deploy running; once it finishes, the
   **Settings → Pages** page shows your live URL, something like
   `https://<your-username>.github.io/<repo-name>/`.

From then on, every push to `main` redeploys automatically — no further
manual steps. The workflow only ever publishes `index.html` itself (plus a
`.nojekyll` marker file); the `test/` folder and this README stay out of
the deployed site.



The `test/` folder has a Playwright-driven end-to-end test that exercises
the real app over a real (locally-run, not the cloud one) PeerJS signaling
server — creating a room, joining it, voting, revealing, resetting, and
disconnecting/reconnecting. It's a verification harness, not part of the
shipped app; the app itself has zero dependencies.

```bash
cd test
npm install
node peerserver.js &       # local stand-in signaling server
node staticserver.js &     # serves the test copy of the page
node run-test.js           # drives two real browser contexts through the full flow
```

`build-test-page.js` regenerates `test/index-test.html` from the canonical
`../index.html` whenever you change the app, swapping in the local
signaling server config so tests don't depend on network access to
`0.peerjs.com`.
