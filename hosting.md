# Hosting the Inventory app on AWS (Amazon Linux + Nginx + PM2)

Simple plan:

1. Launch a server on AWS and connect to it with SSH.
2. Install the tools and clone the repo.
3. Run the backend with **PM2**.
4. Build the frontend with `npm run build`.
5. Point Nginx at the frontend and the backend. Done.

```
Browser ── http://YOUR_IP ──► Nginx (port 80)
                                ├── /api/...      → backend on port 8000 (PM2)
                                └── anything else → frontend/dist (React files)
```

Every backend route starts with `/api` (`/api/product`, `/api/uploads/...`), so Nginx needs just **one** rule for the whole backend.

> Replace **`YOUR_IP`** with your Public IPv4 address and **`YOUR_REPO_URL`** with your GitHub repo link.

---

## Step 1: Create instance

1. AWS Console → **EC2** → **Launch instance**
2. OS: **Amazon Linux 2023**
3. Key pair: **Create new** → name: `inventory-key` → **.pem**
4. Tick **Allow SSH** + **Allow HTTP**
5. Launch → copy the **Public IPv4 address**

> ⏰ **NOTE:** The session has up to a **3 hour limit**. After 3 hours the instance stops.
> Your files are safe. See **[Restart after 3 hours](#restart-after-3-hours)** at the bottom.

---

## Step 2: Connect

Open a terminal in the folder where `inventory-key.pem` is:

```bash
chmod 400 inventory-key.pem
ssh -i inventory-key.pem ec2-user@YOUR_IP
```

---

## Step 3: Install tools

```bash
sudo dnf install -y git
curl -fsSL https://rpm.nodesource.com/setup_24.x | sudo bash -
sudo dnf install -y nodejs
sudo npm install -g pm2
sudo dnf install -y nginx
sudo nginx
```

Check: open `http://YOUR_IP` in the browser. You should see the Nginx page.

---

## Step 4: Download project

```bash
git clone YOUR_REPO_URL hosting
```

---

## Step 5: Env file

```bash
cd ~/hosting/backend
nano .env
```

Paste your env file, then press **Ctrl+O**, **Enter**, **Ctrl+X**.

---

## Step 6: Backend

```bash
cd ~/hosting/backend
npm ci
pm2 start index.js --name backend
pm2 save
pm2 startup
```

`pm2 startup` shows one long command. **Copy it, paste it, and press Enter.**

---

## Step 7: Frontend

```bash
cd ~/hosting/frontend
npm ci
npm run build
chmod 755 /home/ec2-user
```

---

## Step 8: Nginx

```bash
sudo nano /etc/nginx/conf.d/inventory.conf
```

Paste this, then press **Ctrl+O**, **Enter**, **Ctrl+X**:

```nginx
server {
    listen 80;
    server_name _;

    # Frontend: the built React files
    root /home/ec2-user/hosting/frontend/dist;
    index index.html;

    # Allow picture uploads up to 5 MB (Nginx allows only 1 MB by default)
    client_max_body_size 6M;

    # Frontend: any address that isn't /api, e.g. /read or /create, loads the React app
    location / {
        try_files $uri /index.html;
    }

    # Backend: every /api request (all methods: GET, POST, PATCH, DELETE) goes to Node
    location /api {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
    }
}
```

Test and reload. It must say **"syntax is ok"** and **"test is successful"**:

```bash
sudo nginx -t
sudo nginx -s reload
```

---

## Step 9: Open website

```
http://YOUR_IP
```

Use `http`, not `https`.

---

# Restart after 3 hours

### 1. Start the instance

AWS Console → **EC2** → **Instances** → select it → **Instance state** → **Start instance**. Wait for **Running**.

### 2. Copy the new IP

The IP **changes every time**. Copy the new **Public IPv4 address**.

### 3. Connect

```bash
ssh -i inventory-key.pem ec2-user@NEW_IP
```

### 4. Start backend

```bash
cd ~/hosting/Backend
pm2 start index.js --name backend
```

If it says "already launched", that's ok.

### 5. Start frontend

```bash
sudo nginx
```

If it says "Address already in use", that's ok.

### 6. Open website

```
http://NEW_IP
```

---

# Updating the site

After you push new code to GitHub, SSH into the server and run:

```bash
cd ~/hosting && git pull
cd ~/hosting/backend && npm ci && pm2 restart backend
cd ~/hosting/frontend && npm ci && npm run build
```

---

> **Never** upload `inventory-key.pem` or `.env` to GitHub.
