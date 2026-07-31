# Remote Job Finder

A web app that lets job seekers search, filter, and sort live remote job listings — pulled in real time from a public jobs API. Built to solve a real problem: sorting through remote job boards is slow and cluttered; this gives a fast, single-page way to search by keyword, filter by category, and sort by date, title, or company.

**Live app (via load balancer):** http://100.58.191.155

## Features

- Fetches live remote job listings from the Remotive API on page load
- Search by job title, company name, or tag/skill keyword
- Filter by job category
- Sort by newest, oldest, title (A–Z), or company (A–Z)
- Graceful error handling if the API is down or returns unexpected data
- Clear "no results" state when a search/filter combination matches nothing
- Fully responsive, no build step or dependencies — plain HTML/CSS/JS

## API Used

**[Remotive API](https://remotive.com/remote-jobs/api)** — a free, public API for remote job listings. No API key or authentication is required, so there's nothing sensitive to secure or expose. Full credit to Remotive for providing the data; all "Apply" links route back to the original listing on their site.

## Running Locally

No installation or build step needed.

1. Clone the repo:
   ```bash
   git clone https://github.com/ahmedosman-design/remote-job-finder.git
   cd remote-job-finder
   ```
2. Open `index.html` directly in any browser (double-click it, or right-click → Open With → your browser).

That's it — the app fetches live data as soon as the page loads.

## Deployment

The app is deployed on two identical web servers behind a load balancer.

**Architecture:**
```
User → Lb01 (HAProxy, round-robin) → Web01 (nginx) or Web02 (nginx)
```

### Deploying to Web01 and Web02

Both servers run nginx and serve the same static `index.html`.

```bash
# SSH into the server
ssh -i <path-to-key> ubuntu@<server-ip>

# Install nginx if not already present
sudo apt update
sudo apt install nginx -y

# Remove the default nginx landing page
sudo rm -f /var/www/html/index.nginx-debian.html

# Pull the app directly from GitHub into nginx's web root
sudo curl -o /var/www/html/index.html https://raw.githubusercontent.com/ahmedosman-design/remote-job-finder/main/index.html

# Restart nginx to serve the new file
sudo systemctl restart nginx
```

This was run identically on both **Web01** (`3.82.141.92`) and **Web02** (`18.212.171.186`).

### Configuring the Load Balancer (Lb01)

Lb01 runs HAProxy and distributes incoming traffic between Web01 and Web02 using round-robin balancing.

```bash
ssh -i <path-to-key> ubuntu@100.58.191.155
sudo apt update
sudo apt install haproxy -y
sudo nano /etc/haproxy/haproxy.cfg
```

The following block was appended to the end of the existing config file (the default `global` and `defaults` sections were left untouched):

```
frontend http_front
    bind *:80
    default_backend web_servers

backend web_servers
    balance roundrobin
    server web01 3.82.141.92:80 check
    server web02 18.212.171.186:80 check
```

Then validated and restarted:
```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg   # confirms the config is valid
sudo systemctl restart haproxy
```

### Verifying the load balancer works

Requesting the load balancer's address repeatedly shows the `X-Served-By` response header alternating between `7105-web-01` and `7105-web-02`, confirming traffic is being split across both servers:

```bash
curl -sI http://100.58.191.155 | grep -i x-served-by
```

## Challenges & How They Were Solved

- **`nano: command not found` on fresh servers** — resolved by installing it with `sudo apt install nano -y` before editing config/HTML files.
- **Duplicate `frontend`/`backend` blocks in `haproxy.cfg`** caused HAProxy to fail on startup with a "same name as backend" error. Diagnosed using `sudo haproxy -c -f /etc/haproxy/haproxy.cfg`, which pointed to the exact duplicate line, then removed the redundant block.
- **Getting the app file onto the servers without a clunky manual paste** — instead of editing HTML by hand over SSH, the app is pulled directly from the GitHub raw URL using `curl`, which keeps the deployed version in sync with the repo and avoids paste/formatting errors.

## Credits

- Job data: [Remotive API](https://remotive.com/remote-jobs/api)
- Fonts/icons: none external beyond system fonts
- Built and deployed by Ahmed Osman

## Demo Video
