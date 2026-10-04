# puppeteer rotating proxy: a working 9Proxy setup, plus fixes for 407 errors, sticky sessions and per-browser rotation

Your script worked yesterday. Today it returns a challenge page, or a 407, or a login wall with a 200 status code, and you're sitting there wondering whether the problem is the proxy, the fingerprint, or your rotation logic. Usually it's the third one.

Puppeteer doesn't have a "rotating proxy" feature. It has a launch argument that accepts one proxy per browser instance, and that's it. Everything people describe as rotation in Puppeteer is really one of three things: a provider-managed rotating gateway, a local relay like proxy-chain, or you launching multiple browser processes and spreading the work across them. Pick the wrong one for your workload and you'll spend a weekend debugging credentials instead of collecting data.

This guide covers how each approach actually behaves with Puppeteer, how 9Proxy's rotating endpoint is structured (it's configured inside the username, which matters for automation), and what the current 9Proxy packages cost if you're choosing a provider for this.

## Why Puppeteer makes rotation harder than it looks

Three constraints cause most of the pain, and none of them are documented loudly enough.

**Chrome ignores credentials inside `--proxy-server`.** Passing `--proxy-server=http://user:pass@host:port` looks reasonable and fails silently. No helpful error, just `407 Proxy Authentication Required` a few lines later. You need either `page.authenticate()` or a local relay.

**There is no per-page proxy switch.** Playwright lets you set a proxy per browser context. Puppeteer doesn't. If you want ten different exit IPs running concurrently, that's ten browser instances, or one rotating endpoint that hands you a different IP per request. Request interception won't save you here — `setRequestInterception()` is for filtering and modifying requests, not for changing the network path mid-flight.

**IP rotation alone doesn't fix detection.** A residential IP paired with a headless Chrome that claims Windows while reporting a mismatched timezone and locale is still suspicious. Rotation reduces one signal; the rest of the fingerprint is your job.

## How 9Proxy's rotating endpoint is structured

9Proxy runs two product lines with genuinely different mechanics, and the one you pick changes your Puppeteer code.

**Residential by GB** is the rotation-friendly option. You get a fixed hostname and port from the dashboard, and everything else — country, state, city, ISP, session behaviour — is encoded in the username. It works from the dashboard directly, no desktop app, and you authenticate with username/password or an IP whitelist. Endpoint generation is unlimited; only traffic is metered.

**Residential by IP** gives you a fixed pool of residential IPs with unlimited bandwidth. The IPs stay alive anywhere from a few hours to roughly 24 hours, and unused IPs never expire. The catch: it requires the 9Proxy desktop app, which forwards each IP to a local port on `127.0.0.1`. For Puppeteer that's less of a hassle than it sounds, because local ports sidestep the credential problem entirely — Chrome connects to `http://127.0.0.1:PORT` with nothing to authenticate, unless you deliberately enable proxy authentication in the app. The app also supports an Auto Rotation Proxy that switches the exit at intervals you define on selected ports.

### Rotating mode vs sticky mode

The GB line lets you choose per session, and the distinction is the single most useful setting for automation:

| Mode | Username format | Behaviour | Use it for |
| --- | --- | --- | --- |
| Rotating | `subuser-country-us` | New IP on every request | Scraping, price monitoring, SERP checks |
| Rotating + geo | `subuser-country-us-city-newyork` | New IP per request, filtered to a city | Localised data, geo-verification |
| Sticky | `subuser-country-us-sst-15` | Same IP held for 15 minutes | Logins, carts, multi-step flows |
| Parallel sticky | `subuser-country-us-sst-15-ssid-id1` | One sticky IP per `ssid` | Several concurrent bot sessions |

The `sst` value is minutes. The `ssid` is what lets you run parallel sticky sessions from a single configuration — each unique `ssid` gets its own IP. Without it, multiple workers sharing a username can collide on the same exit, which shows up as session-crossing bugs that are miserable to trace.

One practical tip from the provider's own docs: target by country alone when you can. Adding state, city and ISP narrows the available pool and slows session assignment.

## The setup: Puppeteer plus a rotating gateway

### Step 1: prove the credentials outside the browser

Do this before touching Puppeteer. It takes ten seconds and eliminates half the possible causes.

bash
curl -x your_proxy_host:your_port \
  -U "subuser-country-us:your_password" \
  https://ipinfo.io


If that returns an IP, your credentials are fine and any subsequent failure is a Puppeteer problem. If it returns 407, fix that first.

### Step 2: launch the browser through the gateway

Since 9Proxy's GB gateway uses username/password auth, you need `page.authenticate()`, and you need it before the first navigation. Order matters — reverse it and your initial request goes out unauthenticated.

javascript
const puppeteer = require('puppeteer-extra');
const StealthPlugin = require('puppeteer-extra-plugin-stealth');
puppeteer.use(StealthPlugin());

const HOST = process.env.PROXY_HOST;
const PORT = process.env.PROXY_PORT;
const USER = 'subuser-country-us';   // add -city-... -sst-15 -ssid-... as needed
const PASS = process.env.PROXY_PASS;

(async () => {
  const browser = await puppeteer.launch({
    headless: 'new',
    args: [
      `--proxy-server=http://${HOST}:${PORT}`,
      '--no-sandbox',
    ],
  });

  const page = await browser.newPage();
  await page.authenticate({ username: USER, password: PASS });  // before goto()

  await page.setUserAgent(
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 ' +
    '(KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36'
  );
  await page.setViewport({ width: 1280, height: 800 });

  await page.goto('https://ipinfo.io/json', { waitUntil: 'domcontentloaded' });
  console.log(await page.content());

  await browser.close();
})();


Keep credentials in environment variables. Hardcoding them into a script that eventually lands in a repo is a habit that costs people real money.

### Step 3: multiple identities, one script

When you need several distinct exit IPs running at the same time, Puppeteer gives you no clean option except separate browser instances. That's fine — it's cheap, and it's the approach that survives contact with production.

javascript
const puppeteer = require('puppeteer-core');

const workers = [
  { proxy: 'subuser-country-us-sst-30-ssid-w1' },
  { proxy: 'subuser-country-de-sst-30-ssid-w2' },
  { proxy: 'subuser-country-jp-sst-30-ssid-w3' },
];

async function runWorker({ proxy }) {
  const browser = await puppeteer.launch({
    executablePath: process.env.CHROME_PATH,
    args: [`--proxy-server=http://${process.env.PROXY_HOST}:${process.env.PROXY_PORT}`],
    headless: 'new',
  });
  const page = await browser.newPage();
  await page.authenticate({ username: proxy, password: process.env.PROXY_PASS });
  // your scraping logic here
  await browser.close();
}

await Promise.all(workers.map(runWorker));


If your upstream credential string embeds auth in the URL and you'd rather not restructure it, `proxy-chain` does the anonymising for you:

javascript
const { anonymizeProxy, closeAnonymizedProxy } = require('proxy-chain');
const localProxy = await anonymizeProxy(`http://${USER}:${PASS}@${HOST}:${PORT}`);
// launch Puppeteer with --proxy-server=localProxy ...
await closeAnonymizedProxy(localProxy, true);   // in a finally block


Two traps with `proxy-chain`: version 3 dropped CommonJS, so `require()` throws unless you install `proxy-chain@2`, and every `anonymizeProxy()` call opens a local server on a random port. Skip the cleanup in a long-running job and you'll exhaust file descriptors overnight.

Ready to point a real endpoint at your own targets? 👉 [Grab a 9Proxy balance and generate your first rotating endpoint](https://bit.ly/9-Proxy)

## The full 9Proxy package list

9Proxy prices by balance, not subscription. You buy a package once, the balance sits in your account, and nothing auto-renews. Unused IPs never expire; GB traffic carries a 180-day validity, which becomes unlimited on Enterprise.

Worth knowing before you budget: 9Proxy raised prices on its IP-based and bundle packages on 1 June 2026, its first adjustment since launch. GB-based package prices were explicitly left unchanged. The figures below are the current published rates.

**IP-based packages — unlimited bandwidth per IP**

| Package | Price per IP | Total | Billing | Purchase |
| --- | --- | --- | --- | --- |
| 100 IPs | $0.24 | $24 | One-off, balance | [Buy 100 IPs](https://bit.ly/9-Proxy) |
| 500 IPs | $0.144 | $72 | One-off, balance | [Buy 500 IPs](https://bit.ly/9-Proxy) |
| 1,000 IPs + 500 bonus | $0.084 | $126 | One-off, balance | [Buy the 1,500 IP package](https://bit.ly/9-Proxy) |
| 2,500 IPs | $0.084 | $210 | One-off, balance | [Buy 2,500 IPs](https://bit.ly/9-Proxy) |
| 5,000 IPs | $0.072 | $360 | One-off, balance | [Buy 5,000 IPs](https://bit.ly/9-Proxy) |
| 15,000 IPs | $0.048 | $720 | One-off, balance | [Buy 15,000 IPs](https://bit.ly/9-Proxy) |
| 25,000 IPs | $0.035 | $863 | One-off, balance | [Buy 25,000 IPs](https://bit.ly/9-Proxy) |
| 50,000 IPs | $0.029 | $1,438 | One-off, balance | [Buy 50,000 IPs](https://bit.ly/9-Proxy) |
| 100,000 IPs | $0.023 | $2,300 | One-off, balance | [Buy the 100,000 IP business package](https://bit.ly/9-Proxy) |
| 200,000 IPs | $0.021 | $4,140 | One-off, balance | [Buy the 200,000 IP business package](https://bit.ly/9-Proxy) |
| 500,000 IPs | $0.018 | $8,625 | One-off, balance | [Buy the 500,000 IP business package](https://bit.ly/9-Proxy) |

**GB-based packages — pay per gigabyte, unlimited endpoints**

| Package | Price per GB | Total | Validity | Purchase |
| --- | --- | --- | --- | --- |
| 5 GB | $3.00 | $15 | 180 days | [Buy 5 GB to test](https://bit.ly/9-Proxy) |
| 50 GB + 5 GB bonus | $2.10 | $105 | 180 days | [Buy the 55 GB package](https://bit.ly/9-Proxy) |
| 100 GB | $1.50 | $150 | 180 days | [Buy 100 GB](https://bit.ly/9-Proxy) |
| 200 GB | $1.00 | $200 | 180 days | [Buy 200 GB](https://bit.ly/9-Proxy) |
| 1,000 GB | $0.80 | $800 | 180 days | [Buy 1,000 GB](https://bit.ly/9-Proxy) |
| 2,000 GB | $0.75 | $1,500 | 180 days | [Buy 2,000 GB](https://bit.ly/9-Proxy) |

**Bundle packages — IPs and traffic together**

| Package | What you get | Price | Validity | Purchase |
| --- | --- | --- | --- | --- |
| Starter | 100 IPs + 5 GB | $30 | 180 days on traffic | [Get the Starter bundle](https://bit.ly/9-Proxy) |
| Popular | 1,500 IPs + 50 GB | $180 | 180 days on traffic | [Get the Popular bundle](https://bit.ly/9-Proxy) |
| Pro | 5,000 IPs + 500 GB | $720 | 180 days on traffic | [Get the Pro bundle](https://bit.ly/9-Proxy) |

**Enterprise** is quoted per account rather than listed as a fixed tier. What's documented: unlimited data validity, team mode with one owner and up to five members, per-member traffic controls and shared bandwidth that doesn't expire between members, full activity logs, and unlimited share-code creation. 👉 [Talk to 9Proxy about an Enterprise plan](https://bit.ly/9-Proxy)

The advertised network sits at roughly 20 million residential IPs across 90-plus countries, with HTTP, HTTPS and SOCKS5 support. Those are vendor figures, not independently verified numbers — treat pool size claims from any provider as a starting point rather than a guarantee of what you'll get on your specific target.

## Which package actually fits a Puppeteer job

The two billing models solve different problems, and choosing wrong is the most common way to overspend.

**Rotating-heavy scraping belongs on GB.** If your script hits thousands of URLs once each, an IP package is a bad fit — you'd be buying IP activations and burning them on single requests. Calculate your bandwidth instead: if your average page transfer is around 1 MB, that's roughly a thousand page loads per GB, so the entry 5 GB package covers a few thousand pages and 100 GB reaches into the hundred-thousand range. That estimate depends entirely on your target's page weight, so measure one real page before extrapolating.

**Long sessions, logins and account work belong on IP.** Unlimited bandwidth per IP changes the math completely when a single session can pull hundreds of megabytes, or when you need one stable identity for hours. The `sst` sticky mode on the GB line can imitate this, but session duration is capped by what you configure, whereas an IP package gives you roughly 3 to 24 hours of natural lifetime per IP.

**Bundles make sense when you're genuinely doing both** — a client dashboard that needs fixed identities plus a crawler that needs volume. The Starter bundle at $30 is cheap enough to be the sensible first purchase if you're not sure which model you'll end up using.

One more thing worth knowing: 9Proxy's IP-based proxies come with a 60-second replacement policy, so an IP that fails immediately after activation is credited back. Combined with unlimited bandwidth, that's the practical reason small IP packages stay attractive for testing rather than being pure waste.

## What fails, and what to do about it

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `407 Proxy Authentication Required` | Credentials in `--proxy-server`, or `authenticate()` called after `goto()` | Move to `page.authenticate()` and call it first |
| `ERR_NO_SUPPORTED_PROXIES` | Malformed proxy string or wrong protocol prefix | Verify with curl, then check the scheme matches what the provider expects |
| `ERR_TUNNEL_CONNECTION_FAILED` | HTTPS CONNECT issue or dead exit node | Retry — rotating endpoints assign a new node per connection |
| Empty page content with a 200 status | Soft block served as a challenge page | Validate the content, not the status code |
| `403 Forbidden` | Fingerprint mismatch, not IP | Align user agent, viewport, timezone and locale; enable stealth |
| Timeout on a small share of requests | Narrow geo filter shrinking the pool | Loosen targeting: country only, drop state/city/ISP |
| `ERR_REQUIRE_ESM` on startup | `proxy-chain` v3 installed | `npm install proxy-chain@2` |

## Checklist before you blame the provider

- Run the curl test with your exact credential string.
- Confirm `page.authenticate()` executes before the first navigation.
- Verify the exit IP inside the browser, not just in curl — hit `ipinfo.io/json` from within the page and log the result.
- Check that user agent, viewport, timezone and locale tell the same story.
- Add realistic delays. Rotating IPs don't make 50 requests per second from one machine look human.
- Use a separate browser instance per identity, and close it in a `finally` block. Cookies and cache leaking between workers is a silent source of weird behaviour.
- Close `anonymizeProxy` handles if you're using `proxy-chain`.

## FAQ

**Can I assign a different proxy to each page in Puppeteer?**
Not natively. Puppeteer lacks Playwright's per-context proxy option. Use one browser instance per proxy, or a rotating gateway that changes the exit on every request.

**Does 9Proxy support SOCKS5?**
Yes, alongside HTTP and HTTPS. For Puppeteer, SOCKS5 goes in the `--proxy-server` argument, though authentication still needs handling — local ports from the desktop app are the path of least resistance there.

**Do the IP-based and GB-based plans behave the same way in code?**
No. GB-based plans use a remote host with structured usernames and password or whitelist auth. IP-based plans require the desktop app and expose `127.0.0.1` ports, which means your Puppeteer args point at localhost rather than a provider hostname.

**Is there a free trial?**
Third-party listings disagree on this, and the provider's own communications describe trial availability as limited and dependent on current stock. The reliable answer is to ask support. Failing that, the $15 GB package and the $24 IP package are the cheapest ways to verify performance on your own targets, and the 60-second replacement policy covers IPs that die immediately.

**How do I keep a login session alive?**
Use sticky mode with an `sst` value longer than your expected session, and give each concurrent worker its own `ssid`. One browser instance per `ssid`, and the session stays coherent.

## The short version

Puppeteer rotates proxies through one of three mechanisms: a provider gateway that changes the IP per request, a local relay, or multiple browser instances. 9Proxy's GB line covers the first case with rotation and stickiness encoded in the username, and its IP line covers long-lived identity work through local ports that conveniently dodge Chrome's credential limitation. If you're scraping at volume, buy GB and measure your page weight first; if you're running sessions that live for hours, buy IPs. Matching the model to the workload is the difference between a $15 test and a $360 mistake.
