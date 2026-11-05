# FreePBX Installation & Configuration Support on Debian 13 Trixie

Step 1 — Verify your Debian 13 server

cat /etc/os-release

uname -r

free -h

df -h /

ip -br addr

Step 2 — Update Debian 13 Dependency

apt update && apt full-upgrade -y

apt install -y curl ca-certificates gnupg

apt install -y wget curl 

Step 3 — Create Tailscale account https://tailscale.com then Install Tailscale

mkdir -p --mode=0755 /usr/share/keyrings

curl -fsSL https://pkgs.tailscale.com/stable/debian/trixie.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null

curl -fsSL https://pkgs.tailscale.com/stable/debian/trixie.tailscale-keyring.list | sudo tee /etc/apt/sources.list.d/tailscale.list

apt update

apt install -y tailscale

systemctl enable --now tailscaled

tailscale up

tailscale status

tailscale ip -4

tailscale netcheck

systemctl status tailscaled

You will get 100.x.x.x server's private Tailscale IPv4 address example: 100.94.13.32.
Install Tailscale on your Ubuntu/Linux desktop and sign in using the same Tailscale account then Test SSH connectivity: ssh root@100.x.x.x Replace 100.94.13.32 with your actual server Tailscale IP. You can now administer Debian remotely through the private Tailscale network.

Step 3 — Install FreePBX 17 on Debian 13

apt update

wget https://github.com/AnythingLinux/FreePBX-Installation/blob/main/freepbx_debian_install.sh -O /tmp/sng_freepbx_debian_install.sh
chmod +x bash /tmp/freepbx_debian_install.sh
bash /tmp/freepbx_debian_install.sh

Step 4 — Install SSL

apt install -y certbot python3-certbot-apache

certbot --apache -d voip.secure-transaction.co

Select: 2: default-ssl.conf               |                       | HTTPS | Enabled
<img width="1920" height="1080" alt="Screenshot from 2026-10-30 18-51-37" src="https://gist.github.com/user-attachments/assets/32ac9876-17cb-4444-b15e-97f43d7816f8" />

certbot renew --dry-run

The installer log is: cd /var/log/pbx && cat freepbx17-install-2026.11.02-18.25.25.log

Step 4 — After installation, verify:

command -v fwconsole

fwconsole --version

asterisk -rx "core show version"

systemctl start asterisk
systemctl status asterisk
systemctl restart asterisk

Once FreePBX is installed, you can access its web interface through your server's Tailscale IP: http://100.x.x.x/

Step 4 — Secure Debian 13 with Tailscale and UFW Firewall. Check the existing firewall configuration:

Install UFW:

apt update

apt install -y ufw

ufw status verbose

systemctl is-active nftables

systemctl is-active firewalld

ufw version

Configure default firewall policies:

ufw default deny incoming

ufw default allow outgoing

Allow SSH through Tailscale:

ip addr show tailscale0

tailscale ip -4

Allow SSH through the Tailscale interface:

ufw allow in on tailscale0 to any port 22 proto tcp

If you are currently connected through your server's public IP, first allow your current administration IP: 

ufw allow from YOUR_ADMIN_PUBLIC_IP to any port 22 proto tcp

ufw allow 22/tcp

You can optionally permit inbound UDP 41641:

ufw allow 41641/udp

Restrict FreePBX Web GUI to Tailscale:

ufw allow in on tailscale0 to any port 80 proto tcp

ufw allow in on tailscale0 to any port 443 proto tcp

Your FreePBX administration interface will then be accessible through: https://YOUR_TAILSCALE_IP/

Do not expose SIP and RTP to every public IP address. Obtain the signaling IP addresses and media IP ranges from your VoIP provider.

ufw allow from PROVIDER_SIGNALING_IP to any port 5060 proto udp

For RTP media:

ufw allow from PROVIDER_MEDIA_CIDR to any port 10000:20000 proto udp

Enable UFW:

ufw show added

ufw enable

ufw status verbose

Expected default policies:

Default: deny (incoming), allow (outgoing)

Remove temporary public SSH access: Once you have successfully tested SSH through Tailscale from another device, remove the temporary public SSH rule if you created it:

ufw delete allow 22/tcp

Verify firewall and network security:

echo "===== FIREWALL ====="

ufw status numbered

echo "===== TAILSCALE ====="

tailscale status

tailscale ip -4

echo "===== LISTENING PORTS ====="

ss -tulnp

echo "===== FIREWALL SERVICE ====="

systemctl status ufw --no-pager

Check connectivity from your administration computer:

tailscale ping YOUR_SERVER_TAILSCALE_IP

ssh root@YOUR_SERVER_TAILSCALE_IP

DigitalOcean Cloud Firewall: If your Debian 13 server is hosted on DigitalOcean, remember that its Cloud Firewall is separate from UFW.

SSH 22, HTTP 80, HTTPS 443, SIP 5060, RTP 10000–20000, UDP 41641

Verify your actual server:

ufw status verbose

tailscale status

tailscale ip -4

ip -br addr

ss -tulnp

# Setup Desktop tailscale
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
tailscale set --ssh
ssh 100.x.x.x
