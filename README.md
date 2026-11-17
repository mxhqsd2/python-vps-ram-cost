# Python VPS hosting: how much RAM does your app need, what does a year actually cost, and which plan should you buy? (Full plan table, from $10.60/year)

RackNerd's smallest annual KVM plan is **$10.60/year** for 1 GB RAM, 1 vCore, 25 GB SSD and 2 TB of transfer. That covers a single FastAPI or Flask service, a Telegram bot, a webhook receiver, or a nightly scraper. Django with Postgres and a Celery worker wants the **4 GB tier at $29.98/year**. Everything below is about choosing between those two ends: what each tier actually holds, how to get a Python app running on a bare KVM box, and how the annual price compares with the hourly-billed hosts you probably have open in another tab.

> **Quick picks.** One endpoint or a bot → 1 GB at $10.60/year. Flask or FastAPI with a database on the same box → 2.5 GB at $18.66/year. Django with Postgres, Redis or Celery → 4 GB at $29.98/year. Jupyter, pandas or a local model → 8 GB at $62.49/year, which also brings 6 vCores and 20 TB of transfer.
> 👉 [See the current RackNerd VPS specials and live pricing](https://bit.ly/RacKnerd)

## The full RackNerd VPS specials lineup

| Plan | RAM / vCPU | SSD / Transfer | Price | Order |
| --- | --- | --- | --- | --- |
| 1 GB | 1 GB / 1 vCore | 25 GB RAID-10 SSD / 2 TB @ 1 Gbps | **$10.60/year** | [ Order 1 GB](https://bit.ly/RacKnerd) |
| 2.5 GB | 2.5 GB / 2 vCore | 45 GB RAID-10 SSD / 3 TB @ 1 Gbps | $18.66/year | [ Order 2.5 GB](https://bit.ly/RacKnerd) |
| 4 GB | 4 GB / 3 vCore | 65 GB RAID-10 SSD / 6.5 TB @ 1 Gbps | $29.98/year | [ Order 4 GB](https://bit.ly/RacKnerd) |
| 6 GB | 6 GB / 5 vCore | 100 GB RAID-10 SSD / 10 TB @ 1 Gbps | $44.98/year | [ Order 6 GB](https://bit.ly/RacKnerd) |
| 8 GB | 8 GB / 6 vCore | 150 GB RAID-10 SSD / 20 TB @ 1 Gbps | $62.49/year | [ Order 8 GB](https://bit.ly/RacKnerd) |

These are promo-priced annual plans, and promo inventory rotates as stock moves. Confirm the live price for the tier you want in the cart before paying. If a tier is sold out, the regular specials store still lists a 1 GB plan at $21.99/year and a 4 GB plan at $59.99/year, so the promo window is worth several times the effort of waiting for it.

What every one of those plans includes, in RackNerd's own description:

- KVM virtualization with full root SSH access, unmanaged
- 1 Gbps port on every plan
- Choice of CentOS, Rocky Linux, AlmaLinux, Fedora, Debian or Ubuntu at order time; custom ISO installs on request through a support ticket
- The ctrl panel: reboot, OS reinstall, HTML5 VNC console, root password reset, inbound rules via a network firewall, one-click apps, and upgrades
- Provisioned right after payment, deployable in multiple datacenter locations
- 24/7 support for the platform itself

There is no managed tier and no control panel like cPanel bundled in. You get a root shell and a clean OS.

## How much RAM does a Python app actually need?

A Python web app stays resident. Gunicorn or Uvicorn holds the interpreter and every imported module in memory whether or not requests are arriving, and each Gunicorn worker is a separate process carrying its own copy of that footprint. Four workers on a box that idles fine with one will cost roughly four times the per-worker memory before Postgres or Redis enter the picture. That is the reason a PHP site of the same traffic profile fits in less RAM.

RackNerd publishes RAM floors for its one-click apps, and those numbers are the most useful sizing data available for Python workloads:

- **n8n, Coolify:** comfortable from 2 GB
- **Ollama, LibreChat, AnythingLLM, Hermes Agent:** at least 4 GB
- **OpenClaw:** at least 8 GB

Those are vendor minimums, not benchmarks. Mapped onto the plans:

**1 GB.** One Python process, low request volume, no heavy database on the same box (SQLite is fine). Telegram bots long-polling the API, cron scrapers, webhook receivers, a single Flask endpoint. This is the tier where the annual price is so low that the decision costs less than the coffee you'd drink while deciding.

**2.5 GB.** Flask or FastAPI plus Postgres plus Nginx on one machine, with room for a couple of workers. Small production sites, internal tools, side projects that real users touch. n8n and Coolify clear their 2 GB floor here.

**4 GB.** Django with Postgres and a Celery worker backed by Redis, multi-worker Gunicorn setups, or a CPU inference endpoint wrapping scikit-learn or a small transformers model. Anything on RackNerd's one-click list that asks for 4 GB lands here.

**6 GB and 8 GB.** pandas and numpy jobs, Jupyter, larger scraped datasets, model files sitting on local disk. The 8 GB plan gives 6 vCores and 150 GB of SSD, which is the configuration that stops a data job from swapping.

Starting at 1 GB and upgrading later is a real option, since ctrl handles upgrades in place. Starting at 1 GB and staying there while you add Postgres, Redis and four workers is how people end up blaming the host for an OOM kill.

## Putting a Python app on a fresh box: eight steps

This is the same sequence on any Ubuntu KVM VPS. RackNerd gives you root from the first minute, which is all it assumes.

1. `ssh root@your.ip`, then `apt update && apt upgrade -y` to patch the base system before it faces the internet.
2. `apt install python3 python3-pip python3-venv`. Ubuntu 22.04 and 24.04 ship a current Python 3 in the repos. If you need an exact minor version, install pyenv and manage several side by side.
3. Create one virtualenv per project: `python3 -m venv /opt/myapp/venv`. Never `pip install` into the system Python.
4. Activate it and `pip install -r requirements.txt`. numpy and pandas usually arrive as prebuilt wheels, so no compiler is needed unless you build from source.
5. Install the app server. Gunicorn for WSGI (Flask, most Django), `uvicorn[standard]` for ASGI (FastAPI, Django Channels, anything with WebSockets). Bind it to `127.0.0.1` or a Unix socket, never a public interface.
6. Write a systemd unit in `/etc/systemd/system/` that launches your venv's Gunicorn or Uvicorn with `Restart=always`, then `systemctl enable && systemctl start`. This is what survives reboots and crashes.
7. Put Nginx in front as a reverse proxy. Nginx terminates TLS, serves static files and buffers slow clients, none of which you want your Python process doing.
8. `ufw allow 22,80,443/tcp && ufw enable`, point DNS at the box, then run Certbot for a free Let's Encrypt certificate with automatic renewal.

Background jobs go on cron or systemd timers. Celery needs Redis or RabbitMQ running as separate services, which is a memory line item, not a footnote.

## What a year actually costs against Cloudzy, LightNode and OVHcloud

Monthly pricing hides the comparison. Here is the first-year cost of the cheapest plan each provider lists, at the specs published on their own sites.

| Provider | Entry plan | Advertised monthly | First 12 months |
| --- | --- | --- | --- |
| RackNerd | 1 GB / 1 vCore / 25 GB SSD | $0.88 effective | **$10.60** (prepaid annually) |
| Cloudzy | 1 GB / 1 vCPU / 25 GB NVMe | $3.48 promo, $6.95 list | $41.76 promo, $83.40 list |
| LightNode | 2 GB / 1 vCPU / 50 GB NVMe | $7.71 | $92.52 |
| OVHcloud | 4 GB / 2 vCores / 40 GB NVMe | from $4.54 | about $54.48 |

Three things that table does not show.

**Cloudzy's prices are half off.** Its $3.48, $7.48, $14.48 and $26.48 tiers are all 50% discounts against listed prices of $6.95, $14.95, $28.95 and $52.95. When the promotion ends, the 1 GB plan costs $83.40 a year, roughly eight times the RackNerd special. Cloudzy does give you hourly, monthly or annual billing, a 14-day money-back window, a 99.95% uptime SLA and 13 regions, and for a workload you spin up for a week and delete, that flexibility is worth more than the annual discount.

**LightNode's storage does not scale.** All four plans, from the $7.71 Start to the $52.7 Enterprise, ship 50 GB of NVMe. You get more RAM, cores and transfer as you climb, but never more disk. For a scraper with a growing dataset or a directory of model files, that ceiling decides the purchase long before the price does. RackNerd's tiers run 25 GB to 150 GB, which is the same money buying you room to store things.

**OVHcloud is the closest per-year competitor at the 4 GB mark.** Its VPS-1 lists 4 GB RAM, 2 vCores, 40 GB NVMe and unlimited traffic from $4.54/month, about $54.48 a year, against $29.98 for the RackNerd 4 GB plan with 3 vCores, 65 GB SSD and 6.5 TB of transfer. Two caveats on OVHcloud: the "from" price moves with datacenter and contract length, and the page doesn't publish renewal pricing, so budget from the cart rather than the headline.

Two more worth knowing for completeness. HostPapa advertises "plans as low as $2.95 per month" with a footnote that doesn't tie the number to a specific plan, so treat it as a marketing floor rather than a Python VPS rate; its 30-day refund and 99.9% uptime claim are the solid parts. VyomCloud starts at ₹349–360/month in Indian datacenters with pay-as-you-use billing, a 99.5% SLA and a 7-day refund window, which suits latency-sensitive users in South Asia and nobody else.

## Billing, refunds and the parts the sales page doesn't stress

The specials are prepaid annually. You pay for twelve months on day one, and the per-month figures above are arithmetic, not a billing option. Cloudzy and LightNode let you stop paying when the project dies; RackNerd does not. Run the number for how long you'll actually keep the thing.

RackNerd's specials page doesn't publish a money-back window. Cloudzy offers 14 days, HostPapa 30 days excluding domain fees, VyomCloud 7 days. If a trial period is a hard requirement, ask support before you pay rather than assuming one exists.

Prefer monthly billing? RackNerd's regular KVM line bills month to month, and the 12 GB tier there is listed at $55.99/month with no annual commitment. Per gigabyte it costs far more than the specials, which is the trade for not prepaying.

Want a Python-focused box without prepaying a year? 👉 [Check which RackNerd tiers are in stock and what they cost today](https://bit.ly/RacKnerd)

## FAQ

**Can I run Django, Flask or FastAPI on these?** Yes. Full root, any Python version via distro packages or pyenv, Gunicorn or Uvicorn, Nginx in front. Nothing on the box restricts the stack.

**Is it managed?** No. Support covers the platform: power, network, provisioning, the ctrl panel. Application bugs, dependency conflicts and Nginx configs are yours.

**Can I pick a specific Python version?** The OS images ship whatever their repos carry. For an exact minor version, install pyenv. Custom ISO installs are available through a support ticket.

**How fast is setup?** Provisioned immediately after payment, so you can be SSH'd in within minutes rather than waiting on manual approval.

**Can I run Ollama or a local LLM?** RackNerd lists Ollama at a 4 GB minimum, so the 4 GB plan is the floor and the 8 GB plan is the sane choice. There's no GPU here, so this is CPU inference.

**What happens when I outgrow the plan?** Upgrades run through ctrl, in place. Moving from 2.5 GB to 4 GB doesn't mean rebuilding the server.

## Pick by what you're running

A single FastAPI or Flask endpoint, a bot, a cron scraper, a webhook receiver → **1 GB at $10.60/year**, and there is no reason to spend more.

Flask or FastAPI with Postgres on the same machine, or a small internal tool with real users → **2.5 GB at $18.66/year**.

Django with Postgres plus Celery or Redis, a multi-worker setup, or a CPU inference API → **4 GB at $29.98/year**.

pandas, numpy, Jupyter, a growing dataset, or a one-click app that asks for 8 GB → **the 8 GB plan at $62.49/year**, where 6 vCores and 150 GB of SSD do the actual work.

Two things to confirm in the cart before you pay: the renewal price on the tier you picked, and that it's the RAM figure you planned for rather than the plan name you remembered.

👉 [Start with the 1 GB special at $10.60/year, or size up from there](https://bit.ly/RacKnerd)
