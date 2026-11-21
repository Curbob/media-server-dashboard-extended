# Trying the VPN check with gluetun

gluetun runs the VPN in a container, and only the containers set to use its network go through it. That is a
common way to keep just the download client behind a VPN. The dashboard isn't behind it, so it asks gluetun
itself: Settings → Live feed → VPN check → "The VPN runs in a gluetun container".

This folder starts gluetun with ProtonVPN (WireGuard) and a tiny `behind-vpn` container that prints the address
it goes out with every minute, as a download client would. Your other containers aren't touched.

## Set up

1. Copy this folder anywhere on the machine that runs Docker, and open a terminal in it.
2. `cp .env.example .env` and put your WireGuard private key in `.env`
   (account.proton.me → VPN → WireGuard → create a configuration → the value after `PrivateKey =`).
   Keep it private: it stays in `.env`, which Git ignores.
3. Make a key for the dashboard: `docker run --rm qmcgaw/gluetun genkey`. It prints a line of about 22 letters and
   digits (the first run also shows Docker downloading the image). Open `auth/config.toml` in a text editor (no
   `sudo` needed) and put that key in place of `REPLACE-WITH-YOUR-KEY`, keeping the quotes. Check with
   `grep '^apikey' auth/config.toml`.
4. `docker compose up -d`, then `docker compose logs -f gluetun` until it says the VPN is up (Ctrl+C to stop
   watching). `docker compose logs behind-vpn` shows the VPN's address and provider, not yours.

## In the dashboard

Settings → Live feed → VPN check: switch it on, choose **gluetun**, and fill in

- **gluetun's address:** `http://<this machine's address>:8001`. `hostname -I` shows it (the first one,
  e.g. `192.168.1.20`). Not `localhost`: inside the dashboard's container that is the container itself.
- **API key:** the key from step 3.

**Check now** should say "gluetun's VPN runs: its traffic goes out through Proton AG (…)". Save, and the
security guard shows up in the data center within two minutes.

**Seeing the alarm.** Stopping the whole container (`docker compose stop gluetun`) only gives "VPN ?": the
dashboard can't ask a stopped gluetun, and nothing behind it has a network then, so nothing leaks. To see the
real alarm, stop the VPN *inside* gluetun:

1. In `auth/config.toml`, remove the `#` in front of the `test-switch` role, put a second key from
   `docker run --rm qmcgaw/gluetun genkey` in it, and `docker compose restart gluetun`.
2. Stop the VPN (replace `SECOND-KEY`):
   `curl -X PUT -H "X-API-Key: SECOND-KEY" -H "Content-Type: application/json" -d '{"status":"stopped"}' http://localhost:8001/v1/vpn/status`
3. Within two minutes the guard is gone, the red light spins and the feed says "gluetun says the VPN is stopped".
4. Start it again with the same command and `"running"`; the guard comes back. Then put the `#` back in front of the test role.

A VPN app on the machine itself can stay on: gluetun's tunnel then runs inside it and still has an address of its
own. Turn it off to try the usual setup, where only what is behind gluetun goes through a VPN.

## Clean up

`docker compose down` removes both containers.
