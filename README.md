# tareeqalraha.ae

## Separate-process deployment checklist for two websites

If two different domains are showing mixed behavior (for example one domain showing backend/login pages and another domain showing raw Node.js source code), deploy each site as an isolated app process and isolate their web roots and reverse-proxy rules.

### Target architecture

- **Storefront app**: `tareeqalraha.com` (+ optional `www`) on one Node process.
- **Salon app**: `cutiepieladiessalon.ae` (+ optional `www`) on a second Node process.
- **Optional admin/API subdomains** per brand routed to the corresponding process only.

### 1) Create two app folders (do not share runtime folder)

Example:

- `/home/tareeqal/server.tareeqalraha.com`
- `/home/tareeqal/server.cutiepieladiessalon.ae`

Each folder should contain only that project’s:

- `.env`
- `backend/` (or server build)
- `frontend/dist/` (built assets)
- `package.json`

### 2) Use different ports in each `.env`

Example:

- `server.tareeqalraha.com/.env` → `PORT=3000`
- `server.cutiepieladiessalon.ae/.env` → `PORT=3001`

Also keep domain-specific variables separate:

- `FRONTEND_URL`
- `ADMIN_URL`
- `API_URL`
- CORS allowlist entries
- cookie/session domain values

### 3) Run two PM2 processes (one per website)

```bash
cd /home/tareeqal/server.tareeqalraha.com
pm2 start npm --name tareeqalraha-app -- start

cd /home/tareeqal/server.cutiepieladiessalon.ae
pm2 start npm --name cutiepie-app -- start

pm2 save
pm2 status
```

Never run both domains from one PM2 process if each brand should be isolated.

### 4) Configure two reverse-proxy vhosts (or two cPanel Node app mappings)

Each host must proxy to its own port.

- `tareeqalraha.com` and `www.tareeqalraha.com` → `127.0.0.1:3000`
- `cutiepieladiessalon.ae` and `www.cutiepieladiessalon.ae` → `127.0.0.1:3001`

If you use subdomains:

- `server.tareeqalraha.com` → `127.0.0.1:3000`
- `server.cutiepieladiessalon.ae` → `127.0.0.1:3001`

### 5) Fix the "raw code in browser" symptom

If browser shows JavaScript source instead of site UI, Apache/Nginx is likely serving your source file as text from document root.

Confirm:

- Subdomain Document Root is **not** pointing at `backend/src` or project root.
- It should point to a harmless public folder or only act as reverse proxy.
- Reverse-proxy rule is active and forwards all requests to the right Node port.

### 6) Verify both apps independently

```bash
curl -I https://tareeqalraha.com
curl -I https://cutiepieladiessalon.ae
curl -I https://server.tareeqalraha.com
curl -I https://server.cutiepieladiessalon.ae
pm2 logs tareeqalraha-app --lines 50
pm2 logs cutiepie-app --lines 50
```

You should see each domain handled by only its own process.

### 7) DNS sanity check

Make sure A records match your server IP and there are no stale duplicate records:

- `tareeqalraha.com`
- `www.tareeqalraha.com`
- `cutiepieladiessalon.ae`
- `www.cutiepieladiessalon.ae`
- optional `server.*` and `backend.*`

### Quick diagnosis matrix

- **Main site loads but wrong brand login appears** → wrong vhost mapping or shared `.env`.
- **Raw code appears in browser** → wrong document root / missing proxy.
- **Only one domain works** → one PM2 process down or wrong port.
- **Intermittent mixed behavior** → DNS propagation/cache or duplicate vhost rules.
