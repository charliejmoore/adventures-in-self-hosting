# Adventures in Self-hosting

Documenting my basic self-hosting adventures.

## Self-hosted Services
- [JellyFin](https://jellyfin.org/): A free, open source media server. Can host movies/shows/music/books/etc. Great if you have a physical media collection you want to access digitally in a nice UI.
- [Pi-hole](https://docs.pi-hole.net/): A network-level ads and tracker blocking application that acts as a DNS sinkhole for private networks.
- [Grafana](https://grafana.com/): Observability platform. I didn't strictly *need* this for my current set up. I have it wired up to consume data from JellyFin and Pi-hole. Pi-hole already comes with an admin dashboard and the JellyFin stats aren't super interesting. At this point, it was mostly just a toy project to get some basic experience with Grafana. Possibly more utility out of this as I expand my self-hosted services.

## Enabling Technologies
- [Prometheus](https://prometheus.io/): A free software application for event monitoring and alerting. Used here to pull data from JellyFin/Pi-hole and feed it into Grafana.
- [Docker](https://www.docker.com/): Used here for making containers to run the various services in.

## Equipment
*This is obviously very flexible, use what you have on hand. These are just what I had*

- [Raspberry Pi](https://www.raspberrypi.com/) 5 with a microSD card
- Old laptop with a large external hard drive (for media storage)
- Monitor/mouse/keyboard

## Pi-hole Setup
- Have a Raspberry Pi with Linux installed. I didn't have to do anything for this part, mine was already configured so I don't have more instructions to get to through this step.
- I hooked up my Raspberry Pi to a monitor with a mouse and keyboard so I could interact with it via the typical OS GUI. You can also ssh into your Raspberry Pi from another computer if you know the Raspberry Pi's IP and name and password etc.

### On the Raspberry Pi
- [Install Pi-hole](https://docs.pi-hole.net/main/basic-install/)
  - Open up a terminal on your raspberry pi and run `curl -sSL https://install.pi-hole.net | bash` if you aren't sketched out by piping bash. If you are, see the link for alternatives.
  - You may need to be in sudo mode. I had some dependencies that needed updating before this would successfully install.
  - Go through the steps in the pi-hole installer popup
  - It will give you a password for the admin portal at some point, write that down. You can change it later.
- Get your Raspberry Pi's MAC address by running `ip link show`. If you are connected via `eth0` (wired) or `wlan0` (Wi-Fi). I did ethernet, so no Wi-Fi instructions in this README.md. Save the MAC address for use in a bit.

### On another computer
*You may be able to do all of this on the Raspberry Pi itself, but I was getting frustrated with how sluggish the UI was responding to my mouse so I'm glad I was doing this part on my typical laptop*.

- Now, you'll need access to your router. Hopefully you know your login info and password. If you've never messed with it, it might be written physically on your router so take a gander. Google says that typical router IP addresses are `192.168.1.1` or `192.168.0.1` so you might try those out in a browser to see if you get your router admin login page.
- Log into your router via your browser
- Look for some section that has to do with DHCP Reservation/Address Reservation. This might be buried in some "Advanced Settings" type page.
- There should be a section to add a new reservation. Enter the Raspberry Pi's MAC address here and pick an available IP address to map it to.
- Save that.
- (back on the Raspberry Pi now) Reboot the Raspberry Pi by running `sudo reboot`.

## Setting Up Pi-hole to Export Data For Grafana
On the Raspberry Pi:

- Install Docker: `curl -fsSL https://get.docker.com | sh`

- Optional: give Docker permission to run without needing sudo elevation: `sudo usermod -aG docker $USER`

- Go to the Pi-hole admin site  `http://<your-rasperrbypi-ip>/admin`
- Go to `Settings -> API` and switch from Basic to Expert mode (upper right corner).
- Do `Configure app password` and copy and save the password it gives you
- You'll have to log out of the admin page and log back in (with your original password, not the API password you just generated)
- Now, make a docker container called `pihole-exporter` and install [`pihole6_exporter`](https://hub.docker.com/r/amonacoos/pihole6_exporter) in it. You will need to put that Pi-hole API password where you see `PASTE_API_PASSWORD_HERE` below and you will need to update `RASPBERRY_PI_IP_ADDRESS` with your Raspberry Pi's IP address. Update that and then run:

```sh
docker run -d --name pihole-exporter \
  --restart unless-stopped \
  -p 9617:9666 \
  -e PIHOLE_HOST=RASPBERRY_PI_IP_ADDRESS \
  -e PIHOLE_API_KEY=PASTE_API_PASSWORD_HERE \
  -e PIHOLE_SCHEME=http \
  -e PIHOLE_PORT=80 \
  amonacoos/pihole6_exporter:latest
```

- You can check that it worked by running `docker ps -a` and seeing if you see `pihole-exporter` listed there as "Up". If there was a problem, you can see logs by running: `docker logs pihole-exporter`.
- You can also run `curl http://localhost:9617/metrics` to see if you see data output as expected (e.g., `pihole_query_count{category="total"} 1523`, `pihole_client_count{category="active"} 8`, etc.).

That's all we need to do on the Raspberry Pi (at least with my setup). I have JellyFin and Grafana on the old laptop, so we pop over there now.

## Setting Up Self-Hosted Grafana and Hooking It Up To JellyFin and Pi-hole on Laptop
- You'll need [Docker Desktop](https://www.docker.com/products/docker-desktop/) if you don't have it already
- Launch it. Feel free to "skip" signing in and making an account because that is exhausting

### Detour to JellyFin
*I have the JellyFin mac app. Your API keys may live in a different place.*

- Get API key by going through: `Dashboard -> Advanced -> API Keys -> the "+" button`. I made a new key named "Grafana". You can call it whatever you want. Copy that.

### Detour to Your Terminal/File System
- Make a folder `~/monitoring`. I did this by being in the top level home directory to begin with and running `mkdir monitoring`
- Put the `docker-compose.yml` and `prometheus.yml` into `~/monitoring`
- Edit the `prometheus.yml` to include your Raspberry Pi IP address instead of the `RASPBERRY_PI_IP_ADDRESS` placeholder.
- Edit the `docker-compose.yml` to include your JellyFin API key instead of the `PASTE_JELLYFIN_API_KEY_HERE` placeholder.
- Move into `~/monitoring` with `cd ~/monitoring`
- Now start everything by running: `docker compose up -d`. That should install a bunch of stuff.
- Check that it worked with: `docker compose ps`, you should see `prometheus`, `grafana`, `jellyfin-exporter` all listed as Up and running.
- If you forgot to update the files with your keys like I did, you can run `docker compose up -d jellyfin-exporter` (insert whatever container name) to restart which docker container needs updating.
- You can also run this command (insert whatever container name) to see logs: `docker compose logs jellyfin-exporter`

### Detour to Your Browser
Now let's see our good work happening in the browser.

- Prometheus should be visible at: `http://localhost:9090`. If you go to `Status -> Targets` you should see both Pi-hole and JellyFin listed there as green and UP.
- Grafana lives at `http://localhost:3000`. The default login info is `admin`/`admin`. It will prompt you to make a new password after you log in with those.

### In Grafana
- Connect Grafana to Prometheus: `Connections -> Data sources -> Add data source -> Prometheus` and set the URL field to `http://prometheus:9090`. Click `Save & Test` and it should be successful and green.
- Now you can set up your Dashboards.
- For Pi-hole, I went to make a new dashboard and manually set it up. I added visualizations and connected them to queries like `pihole_query_count{category="blocked"}` etc. Available queries should populate in the Query dropdown. I put it in "Code" mode instead of "Build" mode.
- For JellyFin, I made another Dashboard but did import from JSON from the JSON found [here](https://github.com/rebelcore/jellyfin_grafana) (`dashboards>jellyfin.json`). I did have to manually edit the queries in some of the visualizations to get rid of `instance=~\"$instance\"`, but not all of them for some reason.
