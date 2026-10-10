<div align="center">

# @three-ws/portal

<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/three-ws-portal?style=flat&logo=github)](https://github.com/nirholas/three-ws-portal/stargazers) [![License](https://img.shields.io/github/license/nirholas/three-ws-portal?style=flat)](https://github.com/nirholas/three-ws-portal/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/three-ws-portal?style=flat)](https://github.com/nirholas/three-ws-portal/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/three-ws-portal/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/three-ws-portal/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->


**Turn any website into a walkable 3D world.**

Paste an address; get a place. Every section becomes a building sized by what it
says, every link becomes a door you can step through, every image becomes a
billboard, every code block becomes a monolith. Walk it in the browser, download
it as a `.glb`, embed it in your own page, or hand it to an agent over MCP.

[Quick start](#quick-start) · [How a page becomes a place](#how-a-page-becomes-a-place) · [API](#api) · [CLI](#cli) · [MCP](#mcp) · [Manners](#manners)

</div>

---

## Why

Every tool that "visualizes a website" draws a node graph. A graph tells you a
site has 40 pages; it does not tell you which section carries the weight, where
the dead ends are, or what it feels like to arrive. A **place** does, because
size, distance and direction are things people read without being taught.

Portal turns the page you already have into that place, deterministically, with
no model in the loop: the same URL always builds the same city, so a link you
share opens the world you walked.

## Quick start

```bash
npm install @three-ws/portal
```

```js
import { fetchWorld, describeWorld } from '@three-ws/portal';

const { world } = await fetchWorld('example.com');
console.log(describeWorld(world));
// { host: 'example.com', sections: 2, doors: 1, tallest: { name: 'Example Domain', … } }
```

Put a live, walkable world on your own page:

```js
import { mountPortal } from '@three-ws/portal';

mountPortal(document.querySelector('#stage'), { url: 'example.com', height: 560 });
```

Or without any JavaScript at all:

```html
<iframe
	src="https://three.ws/portal?url=example.com&embed=1"
	width="100%" height="520" style="border:0;border-radius:14px"
	loading="lazy" title="Walk example.com in 3D"></iframe>
```

No key, no account, no wallet. `fetchWorld` reads the public three.ws Portal API,
which fetches the page once and caches the result for an hour.

## How a page becomes a place

The mapping is fixed, documented, and testable, which is what makes a world
predictable enough to share:

| On the page | In the world |
| --- | --- |
| The page itself | A plaza with an obelisk carrying the title |
| Each `h1`/`h2`/`h3` section | A district with one building |
| How many words a section has | The building's height, on a log scale |
| How many blocks it holds | The building's footprint |
| Each link | A door on the building's wall. Same-site doors rebuild the world where they point; outside links open the page |
| Each image | A billboard beside the building, showing the real image |
| Each code block | A monolith: a tall, dark, emissive slab |
| `theme-color`, or the host name | The palette, so a site looks like itself |

Layout is a pure function with a seeded PRNG. Call it yourself on any outline:

```js
import { buildWorld, collidersFor } from '@three-ws/portal/layout';

const world = buildWorld(outline);       // no network, no DOM, no three.js
const solids = collidersFor(world);      // circles, for your own player controller
```

## API

| Export | What it does |
| --- | --- |
| `fetchWorld(url, opts?)` | Build (or read from cache) `{ world, outline, cached, stale }` |
| `fetchOutline(url, opts?)` | The structural read only: headings, weights, links, images |
| `fetchWorldGlb(url, opts?)` | The world as a glTF binary (`ArrayBuffer`) |
| `mountPortal(el, { url, height? })` | Mount a live walkable world into an element |
| `embedSnippet(url, opts?)` | The same thing as an HTML string |
| `describeWorld(world)` | A compact summary: districts, doors, tallest section |
| `buildWorld(outline, opts?)` | The pure layout function |
| `collidersFor(world)` | Collision circles for the solids in a world |
| `paletteFor(outline)` | The deterministic palette for a site |
| `PortalError` | Every failure, carrying the API's own `code` |

`opts` accepts `endpoint` (default `https://three.ws/api/portal`), `fetch` (bring
your own), and `signal`.

Errors are typed rather than thrown as strings:

```js
import { fetchWorld, PortalError } from '@three-ws/portal';

try {
	await fetchWorld('example.com/private');
} catch (err) {
	if (err instanceof PortalError && err.code === 'robots_disallowed') {
		console.log('That site asks crawlers to stay out, so Portal does.');
	}
}
```

Codes you can branch on: `invalid_url`, `robots_disallowed`, `blocked_host`,
`not_html`, `no_structure`, `too_large`, `unreachable`, `upstream_status`,
`rate_limited`.

## CLI

```bash
npx @three-ws/portal three.ws                    # the world, summarized
npx @three-ws/portal three.ws --glb site.glb     # save it as a 3D file
npx @three-ws/portal three.ws --json world.json  # save the world document
npx @three-ws/portal three.ws --open             # print the walkable link
```

```
  three.ws
  24 districts · 1563 words · 42 doors (38 internal) · 2 billboards · 0 code monoliths

  ███████████···········    109w  Your 3D agent. In your room.
  █████████·············     66w  Your agents don't just talk. They trade.
  █████·················     20w  One tag. Any site.
```

## MCP

An agent that fetches a page gets a wall of text. Over MCP it can ask for the
page's *shape* instead.

```bash
claude mcp add portal -- npx -y @three-ws/portal portal-mcp
```

| Tool | Returns |
| --- | --- |
| `walk_site` | The world: districts, doors (with targets), billboards, plus a walkable link and a GLB URL |
| `site_shape` | The heading spine with the weight of each section. The cheap read |

Both accept any public URL and honour the same manners as the API.

## Manners

Portal is a crawler that runs on a person's request, and behaves like one:

- **It identifies itself** as `ThreeWSPortalBot/1.0 (+https://three.ws/portal)`.
- **It reads robots.txt first** and honours a `Disallow` for that token with a
  clean `robots_disallowed` error. Rules are matched per RFC 9309: most specific
  user-agent group, longest matching rule, `Allow` wins a tie.
- **It reads a page once.** Worlds are cached for an hour fleet-wide, so a link
  shared with a thousand people is one request to the origin.
- **It is bounded**: 3 MB of HTML, a 12 second deadline, and an SSRF guard that
  re-resolves the host, refuses private address ranges, and pins the socket to
  the address it validated.

## License

Apache-2.0. Built by [three.ws](https://three.ws).

<!-- three.ws:growth -->
## Support the project

If three-ws-portal saves you time, **[star it on GitHub](https://github.com/nirholas/three-ws-portal)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=three-ws-portal%3A%20Turn%20any%20website%20into%20a%20walkable%203D%20world&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fthree-ws-portal) · [Share on Bluesky](https://bsky.app/intent/compose?text=three-ws-portal%3A%20Turn%20any%20website%20into%20a%20walkable%203D%20world%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fthree-ws-portal) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fthree-ws-portal) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fthree-ws-portal&t=three-ws-portal%3A%20Turn%20any%20website%20into%20a%20walkable%203D%20world) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fthree-ws-portal&title=three-ws-portal%3A%20Turn%20any%20website%20into%20a%20walkable%203D%20world)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/three-ws-portal` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/three-ws-portal/issues) or [start a discussion](https://github.com/nirholas/three-ws-portal/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/three-ws-portal)](https://github.com/nirholas/three-ws-portal/graphs/contributors)

## Star history

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/three-ws-portal&type=Date)](https://www.star-history.com/#nirholas/three-ws-portal&Date)

<!-- /three.ws:growth -->

<!-- three.ws:ecosystem -->
## Part of three.ws

[three.ws](https://three.ws) is a platform for 3D AI agents with Solana wallets: avatars, a skill marketplace, x402 payments and more than seventy MCP servers. `@three-ws/portal` is one package from it.

- Documentation site: https://nirholas.github.io/three-ws-portal/
- npm: https://www.npmjs.com/package/@three-ws/portal
- Canonical source: https://github.com/nirholas/three.ws (this repository is a generated mirror of `packages/portal`)
- Agent-readable summary: [llms.txt](./llms.txt) and [AGENTS.md](./AGENTS.md)
- Issues and ideas: https://github.com/nirholas/three-ws-portal/issues
<!-- /three.ws:ecosystem -->
