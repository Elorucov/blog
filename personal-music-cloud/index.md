---
title: My personal music cloud, or how I united my music library
summary: ...by using an old laptop for storage and a free VDS for proxying.
published: 2026-04-26
---

## Background

### The Offline Era

Back in the days of internet cafés, I used to visit them with one specific mission: downloading songs I'd heard on the radio or TV — for myself and my sister — onto our phones. We didn't have a computer or home internet back then. Our only connected devices were a pair of *"Chinese"* knock-off phones (shanzhai, to be precise).

Time passed. By 2012, I was getting online via EDGE on my phone — or occasionally through public Wi-Fi — and keeping my music collection on an SD card. A year later, I got a laptop and copied everything over. Over the next couple of years I went through two more phones; new tracks went to the computer first, and only some made it onto the phone. Then, one fine autumn day in 2014, I accidentally wiped my laptop. A large chunk of my data was gone, music included. Whatever had been sitting on my phone at the time survived.

Meanwhile, I'd started listening to music online — mostly on VKontakte. But streaming was a luxury: I was living in rented accommodation with no home internet and no 3G coverage in my area, so I still relied on downloading. VK was mainly a discovery tool for finding tracks I'd then grab locally.

### The Online Era

In the second half of the 2010s I finally got a reliable-enough internet connection and started saving music directly to VK more and more. SoundCloud joined the mix too. Downloads to the laptop became the exception rather than the rule, and on my phone I was listening almost exclusively through VK.

That's when the **desync problem** first appeared. Some tracks existed everywhere; others were only on the laptop, or only in VK and/or SoundCloud. My library was quietly fragmenting.

### The Spotify & YouTube Music Era

In 2021 I decided to try Spotify. What I loved about it was the genre-aware recommendations — listening to something would surface similar tracks, and I kept discovering new artists. Naturally, everything new I found on Spotify went straight into my Spotify library. Including a handful of tracks I already had on the laptop or in VK, which made the desync even worse. SoundCloud got quietly abandoned.

I started thinking about consolidating everything into one place. At the time I was using Spotify's free tier through a browser and was considering going Premium — but in 2025 my Spotify account literally *vanished*. "No account with that email," the app said, yet trying to register with the same email produced a message saying the account *already exists*. Classic. Fortunately, a few months earlier I had migrated my Spotify library to YouTube Music. I hadn't been very impressed with YTM at the time, so I kept using Spotify as my primary app — but after losing the account, I still had roughly 70% of my Spotify library waiting for me in YouTube Music.

### The Self-Hosted Era

From late 2025 I started seriously thinking about a full library overhaul. My goals were:

* Merge all the scattered libraries into one
* Store music in the cloud (my own cloud)
* Prune tracks I no longer listen to

There's also a fundamental problem with streaming services worth mentioning: **rights holders can pull their content at any time**, and when they do, those tracks simply disappear from your library. It's happened to me more than once. When you own your files, your favorites are yours to keep — forever.

My first thought was storing everything on a VDS. I already have a free Oracle VDS, but the storage is tiny, and it's already doing other things. Renting another VDS wasn't an option at the time for financial reasons.

But I *do* have an old laptop sitting around doing nothing. I figured I could turn it into a private cloud storage server for the family — but decided to start smaller: use it as a personal music server first and see how it goes. Here's what came out of that experiment.

---

## Architecture

### Choosing a Media System

Through conversations with like-minded people, I learned that some of them use [Jellyfin](https://jellyfin.org). I did some research into the alternatives and initially landed on [Navidrome](https://www.navidrome.org). I installed it on my main laptop and started testing client apps — but ultimately Navidrome didn't work for me. The web UI had a full song list, but the API (and therefore all third-party apps) didn't expose one. You could browse all albums and all artists, but not all songs. That's a dealbreaker for me, since I refuse to use web UIs if I can help it.

So I installed Jellyfin. It works well enough overall. The one pain point is the Android client situation — there are a few options, but each has its own issues:

- **The official app** is just a web view wrapper
- **One third-party app** is built in React Native, noticeably sluggish — and couldn't even play music properly
- **Another** is in Flutter — didn't even bother installing it
- **A third** uses Jetpack Compose with a "pseudo-native" feel, but after installing I discovered it only supports video content and has zero music library support

Eventually I found a [fork](https://github.com/adrianvic/jamfish) of a long-abandoned alternative client written in Java (rare these days, but honestly I kind of like it). It has its own quirks — the song list shows the track name and album but no artist — but it's fixable. I know a bit of Java, so I might take a crack at it eventually.

Jellyfin's API is solid, and I'm planning to write a native desktop client for it at some point (yes, I'm serious about avoiding web UIs). In the end, Jellyfin is what I went with.

### Exposing the Storage to the Outside World

The next problem: how do I make my old laptop (let's call it the *storage server* from here on) accessible from the internet? Ideally with its own domain.

My plan: the storage server connects to a VPN running on my VDS. On the VDS I run **nginx** as a reverse proxy, forwarding incoming requests to Jellyfin on the storage server.

I initially chose **WireGuard** as the VPN solution. A quick disclaimer: I had no prior experience setting up a VPN for device-to-device communication (years ago I set up Outline VPN to bypass geo-blocks before a trip to Russia — that's the extent of it). No matter what I tried, I couldn't get the VDS and the storage server to see each other, even though the client was successfully connecting to the VPN. The culprit turned out to be [CGNAT](https://en.wikipedia.org/wiki/CGNAT) — my storage server sits behind it, which means a traditional VPN won't work. I needed a mesh VPN instead.

That's how I discovered [Tailscale](https://tailscale.com/). I deployed [Headscale](https://headscale.net/) (a self-hosted, open-source implementation of the Tailscale control server) on my VDS, installed Tailscale clients on all my devices — including the storage server and the VDS itself — and got everything talking to each other. As a bonus, I can now RDP into my main laptop from my phone without being on the same local network.

All that remained was configuring nginx — which we'll cover in the next section.

---

## Installation & Configuration

### The Storage Server

You can use pretty much any machine for this — even a Raspberry Pi (preferably a recent, beefier model). In my case it's a 2012 laptop: Intel Core i3, 8 GB RAM, 500 GB HDD. Not exactly a powerhouse — it was a budget machine even back then, and it originally shipped with 4 GB RAM and a 1 TB HDD. The hard drive started failing just four years later, but that's a story for another day.

For the operating system, I'd strongly recommend Linux. Unfortunately, I'm currently running **Windows Server 2022** on mine due to a somewhat embarrassing reason: I can't physically place the laptop next to the router for a wired connection, and its Wi-Fi adapter doesn't play nicely with Linux. Once I can sort out the cable situation, I plan to switch to Linux.

Once you've prepared your device and installed an OS, install Jellyfin. Downloads are [here](https://jellyfin.org/downloads); installation instructions are on the site (or just Google it 😊).

One important step: in the Jellyfin dashboard, go to **Dashboard → Networking** and make sure **"Allow remote connections to this server"** is enabled.

Next, install the Tailscale client on the storage server — but first, let's set up the VDS.

### The VDS

My VDS runs Ubuntu, so the steps below are Ubuntu-specific.

1. Install **nginx**. If you've pointed a domain at this server (as I have), configure HTTPS.
2. Install Headscale — official Ubuntu support is available, and the [installation guide](https://headscale.net/stable/setup/install/official/) is straightforward.
3. Edit the Headscale configuration at `/etc/headscale/config.yaml`:

```yaml
server_url: https://media.example.com

listen_addr: 0.0.0.0:8080
metrics_listen_addr: 127.0.0.1:9090

prefixes:
  v4: 100.64.0.0/10

db_type: sqlite3
db_path: /var/lib/headscale/db.sqlite

tls_cert_path: ""
tls_key_path: ""

derp:
  server:
    enabled: true
    region_id: 999
    region_code: "example_mediaserver"
    region_name: "Headscale DERP on VDS"
    stun_listen_addr: "0.0.0.0:3478"
    private_key_path: /var/lib/headscale/derp_server_private.key
  urls:
    - https://controlplane.tailscale.com/derpmap/default
  auto_update_enabled: true

dns:
  magic_dns: true
  base_domain: vpn.local
  nameservers:
    global:
      - 1.1.1.1
      - 1.0.0.1

log:
  level: info

noise:
  private_key_path: /var/lib/headscale/noise_private.key

database:
  type: sqlite
  debug: false
  gorm:
    prepare_stmt: true
    parameterized_queries: true
    skip_err_record_not_found: true
    slow_threshold: 1000

  sqlite:
    path: /var/lib/headscale/db.sqlite
    write_ahead_log: true
    wal_autocheckpoint: 1000
```

The key value to customize is `server_url` — replace `https://media.example.com` with your actual server address. The `derp` → `server` → `region_code` and `region_name` fields are free-form; set them to whatever makes sense. Leave everything else as-is.

Once done, restart the `headscale` service.

4. Install the Tailscale client on the VDS and connect it to your Headscale node. The VDS itself needs to be part of the Tailscale network so it can reach the storage server via its Tailscale IP:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --advertise-exit-node --login-server https://media.example.com
```

The second command will prompt you to visit a URL like `https://media.example.com/register/randomLettersAndNumbersHere`. Open that in any browser and follow the instructions.

### Connecting the Storage Server to the VPN

Now install the Tailscale client on the storage server. The only difference from a standard install is that you need to point it at your own login server. On Windows, after installing the client, **don't log in through the GUI** — instead, open a Command Prompt or PowerShell and run:

```
tailscale up --login-server https://media.example.com
```

On Linux, the command is identical.

After connecting, run `tailscale ip` to get the storage server's Tailscale IP address. Take note of it — you'll need it for the nginx config. For the examples below I'll use `100.64.0.4`.

### Configuring the Reverse Proxy on the VDS

Back on the VDS, it's time to configure **nginx**. Config file paths may vary by setup. Note that in my case Jellyfin is served under `https://media.example.com/music/` — a dedicated sub-path.

```nginx
server {
  ...
  location /music/ {
    proxy_pass http://100.64.0.4:8096/;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Protocol $scheme;
    proxy_set_header X-Forwarded-Host $http_host;

    proxy_buffering off;
  }

  location /music/socket {
    proxy_pass http://100.64.0.4:8096/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Protocol $scheme;
    proxy_set_header X-Forwarded-Host $http_host;
  }
  ...
}
```

In `proxy_pass`, use the Tailscale IP of your storage server. Port `8096` is Jellyfin's default — change it only if you've configured a different one.

Restart nginx and verify everything works. You're done!

![Web-version of Jellyfin](https://raw.githubusercontent.com/Elorucov/blog/refs/heads/main/personal-music-cloud/jellyfin-web.jpg)