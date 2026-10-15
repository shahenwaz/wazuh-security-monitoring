# Wazuh Server Setup

Wazuh SOC home lab - Ubuntu Server 24.04 LTS

This guide covers installing Wazuh on the Ubuntu VM, accessing the dashboard, and checking the services. Troubleshooting is included at the end only if needed.

**Before starting:** Complete the [Ubuntu Server Setup](ubuntu-server-setup.md). In my lab, the Ubuntu VM uses `192.168.56.10` on the VirtualBox Host-only network.

## 1. Install Wazuh

Start the Ubuntu VM and connect from Windows Terminal:

```powershell
ssh socadmin@192.168.56.10
```

Download the Wazuh installation script used in this lab (Wazuh 4.14):

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
```

Run the all-in-one installer:

```bash
sudo bash ./wazuh-install.sh -a
```

The installer sets up the **Wazuh Manager, Indexer, Dashboard, and Filebeat** on the Ubuntu server. It may take some time, so wait until installation finishes.

Save the dashboard credentials shown at the end. **Do not commit passwords or generated installation files to GitHub or any of your documentations.**

> **Note:** These commands reflect the Wazuh 4.14 installation used in my lab. Check the [official Wazuh quickstart](https://documentation.wazuh.com/current/quickstart.html) for the appropriate installer if you use a newer release.

## 2. Open the dashboard

With the Ubuntu VM running, open this address in a browser on the Windows host:

```text
https://192.168.56.10
```

Sign in with the credentials generated during installation. A browser warning about the self-signed certificate is expected in this local lab; confirm that you're connecting to your own VM before proceeding.

## 3. Verify the services

In the Ubuntu SSH terminal, run:

```bash
sudo systemctl is-active wazuh-manager wazuh-indexer filebeat wazuh-dashboard
```

All four should return `active`. In the dashboard, also check that the **Server API** is **Online**.

If everything is working, the Wazuh Server setup is complete. Agent installation and monitoring will be documented separately.

## Optional troubleshooting

Only use the relevant section if you encounter the same problem.

### ⏳ Wazuh installation takes too long

The installer doesn't show a percentage. To watch its log, **open a second tab in the windows terminal and connect to the SSH again there with your credentials and run:**

```bash
sudo tail -f /var/log/wazuh-install.log
```

Press `Ctrl+C` in the second terminal to stop watching the log; leave the original installer running. Some components can take several minutes without new messages.

### ⚠️ Wazuh Manager fails to start (timeout)

Check the service and its logs:

```bash
sudo systemctl status wazuh-manager --no-pager -l
sudo tail -n 40 /var/ossec/logs/ossec.log
```

In my VM, systemd allowed only **45 seconds** for startup. The manager timed out while starting, so I increased the limit to **180 seconds**.

Check your current timeout first:

```bash
systemctl show wazuh-manager -p TimeoutStartUSec
```

If the failure is also a startup timeout, edit the service override:

```bash
sudo systemctl edit wazuh-manager
```

Add the following **above the editor's comment section**, save, and exit:

```ini
[Service]
TimeoutStartSec=180
```

Apply and test:

```bash
sudo systemctl daemon-reload
sudo systemctl restart wazuh-manager
sudo systemctl is-active wazuh-manager
```

Expected: `active`. Don't use this workaround for unrelated service errors.

### 🔌 Dashboard shows Server API Offline

First check that the manager is active and the API is listening:

```bash
sudo systemctl is-active wazuh-manager
sudo ss -lntp | grep 55000
```

Test the API locally:

```bash
curl -sk --max-time 10 https://127.0.0.1:55000/
```

An `Unauthorized` response without a token is normal. It confirms that the API is responding and requires authentication.

The dashboard's default API address was `https://127.0.0.1:55000`, so I left it unchanged.

If the API still shows **Offline**, check the logs:

```bash
sudo tail -n 40 /var/ossec/logs/api.log
sudo journalctl -u wazuh-dashboard -n 40 --no-pager
```

In my case, the logs showed a **CTI service request error** during version checks, but other authenticated API requests were successful.

After confirming the API was responding:

1. Open the Wazuh Dashboard at `https://192.168.56.10`.
2. Press **Ctrl + Shift + R** to force-refresh the page.
3. Wait for the dashboard to reload.
4. Check **Server APIs** again.

The API status changed to **Online** without modifying any credentials, certificates, or connection settings.

> **Note:** A force refresh can resolve temporary dashboard loading issues, but it won't fix an API service that is actually down. Always verify the service and logs first.

---

**Result:** Wazuh was successfully installed, and both the Dashboard and Server API were verified as working.

I also created a VirtualBox snapshot of the working server before moving on to the Windows agent setup. This gives me a recovery point in case something goes wrong during the next stages of the lab.
