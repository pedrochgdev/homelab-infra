# Display

## Summary

`display` is a real-time statistics dashboard for the personal workstation,
served from the `monitor` VM and viewed on the LAN. It is a custom single-page
application, not part of the Prometheus and Grafana stack, even though it runs on
the same host.

It shows live workstation telemetry over a full-screen background image or video,
and is intended to be left running on a secondary screen.

## Host

- Node: `monitor` (`192.168.1.50`)
- Served by: nginx on port `80`
- Document root: `/var/www/display`
- Internal hostname: `display.home.arpa`, via the reverse proxy on `rpi-01`

## Composition

| Path | Purpose |
|---|---|
| `/var/www/display/index.html` | The entire application: markup, styles and logic in one file, around 26 KB |
| `/var/www/display/backgrounds/` | Background images and video, listed over HTTP and rotated by the page |

## Data Source

The page polls a **custom metrics agent running on the workstation**:

```
http://192.168.1.109:9500/metrics
```

Requests carry a 4 second timeout, with a shorter reachability probe against the
agent root. The page degrades on its own when the agent does not answer.

This agent is distinct from the exporters that Prometheus scrapes on the same
machine — `windows_exporter` on `9182` and the NVIDIA exporter on `9835`. It is a
separate service on a separate port, written for this dashboard.

## Dependencies and Fragility

The dashboard has three dependencies that are worth stating plainly, because none
of them is currently monitored:

**The agent on port 9500 is unmonitored.** Nothing scrapes it and nothing alerts
on it. If the agent stops, the page keeps loading and simply shows no data. The
failure is silent by design, which is fine for a dashboard but means the agent's
health is invisible.

**The workstation address is not reserved.** `192.168.1.109` sits inside the DHCP
pool (`.100` to `.199`). The address is hardcoded in the page, so a changed lease
breaks the dashboard. The same address is hardcoded in the Prometheus scrape
configuration and in the reverse proxy on `rpi-01`, so a single lease change
breaks three things at once.

**Fonts load from the internet.** The page pulls typefaces from Google Fonts, so
a LAN-only dashboard depends on WAN availability for its appearance. Self-hosting
the fonts under `/var/www/display` would remove that dependency.

## Operational Notes

- the page sets aggressive no-cache headers, so it always fetches fresh markup
- content is authored directly in `index.html`; there is no build step
- background media is uploaded to `backgrounds/` over SSH
- the whole thing lives only on `monitor` and is not versioned anywhere

That last point is the most consequential: `index.html` represents real authoring
effort and exists in exactly one place. It belongs in a backup tier, or in this
repository. See [`docs/runbooks/backups.md`](../runbooks/backups.md).
