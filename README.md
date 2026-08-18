# Apache IP Logging | Proxmox

How I deployed an Ubuntu LXC in Proxmox, installed Apache and PHP, and served a page that records visitor IP addresses and rough geolocation. It also covers putting the service behind a Cloudflare Tunnel so it is reachable from outside the local network.

Built as a demo of how much a web server can work out about whoever connects to it. The Cyber Lab page on hmax.space runs on this.

## Overview

- Platform: Proxmox VE
- Container OS: Ubuntu LXC
- Web stack: Apache, PHP
- Optional internet exposure: Cloudflare Tunnel
- Goal: Display and log visitor IP and location data

## Architecture

```text
Visitor -> Cloudflare Tunnel -> Apache/PHP on Ubuntu LXC -> /var/log/visits.log
```

## Prerequisites

- A Proxmox host
- Internet access for package installation
- A new Ubuntu LXC container
- A domain name if you want external access
- A Cloudflare account for tunnel setup

## 1. Create the Ubuntu LXC in Proxmox

Create a new Ubuntu LXC container in Proxmox. If you use Proxmox helper scripts, complete the container deployment first and then open a shell inside the container.

## 2. Update the container

Because this is a fresh install, update the package lists and upgrade installed packages:

```bash
apt update && apt upgrade -y
```

## 3. Install Apache and PHP

Install Apache, PHP, and the Apache PHP module:

```bash
apt install apache2 php libapache2-mod-php -y
```

Once installation finishes, verify that Apache is running:

```bash
systemctl status apache2
```

## 4. Find the container IP address

Use the following command to identify the IP address assigned to the container:

```bash
ip a
```

This is the local IP address of the Apache server.

## 5. Create the PHP page

Edit the default web root and replace the default site with a PHP file:

```bash
nano /var/www/html/index.php
```

Why PHP instead of plain HTML?

- HTML is static and only displays what you wrote.
- PHP runs on the server before the page is sent to the visitor.
- That makes it possible to capture and display dynamic information such as the visitor's IP address.

## 6. Basic visitor IP and location page

Paste the following code into `/var/www/html/index.php`:

```php
<?php
$ip = $_SERVER['REMOTE_ADDR'];
$geo = file_get_contents("http://ip-api.com/json/{$ip}");
$data = json_decode($geo);
$city = $data->city ?? 'Unknown';
$country = $data->country ?? 'Unknown';
?>
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Visitor Logger Demo</title>
</head>
<body>
  <h1>
    If you are reading this, the server has logged your IP of
    <?= htmlspecialchars($ip) ?>
    and location:
    <?= htmlspecialchars($city) ?>, <?= htmlspecialchars($country) ?>
  </h1>
</body>
</html>
```

Example output:

```text
If you are reading this, the server has logged your IP of 192.168.10.85 and location: Unknown, Unknown
```

## 7. Remove the default Apache HTML page

Apache often serves `index.html` before `index.php`, so remove the default HTML page:

```bash
rm /var/www/html/index.html
```

After that, Apache will load `index.php` first.

## 8. Cloudflare aware version with logging

If the site is accessed through Cloudflare Tunnel, the real client IP may be forwarded in the `HTTP_CF_CONNECTING_IP` header. The version below supports that and also writes each visit to a log file.

Before using it, create the log file and give Apache permission to write to it:

```bash
touch /var/log/visits.log
chown www-data:www-data /var/log/visits.log
chmod 664 /var/log/visits.log
```

Replace `/var/www/html/index.php` with:

```php
<?php
header('Content-Type: text/html; charset=UTF-8');

$ip = $_SERVER['HTTP_CF_CONNECTING_IP'] ?? $_SERVER['REMOTE_ADDR'];
$geo = @file_get_contents("http://ip-api.com/json/{$ip}");
$data = $geo ? json_decode($geo) : null;

$city = $data->city ?? 'Unknown';
$country = $data->country ?? 'Unknown';

$log = date('Y-m-d H:i:s') . " - IP: {$ip} - Location: {$city}, {$country}\n";
file_put_contents('/var/log/visits.log', $log, FILE_APPEND);
?>
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Visitor Logger Demo</title>
</head>
<body>
  <h1>
    If you are reading this, hmax has logged your IP of
    <?= htmlspecialchars($ip) ?>
    and location:
    <?= htmlspecialchars($city) ?>, <?= htmlspecialchars($country) ?>
  </h1>
  <p>
    This page was built as a demonstration of how web servers can log visitor
    information. Any website you visit can see the IP address used to connect.
  </p>
  <p>
    This link appears harmless, but it demonstrates that links can go somewhere
    unexpected:
    <a href="https://google.com">Open link</a>
  </p>
</body>
</html>
```

To review the log file:

```bash
cat /var/log/visits.log
```

## 9. Expose the site with a Cloudflare Tunnel

Making the Apache page reachable from outside the local network without port forwarding the container.

Rough process:

1. Buy a domain on Namecheap.
2. Create a free Cloudflare account.
3. Add the domain to Cloudflare.
4. Point the domain nameservers at Namecheap to the ones Cloudflare gives you.
5. Install and authenticate `cloudflared` inside the Ubuntu LXC.
6. Create a tunnel and route your chosen hostname to the Apache service.
7. Configure the tunnel to forward traffic to `http://localhost:80`.

## Security and ethics

- This project is intended for homelab learning and security awareness.
- Logging visitor IP addresses may be subject to privacy laws, school rules, workplace rules, or platform policy.
- If you publish this project, make it clear that the page records connection data.
- Exposing a lab system to the internet increases risk, even when using a tunnel.

## End result

At the end of this homelab, you will have:

- An Ubuntu LXC container running in Proxmox
- Apache and PHP installed
- A PHP page that displays visitor IP and location data
- A log of every visit written on the server
- Optional public access through Cloudflare Tunnel
