# 🔍 Web Enumeration

---

## 📖 Description

Web enumeration is the reconnaissance phase of a web penetration test. Before attempting any exploitation, it's essential to map the target thoroughly: what technologies it runs, what subdomains and virtual hosts exist, what directories and files are exposed, and what known vulnerabilities the stack may have.

A solid enumeration phase directly determines the quality of the attack surface identified. Skipping or rushing it often means missing critical entry points.

---

## 1️⃣ Technology & Version Enumeration

The first step is identifying what the target is running — web server, programming language, frameworks, and CMS. This information shapes every subsequent decision.

```bash
whatweb -v http://TARGET
whois domain.com
```

Also check these files manually, as they often reveal structure and sensitive paths:

```
http://TARGET/robots.txt     → disallowed paths (often hides admin panels or sensitive dirs)
http://TARGET/sitemap.xml    → full site structure
```

---

## 2️⃣ Subdomain & Virtual Host Enumeration

### Subdomain enumeration

Subdomains often host forgotten applications, staging environments, or internal tools with weaker security. Start with a smaller wordlist and scale up if needed:

```bash
gobuster dns -d domain.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# Scale up if nothing interesting is found
gobuster dns -d domain.com -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
```

### Virtual host (VHost) enumeration

Virtual hosts share the same IP but serve different applications based on the `Host` header. Gobuster DNS won't find these — use ffuf to fuzz the header directly:

```bash
# First pass without filter to identify the baseline response size
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -H "Host: FUZZ.domain.com" \
  -u http://TARGET

# Second pass filtering out the baseline size
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -H "Host: FUZZ.domain.com" \
  -u http://TARGET \
  -fs BASELINE_SIZE
```

### Adding discovered hosts

For every VHost or subdomain found, add it to `/etc/hosts` so the browser and tools can resolve it:

```bash
echo "IP vhost.domain.com" | sudo tee -a /etc/hosts
```

---

## 3️⃣ Directory Enumeration

Directory brute-forcing reveals hidden files, admin panels, backup files, and endpoints not linked from the main application. Always run two passes: a fast one to get quick wins, and a deeper one in the background.

```bash
# First pass — fast
gobuster dir -u http://domain.com \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,bak,old,zip,html,js \
  -t 50

# Second pass — broader (run in background while working on other findings)
gobuster dir -u http://domain.com \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,bak,old,zip,html,js \
  -t 50
```

If the target uses HTTPS, add `-k` to skip certificate verification:

```bash
gobuster dir -u https://domain.com \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,bak,old,zip,html,js \
  -t 50 -k
```

> 🚨 **Important**: For every interesting subdirectory found, iterate into it — there may be more hidden content one level deeper:
>
> ```bash
> gobuster dir -u http://domain.com/directory/ \
>   -w /usr/share/wordlists/dirb/common.txt \
>   -x php,txt,bak,old,zip,html,js \
>   -t 50
> ```

---

## 4️⃣ Nikto Scan

Nikto performs automated checks for common web server misconfigurations, outdated software, dangerous files, and known vulnerabilities. It's not subtle, but it's fast and often surfaces low-hanging fruit:

```bash
nikto -h http://TARGET
```
