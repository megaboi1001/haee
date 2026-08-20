# AWS Client VPN — Quick Start Guide (for Users)

*How to install the VPN client and connect to the HMSG network.*

> What you'll need: **one `.ovpn` file** (your personal VPN profile, provided by IT) and
> your **Microsoft work account** (email + password + sign-in on your phone if asked).

---

## 1. Install the app

| Your computer | How to install |
|---|---|
| **Windows 10 / 11** | Go to **https://aws.amazon.com/vpn/client-vpn-download/** → download the **AWS Client VPN for Windows** (.msi) → double-click it and follow the installer (you'll need administrator rights on your PC). |
| **Mac** | Open the **App Store** → search **"AWS Client VPN"** → install the free app. |

*One-time step — you only install the app once.*

## 2. Import your profile

1. Save your `.ovpn` file somewhere you can find it (e.g. **Downloads**). 📁
2. Open the **AWS Client VPN** app.
3. In the top menu click **File → Import Profile…**
4. Select your `.ovpn` file and click **Open**.

You'll see your profile appear in the app (it's usually called something like `hmsg-rac-prd`).

## 3. Connect

1. Select your profile in the app.
2. Click **Connect**. 🔌
3. A **sign-in window** will open in your web browser:
   - Choose your **Microsoft (Entra ID)** work account
   - Enter your **email and password**
   - Approve the **Multi-Factor sign-in** on your phone if it asks
4. Go back to the app — it should now show **Connected** ✅

> Done — you're on the internal network. Only traffic to HMSG systems goes through the VPN;
> your normal internet (Google, email…) is unaffected.

## 4. Disconnect when finished

- In the app, click **Disconnect**, or
- Click the tray/menu icon and **Exit** the app.

## Troubleshooting

| Problem | What to do |
|---|---|
| **"Could not connect" or profile has an error** | Your profile may have expired or been updated. Ask IT for a **fresh .ovpn file** and re-import it (step 2). |
| **Sign-in keeps prompting or fails** | Make sure you're using your **work Microsoft account**, not a personal one. If you just changed your password, try again after a few minutes. |
| **"Connection timed out"** | The network/firewall you're on may block the VPN port. Try a different network (e.g. phone hotspot) — or contact IT. |
| **Connected, but can't reach the system** | You must be connected with the browser sign-in **approved**. If it still fails, contact IT with the system name you're trying to reach. |

## Please remember 🙏

- **Never share your `.ovpn` file** — it's personal to you, like a password.
- **Don't copy it** to shared drives or email it to others.
- Disconnect when you're done, especially on shared/company machines.

---

*Support contact: **IT / HAEE** — mention you're using AWS Client VPN because the tunnel uses
a special port (UDP 443) which some networks block.*