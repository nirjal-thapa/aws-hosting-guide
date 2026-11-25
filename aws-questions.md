# AWS & Deployment: Questions and Easy Answers

Short answers you can remember. Read one section at a time.

**Sections**
1. [Cloud basics](#1-cloud-basics)
2. [EC2 basics](#2-ec2-basics)
3. [How our app runs](#3-how-our-app-runs)
4. [Commands we used and why](#4-commands-we-used-and-why)
5. [Money: cost, start and stop](#5-money-cost-start-and-stop)
6. [CI/CD pipeline](#6-cicd-pipeline)
7. [DevOps](#7-devops)
8. [IaaS and IaC](#8-iaas-and-iac)
9. [Architecture](#9-architecture)
10. [How we built it (Kasari banayo?)](#10-how-we-built-it-kasari-banayo)
11. [Docker](#11-docker)
12. [How GitHub Actions works](#12-how-github-actions-works)
13. [When something breaks](#13-when-something-breaks)
14. [Quick recall table](#14-quick-recall-table)
15. [Key points to remember](#15-key-points-to-remember)

---

## 1. Cloud basics

### What is cloud computing?
Using **someone else's computers over the internet** instead of buying your own.
You rent servers, storage and databases and **pay only for what you use**.

> 🧠 Remember: **Cloud = renting computers on the internet.**

### Why use the cloud instead of my own computer?
- No need to buy hardware
- Start a server in **minutes**
- Runs **24/7**, even when your laptop is off
- Get a **public IP**, so anyone can open your website
- **Pay as you go**, and stop it when you don't need it

### What is AWS?
**Amazon Web Services**, Amazon's cloud. It has 200+ services. The ones we touched:

| Service | What it is |
|---|---|
| **EC2** | Rent a virtual computer (server) |
| **EBS** | The hard disk attached to EC2 |
| **Security Group** | Firewall: which ports are open |
| **Billing / Budgets** | See and control your costs |

### What are IaaS, PaaS and SaaS?

| Type | You get | Example |
|---|---|---|
| **IaaS** (Infrastructure) | A raw server. You install everything. | **AWS EC2** (what we used) |
| **PaaS** (Platform) | Upload code, they run it. | AWS Elastic Beanstalk, Render, Vercel |
| **SaaS** (Software) | A ready-to-use app | Gmail, Google Docs |

> 🧠 Remember: **I**aaS = **I** do everything, **S**aaS = **S**omeone else does everything.

### What is a Region and an Availability Zone?
- **Region** = a city or area where AWS has data centers (for example Mumbai `ap-south-1`, N. Virginia `us-east-1`).
- **Availability Zone (AZ)** = one data center building inside a region.

Pick the region **closest to your users** for a faster site.

---

## 2. EC2 basics

### What is EC2?
**Elastic Compute Cloud**, a **virtual computer** you rent from AWS.

### What is an instance?
One running EC2 computer. "Launch an instance" = "start a new server".

### What is an AMI?
**Amazon Machine Image**, the **operating system template** for the server.
We used **Amazon Linux 2023**.

### What is an instance type?
The **size** of the computer (CPU + RAM).

| Type | CPU | RAM | Good for |
|---|---|---|---|
| `t2.micro` / `t3.micro` | 1–2 | 1 GB | Learning, small apps (free tier) |
| `t3.small` | 2 | 2 GB | Small real apps |
| `t3.medium` | 2 | 4 GB | Bigger apps |

### What is a key pair (`.pem` file)?
Your **password file** for the server. AWS keeps the lock, you keep the key.
**Lose it = can't log in. Share it = anyone can log in.**

### What is a Security Group?
A **firewall** that decides which ports are open.

| Port | Used for | Why we opened it |
|---|---|---|
| **22** | SSH | So we can log in from the terminal |
| **80** | HTTP | So people can open the website |
| **443** | HTTPS | Only needed if you add SSL later |

Port **8000** (backend) is **not** opened. Only Nginx talks to it, from inside the server.

### What is SSH?
**Secure Shell**, a way to **control the server from your terminal**.

```bash
ssh -i inventory-key.pem ec2-user@YOUR_IP
```
= "Log in as **ec2-user** to the computer at **YOUR_IP** using this **key**."

### What is the difference between public IP and private IP?
- **Public IP**: the internet uses it to reach your server (put it in the browser).
- **Private IP**: only used inside AWS (`172.31.x.x`).

### Why does the IP change after I stop and start?
AWS gives a **new public IP every time the instance starts**.
To keep the same IP forever, use an **Elastic IP** (a fixed IP).

---

## 3. How our app runs

### How does the whole thing work?

```
Browser ── http://YOUR_IP ──► Nginx (port 80)
                                ├── /api/...      → backend on port 8000 (PM2)
                                └── anything else → frontend/dist (React files)
```

1. The user opens `http://YOUR_IP`.
2. **Nginx** gets the request on port 80.
3. If the URL starts with `/api`, Nginx sends it to the **Node backend** on port 8000.
4. Anything else, Nginx sends the **React files** from `frontend/dist`.

> 🧠 Remember: **Nginx is the front door. React is the house. Node is the kitchen.**

### How is the frontend running?
It is **not "running"** like a server. `npm run build` turns React into **plain HTML, CSS and JS files** in the `dist` folder.
**Nginx just hands those files** to the browser.

### How is the backend running?
**PM2** runs `node index.js` in the background on **port 8000**.
It keeps running after you close the terminal and **restarts it if it crashes**.

### What is Nginx?
A **web server** + **reverse proxy**.
- **Web server**: gives out files (our React build).
- **Reverse proxy**: passes requests to another program (our Node backend).

### What is a reverse proxy?
A **middle man**. The browser only talks to Nginx, and Nginx talks to the backend for you.
The user never sees port 8000.

### What is PM2?
A **process manager for Node.js**. Think of it as a **babysitter for your backend**:
- runs it in the background
- restarts it if it crashes
- starts it again after a reboot (`pm2 startup` + `pm2 save`)

### Why not just run `node index.js`?
Because it **stops when you close the terminal**, and doesn't come back if it crashes.

### Why do all backend routes start with `/api`?
So Nginx can tell them apart with **one rule**:
`/api/...` → backend, everything else → frontend.

### Why does `/read` or `/create` load React and not show "404"?
React has its own pages (React Router). The rule `try_files $uri /index.html` means:
"If the file doesn't exist, send `index.html` and let React show the right page."

### Why `http` and not `https`?
We have **no SSL certificate**. For HTTPS you need a **domain name** + a free certificate from **Let's Encrypt** (using a tool called `certbot`).

---

## 4. Commands we used and why

| Command | What it does | Why |
|---|---|---|
| `chmod 400 inventory-key.pem` | Only you can read the key | SSH refuses keys others can read |
| `sudo` | Run as admin | Installing software needs admin |
| `dnf install` | Install software on Amazon Linux | Like an app store for Linux |
| `git clone ... hosting` | Download your code into a folder `hosting` | Nginx expects that path |
| `nano .env` | Create the secrets file | `.env` is not on GitHub |
| `npm ci` | Install exact packages from `package-lock.json` | Same versions as your laptop; cleaner than `npm install` |
| `pm2 start index.js --name backend` | Run the backend in the background | Keeps it alive |
| `pm2 save` + `pm2 startup` | Remember the apps and start them on boot | Backend comes back after a restart |
| `npm run build` | Turn React into files in `dist` | Nginx serves these files |
| `chmod 755 /home/ec2-user` | Let other users enter your folder | Nginx must read `dist` |
| `sudo nginx -t` | Test the Nginx config | Catch mistakes before reloading |
| `sudo nginx -s reload` | Apply the new config | No downtime |

### Why is `.env` not on GitHub?
It has **secrets** (database password, API keys). If it's public, anyone can use your database.

### What does `client_max_body_size 6M` do?
Lets users **upload files up to about 5 MB**. Nginx allows only 1 MB by default.

---

## 5. Money: cost, start and stop

> ⚠️ Prices are **approximate** (US regions) and change over time. Always check the **AWS pricing page** and your **Billing dashboard**.

### What costs money on EC2?

| Thing | Charged when | Rough cost |
|---|---|---|
| **Instance** (`t3.micro`) | Only while **running** | about **$0.01/hour** (~$7–8/month if always on) |
| **Disk** (EBS, 8 GB) | **Always**, even when stopped | about **$0.64/month** |
| **Public IPv4** | While attached to a running instance | about **$0.005/hour** (~$3.6/month) |
| **Elastic IP** | When **not** attached to a running instance | small hourly fee |
| **Data out to internet** | After the free amount | very small for a hobby app |

> 🧠 Remember: **Running = pay for computer. Stopped = pay only for the disk. Terminated = pay nothing.**

### What is the Free Tier?
AWS gives new accounts **free usage or free credits** for a limited time (for example small `t2.micro`/`t3.micro` instances).
The rules depend on **when you made your account**, so check **Billing → Free Tier** in the console.

### Stop vs Terminate vs Reboot?

| Action | What happens | Data | Cost after |
|---|---|---|---|
| **Reboot** | Restart the computer | ✅ Kept | Same |
| **Stop** | Turn it off | ✅ Kept (disk stays) | Only disk |
| **Terminate** | **Delete** the server | ❌ **Gone forever** | Nothing |

> 🧠 Remember: **Stop = sleep. Terminate = delete.**

### When should I start and stop the instance?
- **Start** when you are working on it or showing it to someone.
- **Stop** when you are done for the day, to save money.
- **Terminate** when the project is **finished** and you don't need it.
- In a **lab account**, the session ends after **up to 3 hours** and the instance stops by itself. Just start it again (see `hosting.txt`).

### How do I avoid a surprise bill?
1. **Billing → Budgets → Create budget** → set it to **$5**. AWS emails you before you go over.
2. **Stop** instances you are not using.
3. **Terminate** old instances you don't need.
4. Check **Billing → Bills** once a week.
5. Check **every region**: a forgotten server in another region still costs money.

### End-of-day checklist
- [ ] Instance **stopped** (or terminated if finished)
- [ ] No extra instances running in other regions
- [ ] No unused **Elastic IPs**
- [ ] Budget alert is on

---

## 6. CI/CD pipeline

### What is a pipeline?
A **list of steps that run automatically, one after another**, every time you push code.

```
git push ──► Test ──► Build ──► Deploy ──► Website updated ✅
```

### What is CI?
**Continuous Integration**: every push is **automatically tested and built**, to catch errors early.

### What is CD?
**Continuous Delivery / Deployment**: after the build passes, the code is **automatically put on the server**.

> 🧠 Remember: **CI = check the code. CD = ship the code.**

### What do we do now without CI/CD?
We deploy **by hand**: SSH in, then `git pull`, `npm ci`, `npm run build`, `pm2 restart`.
It works, but it's slow and easy to forget a step.

### What would a CI/CD pipeline look like for our app?

```
You push to GitHub
      │
      ▼
GitHub Actions starts
      │
      ├── 1. Install packages (npm ci)
      ├── 2. Run tests / build (npm run build)
      └── 3. SSH into EC2 and run:
             git pull → npm ci → npm run build → pm2 restart backend
      │
      ▼
Website is updated automatically ✅
```

### Which tools are used for CI/CD?
- **GitHub Actions**: built into GitHub, easiest to start with
- **AWS CodePipeline**: AWS's own pipeline tool
- **Jenkins / GitLab CI**: other popular options

### Why use CI/CD?
- **Faster**: deploy with just `git push`
- **Safer**: tests run before going live
- **Same steps every time**: no forgotten commands

### What are the stages of a pipeline?

| Stage | What happens | Our app |
|---|---|---|
| **Source** | Code is pushed to GitHub | `git push` |
| **Build** | Install packages and build | `npm ci`, `npm run build` |
| **Test** | Run tests, check for errors | `npm test` (if you have tests) |
| **Deploy** | Put the new code on the server | `git pull`, `pm2 restart backend` on EC2 |
| **Monitor** | Watch that it works | `pm2 logs`, open the website |

> 🧠 Remember the order: **Source → Build → Test → Deploy → Monitor** ("**S**ome **B**oys **T**ake **D**eep **M**aths").

### What if the pipeline fails?
It **stops at the failed step** and doesn't deploy. GitHub shows a **red ❌** and emails you.
Your live website stays on the **old working version**.

### What is a rollback?
Going **back to the last working version** when a new deploy breaks something.
Simple way: `git checkout <old-commit>` on the server, then rebuild and restart.

### Continuous Delivery vs Continuous Deployment?
- **Delivery**: the pipeline gets everything ready, and a **human clicks "deploy"**.
- **Deployment**: it goes live **automatically**, with no click.

---

## 7. DevOps

### What is DevOps?
**Dev** (developers who write code) + **Ops** (operations people who run servers) **working together**.
It's a way of working where you **build, test, deploy and monitor quickly and automatically**.

> 🧠 Remember: **DevOps = Developers + Operations = ship fast, ship safe.**

### What is the DevOps loop?

```
  Plan ─► Code ─► Build ─► Test
   ▲                        │
   │                        ▼
 Monitor                 Release
   ▲                        │
   │                        ▼
 Operate ◄────────────── Deploy
```

It never ends: you keep improving the app in small steps.

### Which tools are used in DevOps?

| Stage | Tools |
|---|---|
| Plan | Jira, Trello, GitHub Issues |
| Code | Git, GitHub, VS Code |
| Build / Test | npm, GitHub Actions, Jenkins |
| Package | **Docker** |
| Deploy | GitHub Actions, AWS CodePipeline |
| Infrastructure | **Terraform**, AWS CloudFormation (IaC) |
| Run | AWS EC2, Kubernetes |
| Monitor | AWS CloudWatch, PM2 logs, Grafana |

### What does a DevOps engineer do?
Builds the **road from code to server**: sets up servers, pipelines, Docker, monitoring and security,
so developers can just `git push` and the app goes live.

### What DevOps did WE do in this project?
- Rented a server (**EC2**)
- Set up a **web server** (Nginx)
- Kept the app alive with a **process manager** (PM2)
- Wrote **deploy steps** (`hosting.txt`)
- Next step: automate them with a **CI/CD pipeline**

---

## 8. IaaS and IaC

They sound the same but are **different things**.

| | **IaaS** | **IaC** |
|---|---|---|
| Full name | Infrastructure **as a Service** | Infrastructure **as Code** |
| Meaning | **Renting** servers from a cloud | **Writing** your servers in a code file |
| Example | AWS EC2, Azure VM, Google Compute Engine | Terraform, AWS CloudFormation, Ansible |
| In our project | ✅ We used EC2 | ❌ We clicked in the AWS Console instead |

> 🧠 Remember: **IaaS = rent it. IaC = write it.**

### What is IaaS?
The cloud gives you **raw infrastructure** (a computer, disk, network). **You** install and manage the rest.
EC2 is IaaS: AWS gave us an empty Linux server, and **we** installed Node, PM2 and Nginx.
(See also IaaS vs PaaS vs SaaS in [section 1](#what-are-iaas-paas-and-saas).)

### What is IaC?
Instead of **clicking** in the AWS Console to create a server, you **write a file** that describes it.
A tool reads the file and **creates everything for you**.

Example with **Terraform** (creates our EC2 server):

```hcl
resource "aws_instance" "inventory_server" {
  ami           = "ami-xxxxxxxx"   # Amazon Linux 2023
  instance_type = "t3.micro"
  key_name      = "inventory-key"

  tags = {
    Name = "inventory-server"
  }
}
```

```bash
terraform init      # download the AWS plugin
terraform plan      # show what will be created
terraform apply     # create it
terraform destroy   # delete it all
```

### Why use IaC?
- **Repeatable**: make the same server again in 1 command
- **No forgetting**: every setting is written down
- **Saved in Git**: you can see who changed what
- **Easy cleanup**: `terraform destroy` removes everything, so no surprise bills

### Popular IaC tools?

| Tool | Notes |
|---|---|
| **Terraform** | Most popular, works with AWS, Azure, Google Cloud |
| **AWS CloudFormation** | AWS's own, uses YAML/JSON |
| **AWS CDK** | Write infrastructure in JavaScript/TypeScript/Python |
| **Ansible** | Mostly for **setting up** servers (installing software) |

---

## 9. Architecture

### What is architecture?
The **big picture** of how all the parts of an app are connected, like a **building plan**.

### What is the architecture of our app?

```
                User (browser)
                       │
                       │  http://YOUR_IP
                       ▼
  ┌─────────────────────────────────────────┐
  │  AWS EC2 instance (Amazon Linux 2023)   │
  │                                         │
  │  Security Group: allow port 22 and 80   │
  │                    │                    │
  │                    ▼                    │
  │             Nginx (port 80)             │
  │          ┌─────────┴─────────┐          │
  │          ▼                   ▼          │
  │    frontend/dist       Node backend     │
  │    (React files)       port 8000 (PM2)  │
  │                              │          │
  └──────────────────────────────┼──────────┘
                                 ▼
                             Database
```

### What is 3-tier architecture?
Splitting an app into **3 layers**:

| Tier | Job | In our app |
|---|---|---|
| **1. Presentation** | What the user sees | React (frontend) |
| **2. Application** | Logic and rules | Node.js (backend) |
| **3. Data** | Stores data | Database |

> 🧠 Remember: **Look → Think → Store.**

### Monolith vs Microservices?
- **Monolith**: the whole backend is **one app** (like ours, `index.js`). Simple, good for small projects.
- **Microservices**: the backend is split into **many small apps** (users, products, payments). Harder, good for big teams.

### Is our architecture good for a big real app?
It's **perfect for learning and small apps**, but everything is on **one server**: if it stops, the whole site is down.
A bigger setup on AWS could look like this:

| Now (simple) | Bigger setup |
|---|---|
| React files on EC2 | React files on **S3 + CloudFront** (fast, global) |
| 1 EC2 server | **Many EC2 servers + Load Balancer** |
| Pictures on the server disk | Pictures on **S3** |
| Database set up by hand | **Amazon RDS / MongoDB Atlas** (managed database) |
| Public IP that changes | **Domain name + HTTPS** |
| Manual deploy | **CI/CD pipeline** |

---

## 10. How we built it (Kasari banayo?)

### Big picture

```
Your laptop  ──git push──►  GitHub  ──git clone──►  AWS EC2 server  ──►  Website live
```

### Step by step, in simple words

| # | What we did | Why |
|---|---|---|
| 1 | Created an **EC2 instance** (Amazon Linux) | To get a computer on the internet |
| 2 | Downloaded the **key (.pem)** and opened ports **22 + 80** | To log in and to show the website |
| 3 | Connected with **SSH** | To control the server from our terminal |
| 4 | Installed **Git, Node.js, PM2, Nginx** | The tools the app needs |
| 5 | **Cloned** the repo from GitHub | To get our code onto the server |
| 6 | Created **`.env`** with `nano` | Secrets are not on GitHub |
| 7 | Started the **backend with PM2** | So it runs 24/7 on port 8000 |
| 8 | **Built** the frontend (`npm run build`) | To make the `dist` files |
| 9 | Set up **Nginx** | To show React and send `/api` to the backend |
| 10 | Opened **`http://YOUR_IP`** | The website is live 🎉 |

> 🧠 Remember in one line: **Server → Connect → Install → Clone → Env → Backend → Frontend → Nginx → Live.**

### Where are the exact commands?
In **`hosting.txt`** / **`hosting.md`** in this repo.

---

## 11. Docker

### What is Docker?
A tool that **packs your app + everything it needs** (Node version, packages, settings) into one box called a **container**.
That box runs **the same way on every computer**.

> 🧠 Remember: **"It works on my machine" → Docker ships your machine.**

### What problem does Docker solve?
Without Docker, the server needs the **same Node version and setup as your laptop**, and we installed it all by hand.
With Docker, you just run the container, and everything is already inside.

### Image vs Container vs Dockerfile?

| Word | Meaning | Like... |
|---|---|---|
| **Dockerfile** | Instructions to build the image | A **recipe** |
| **Image** | The packed app (read-only) | A **cake mix box** |
| **Container** | A running copy of the image | The **cake you baked** |
| **Docker Hub** | Online store for images | An **app store** for images |

> 🧠 Remember: **Dockerfile → (build) → Image → (run) → Container.**

### What would a Dockerfile for our backend look like?

```dockerfile
# Start from a small Linux with Node 24
FROM node:24-alpine
# Work inside the /app folder
WORKDIR /app
# Copy the package list and install packages
COPY package*.json ./
RUN npm ci
# Copy the rest of the code
COPY . .
# The app uses port 8000
EXPOSE 8000
# Start the backend
CMD ["node", "index.js"]
```

```bash
docker build -t backend .                                # make the image
docker run -d -p 8000:8000 --env-file .env backend       # run it as a container
docker ps                                                # see running containers
docker logs <container-id>                               # see the logs
docker stop <container-id>                               # stop it
```

### Docker vs Virtual Machine (EC2)?

| | **Virtual Machine (EC2)** | **Docker container** |
|---|---|---|
| Size | Big (a full OS) | Small (just the app) |
| Starts in | Minutes | Seconds |
| Runs | One whole computer | Many apps on one computer |

They **work together**: you can run Docker **inside** an EC2 instance.

### What is Docker Compose?
A file (`docker-compose.yml`) that starts **many containers together**, for example frontend + backend + database, with **one command**:

```bash
docker compose up -d
```

### Do we need Docker for our project?
**No.** PM2 + Nginx works fine for a small app. Docker becomes useful when you have **many apps**, a **team**, or use **CI/CD and Kubernetes**.

---

## 12. How GitHub Actions works

### What is GitHub Actions?
GitHub's **built-in robot** that runs commands for you **when something happens** in your repo (like a push).
It's the easiest way to build a **CI/CD pipeline**.

### The main words

| Word | Meaning |
|---|---|
| **Workflow** | The whole automation, written in a `.yml` file |
| **Event / Trigger** | **When** it runs (`push`, `pull_request`, a schedule, a button) |
| **Job** | A group of steps that run on one machine |
| **Step** | One command or one action |
| **Runner** | The computer GitHub gives you to run the job (for example Ubuntu) |
| **Action** | A ready-made step someone else wrote (for example `actions/checkout`) |
| **Secret** | A hidden value (key, password) stored in GitHub settings |

> 🧠 Remember: **Event → Workflow → Job → Steps, running on a Runner.**

### How does it work, step by step?

```
1. You push code to GitHub
2. GitHub sees the file .github/workflows/deploy.yml
3. GitHub starts a fresh computer (runner)
4. The runner runs your steps one by one
5. Green ✅ = success, Red ❌ = failed (you get an email)
```

### Where do I put the workflow file?
In your app's repo, at **`.github/workflows/deploy.yml`**. GitHub finds it automatically.

### Example 1: CI (check the frontend builds on every push)

```yaml
name: CI

on: [push]                       # run on every push

jobs:
  build:
    runs-on: ubuntu-latest       # the runner
    steps:
      - uses: actions/checkout@v4          # download the code
      - uses: actions/setup-node@v4        # install Node
        with:
          node-version: 24
      - run: cd frontend && npm ci && npm run build   # build it
```

If the build fails, you see a ❌ **before** anything goes live.

### Example 2: CD (deploy to our EC2 on every push to `main`)

```yaml
name: Deploy to EC2

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy over SSH
        env:
          KEY: ${{ secrets.EC2_KEY }}     # contents of inventory-key.pem
          HOST: ${{ secrets.EC2_HOST }}   # your EC2 public IP
        run: |
          echo "$KEY" > key.pem
          chmod 400 key.pem
          ssh -o StrictHostKeyChecking=no -i key.pem ec2-user@$HOST '
            cd ~/hosting && git pull &&
            cd backend && npm ci && pm2 restart backend &&
            cd ../frontend && npm ci && npm run build
          '
```

It runs the **same update commands** we type by hand, just automatically.

### How do I add the secrets?
GitHub repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**:
- `EC2_KEY` = the full text of `inventory-key.pem`
- `EC2_HOST` = your EC2 public IP

> ⚠️ **Never** put the key or IP directly in the `.yml` file. Always use **secrets**.
> ⚠️ In a lab account the **IP changes** after every restart, so update `EC2_HOST` each time.

### Where do I see if it worked?
GitHub repo → **Actions** tab. Click a run to see every step and its logs.

---

## 13. When something breaks

| Problem | Most likely reason | Fix |
|---|---|---|
| `ssh` times out | Instance stopped, IP changed, or port 22 closed | Start it, use the new IP, check the Security Group |
| Website doesn't open | Port 80 closed or Nginx off | Open HTTP in the Security Group, run `sudo nginx` |
| **502 Bad Gateway** | Backend is down | `pm2 list`, `pm2 logs backend` |
| **403 Forbidden** | Nginx can't read files | `chmod 755 /home/ec2-user` |
| Old site after update | Forgot to rebuild | `npm run build` in `frontend` |
| `npm run build` says **Killed** | Not enough RAM | Use a bigger instance or add swap |

---

## 14. Quick recall table

| Word | One-line meaning |
|---|---|
| **Cloud** | Renting computers over the internet |
| **AWS** | Amazon's cloud |
| **EC2** | A rented virtual computer |
| **Instance** | One running EC2 server |
| **AMI** | The OS template (Amazon Linux) |
| **Instance type** | Size of the server (t3.micro) |
| **Key pair (.pem)** | Your login key, keep it secret |
| **Security Group** | Firewall: which ports are open |
| **SSH** | Control the server from your terminal |
| **Public IP** | The server's internet address (changes on restart) |
| **Elastic IP** | A fixed IP that doesn't change |
| **EBS** | The server's hard disk |
| **Nginx** | Front door: serves React and forwards `/api` |
| **Reverse proxy** | Middle man between the browser and the backend |
| **PM2** | Keeps the Node backend running |
| **`npm run build`** | Turns React into static files (`dist`) |
| **`.env`** | Secret settings, never on GitHub |
| **Port 80** | Website (HTTP) |
| **Port 8000** | Our backend (internal only) |
| **Stop** | Turn off, keep data, pay only for disk |
| **Terminate** | Delete forever |
| **Budget alert** | Email when cost goes over a limit |
| **CI** | Automatically test and build on every push |
| **CD** | Automatically deploy after the build passes |
| **Pipeline** | Automatic steps: test → build → deploy |
| **Rollback** | Go back to the last working version |
| **DevOps** | Developers + Operations: ship fast and safe |
| **IaaS** | Renting servers (EC2) |
| **IaC** | Writing servers as code (Terraform) |
| **Terraform** | The most popular IaC tool |
| **Architecture** | The big picture of how the parts connect |
| **3-tier** | Frontend → Backend → Database |
| **Monolith** | The whole backend in one app |
| **Docker** | Packs the app + everything it needs into a box |
| **Image** | The packed app (the recipe result) |
| **Container** | A running image |
| **Dockerfile** | Instructions to build an image |
| **GitHub Actions** | GitHub's robot that runs your pipeline |
| **Workflow** | The `.yml` file in `.github/workflows/` |
| **Runner** | The computer that runs the workflow |
| **Secret** | A hidden value (key, IP) stored in GitHub |

---

## 15. Key points to remember

If you forget everything else, remember these. ✅

### ☁️ Cloud and AWS
1. **Cloud** = renting computers on the internet. Pay only for what you use.
2. **EC2** = a rented computer. **Instance** = one running EC2.
3. **IaaS** = you rent the server, **you** install everything (that's EC2).
4. **Security Group** = firewall. Open **22** (SSH) and **80** (website).
5. **.pem key** = your login key. **Never lose it, never share it.**

### 🏗️ How our app works
6. **Nginx** = the front door on port **80**.
7. `/api` goes to the **backend** (Node on port **8000**, kept alive by **PM2**).
8. Everything else goes to the **frontend** (React files in `dist`).
9. **`npm run build`** turns React into simple files. Nginx just hands them out.
10. **`.env`** = secrets. **Never put it on GitHub.**

### 🛠️ How we built it
11. **Server → Connect → Install → Clone → Env → Backend → Frontend → Nginx → Live.**
12. Open the site with **`http://`**, not `https://`.

### ⏰ Restart and money
13. The **lab session ends after up to 3 hours**. The instance stops, but your **files stay**.
14. After starting again, the **IP changes**. Use the new IP.
15. Then start the **backend** (`pm2 start`) and the **frontend** (`sudo nginx`).
16. **Running = pay. Stopped = pay only for the disk. Terminated = pay nothing.**
17. **Stop = sleep. Terminate = delete forever.**
18. Always set a **budget alert** (for example $5) and **stop** what you don't use.

### 🔁 CI/CD and DevOps
19. **Pipeline** = automatic steps: **Source → Build → Test → Deploy → Monitor**.
20. **CI = check the code. CD = ship the code.**
21. **DevOps** = developers + operations working together, to ship **fast and safe**.
22. **GitHub Actions** = GitHub's robot. A `.yml` file in `.github/workflows/` runs on every push.
23. Put keys and IPs in **GitHub Secrets**, never in the code.

### 📦 Docker and IaC
24. **Docker** = packs your app + everything it needs into a **container**.
25. **Dockerfile → Image → Container** (recipe → cake mix → cake).
26. **IaC** = writing servers as code (**Terraform**). **IaaS = rent it. IaC = write it.**

### 🚑 When it breaks
27. **502** = backend is down → `pm2 logs backend`.
28. **Can't connect** = instance stopped or IP changed.
29. **Site not opening** = port 80 closed or Nginx is off.

> 🧠 **One-line summary:**
> **We rented a computer on AWS (EC2), put our code on it from GitHub, ran the backend with PM2,
> built the frontend, and used Nginx as the front door, so anyone can open our app at `http://YOUR_IP`.**
