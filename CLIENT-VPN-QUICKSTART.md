# AWS Client VPN — Quick Start Guide (for Users)

*How to install the VPN client and connect to the HMSG network.*

> What you'll need:
> - **One `.ovpn` file** — your personal VPN profile, provided by IT (filename: `HSMG-client-config.ovpn`)
> - **Your Microsoft work account** — email + password (+ sign-in on your phone if asked)

---

## 1. Install the app (one-time)

| Your computer | How to install |
|---|---|
| **Windows 10 / 11** | Go to **https://aws.amazon.com/vpn/client-vpn-download/** → download **AWS Client VPN for Windows** (.msi) → run it and follow the installer (you'll need administrator rights on your PC). |
| **Mac** | Open the **App Store** → search **"AWS Client VPN"** → install the free app. |
| **Linux (Ubuntu / Fedora)** | On the same download page, pick your distribution's package (.deb / .rpm), install it with your package manager, then use the same steps below. |

## 2. Import your profile

1. Save the `.ovpn` file you received (`HSMG-client-config.ovpn`) somewhere easy to find — e.g. your **Downloads** folder. 📁
2. Open the **AWS Client VPN** app.
3. In the top menu click **File → Import Profile…**
4. Select the `.ovpn` file and click **Open**.

Your profile now appears in the app (listed as `HSMG-client-config`).

## 3. Connect

1. Select your profile (`HSMG-client-config`) in the app.
2. Click **Connect**. 🔌
3. A **sign-in window** opens in your web browser:
   - Choose your **Microsoft (Entra ID) work account**
   - Enter your **email and password**
   - Approve the **multi-factor sign-in** on your phone if it asks
4. Go back to the app — it should now show **Connected** ✅

> Only traffic to HMSG systems goes through the VPN. Your normal internet (Google, email…) is unaffected.

## 4. What you can do once connected

- **Remote Desktop (RDP)** to the internal systems IT points you to (including the database server).
- Access internal web/database tools that are only reachable from the HMSG network.

> IT will give you the exact address(es) and login details for the system(s) you need.

## 5. Disconnect when finished

- In the app, click **Disconnect**, or
- Click the tray/menu icon and **Exit** the app.

## Troubleshooting

| Problem | What to do |
|---|---|
| **"Could not connect" or profile has an error** | Your profile may have expired or been updated. Ask IT for a **fresh `.ovpn` file** and re-import it (step 2). |
| **Sign-in keeps prompting or fails** | Make sure you're using your **work Microsoft account**, not a personal one. If you just changed your password, wait a few minutes and try again. |
| **"Connection timed out"** | The network/firewall you're on may block the VPN port (UDP 443). Try another network (e.g. a phone hotspot) — or contact IT. |
| **Connected, but can't reach a system** | Confirm the browser sign-in was approved. If it still fails, tell IT which system you're trying to reach. |

## Please remember 🙏

- **Never share your `.ovpn` file** — it's personal to you, like a password.
- **Don't copy it** to shared drives or email it to others.
- **Disconnect** when you're done, especially on a shared or company machine.

---

*Support contact: **IT / HAEE** — say you're using **AWS Client VPN** (the tunnel uses port UDP 443, which some networks block).*