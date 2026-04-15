# IronForge Gym — IIS Deployment Guide

Complete step-by-step guide to hosting your Node.js + PostgreSQL gym website on Windows Server with IIS.

---

## Prerequisites

You will need a Windows Server (2016, 2019, or 2022) or Windows 10/11 with IIS enabled, plus admin access.

---

## Step 1: Install Node.js

1. Download the **LTS** version of Node.js from https://nodejs.org
2. Run the installer — accept defaults and ensure **"Add to PATH"** is checked
3. Verify installation by opening **Command Prompt** (as Administrator):

```
node --version
npm --version
```

Both commands should return version numbers (e.g., `v20.x.x`).

---

## Step 2: Install PostgreSQL

1. Download PostgreSQL from https://www.postgresql.org/download/windows/
2. Run the installer:
   - Choose a password for the `postgres` user (remember this!)
   - Default port: **5432**
   - Keep defaults for everything else
3. Verify by opening **pgAdmin 4** (installed with PostgreSQL) or **SQL Shell (psql)**:

```
psql -U postgres
Password: <your_password>
postgres=# \q
```

---

## Step 3: Install IIS with Required Features

### Enable IIS via Windows Features:

1. Open **Server Manager** → **Add Roles and Features**
   (On Windows 10/11: **Control Panel** → **Programs** → **Turn Windows features on or off**)
2. Enable **Web Server (IIS)** and make sure these sub-features are checked:
   - Web Server → Common HTTP Features → **Static Content, Default Document**
   - Web Server → Application Development → **CGI** (not strictly needed but useful)
   - Management Tools → **IIS Management Console**
3. Click **Install** and wait for completion

### Verify IIS is running:

Open a browser and navigate to `http://localhost`. You should see the IIS default welcome page.

---

## Step 4: Install iisnode

iisnode is the bridge between IIS and Node.js.

1. Download iisnode from: https://github.com/azure/iisnode/releases
   - Pick the installer that matches your system (x64 for most servers)
2. Run the installer
3. After installation, you may need to restart IIS:

```
iisreset
```

---

## Step 5: Install URL Rewrite Module

1. Download from: https://www.iis.net/downloads/microsoft/url-rewrite
2. Run the installer
3. Restart IIS:

```
iisreset
```

---

## Step 6: Deploy the Application Files

1. Copy the entire `gym-website` project folder to your server. A common location is:

```
C:\inetpub\ironforge-gym\
```

2. Your folder structure should look like:

```
C:\inetpub\ironforge-gym\
├── config\
├── middleware\
├── models\
├── public\
│   ├── css\
│   ├── js\
│   └── images\
├── routes\
├── sql\
├── views\
├── .env
├── package.json
├── server.js
└── web.config        ← This is critical for IIS
```

3. Open **Command Prompt as Administrator**, navigate to the project, and install dependencies:

```
cd C:\inetpub\ironforge-gym
npm install --production
```

---

## Step 7: Configure Environment Variables

1. Copy `.env.example` to `.env`:

```
copy .env.example .env
```

2. Edit `.env` with Notepad (or any text editor) and set your values:

```
PORT=3000
NODE_ENV=production
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ironforge_gym
DB_USER=postgres
DB_PASSWORD=YOUR_ACTUAL_POSTGRES_PASSWORD
SESSION_SECRET=generate-a-long-random-string-here
ADMIN_EMAIL=admin@ironforge.com
ADMIN_PASSWORD=YourSecureAdminPassword123!
```

**Tip**: Generate a strong session secret by running:
```
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

---

## Step 8: Initialize the Database

Run these commands from the project directory:

```
cd C:\inetpub\ironforge-gym

# Create database and tables
npm run db:init

# Seed with sample data (trainers, plans, classes)
npm run db:seed
```

You should see success messages for each step.

---

## Step 9: Test the App Locally First

Before configuring IIS, verify the app works standalone:

```
node server.js
```

Open `http://localhost:3000` in a browser. You should see the IronForge website. Press `Ctrl+C` to stop once confirmed.

---

## Step 10: Configure the IIS Website

### A. Open IIS Manager

1. Press `Win + R`, type `inetmgr`, press Enter

### B. Create a New Website

1. In the left panel, expand your server name
2. Right-click **Sites** → **Add Website**
3. Fill in:
   - **Site name**: `IronForge`
   - **Physical path**: `C:\inetpub\ironforge-gym`
   - **Binding**:
     - Type: `http`
     - IP Address: `All Unassigned`
     - Port: `80` (or `8080` if port 80 is in use)
     - Host name: `ironforge.com` (or leave blank for IP-based access)
4. Click **OK**

### C. Stop Default Website (if using port 80)

1. Click on **Default Web Site** in the left panel
2. Click **Stop** in the right panel under Actions

### D. Set Application Pool Identity

1. Click **Application Pools** in the left panel
2. Find the pool named `IronForge` (created automatically)
3. Right-click → **Advanced Settings**
4. Set **Start Mode** to `AlwaysRunning`
5. Under **Process Model → Identity**, click the `...` button
6. Select **LocalSystem** (or create a dedicated service account)
7. Click **OK**

---

## Step 11: Set Folder Permissions

IIS needs read/write access to the project folder:

1. Right-click `C:\inetpub\ironforge-gym` → **Properties** → **Security** tab
2. Click **Edit** → **Add**
3. Type `IIS_IUSRS` and click **Check Names** → **OK**
4. Grant **Read & Execute**, **List folder contents**, **Read** permissions
5. Also add `IUSR` with the same permissions
6. For the `iisnode` log folder, grant **Write** permission as well
7. Click **Apply** → **OK**

---

## Step 12: Configure Firewall

Allow HTTP traffic through Windows Firewall:

```powershell
# Run in PowerShell as Administrator
New-NetFirewallRule -DisplayName "IIS HTTP" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow
New-NetFirewallRule -DisplayName "IIS HTTPS" -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow
```

---

## Step 13: Test Your Deployment

1. Restart IIS:

```
iisreset
```

2. Open a browser and go to:
   - `http://localhost` (if using port 80)
   - `http://localhost:8080` (if using another port)
   - `http://YOUR_SERVER_IP` (from another machine)

You should see the IronForge gym website!

---

## Step 14 (Optional): Enable HTTPS with SSL

### Using a Self-Signed Certificate (for testing):

1. In IIS Manager, select your server in the left panel
2. Double-click **Server Certificates**
3. Click **Create Self-Signed Certificate** → Name it `IronForge SSL` → OK
4. Go to your **IronForge** site → **Bindings** → **Add**
5. Type: `https`, Port: `443`, SSL Certificate: `IronForge SSL`
6. Click **OK**

### Using Let's Encrypt (for production):

1. Install **win-acme** from: https://www.win-acme.com/
2. Run it and follow the prompts to generate a free SSL certificate
3. It auto-configures IIS and sets up automatic renewal

---

## Step 15 (Optional): Set Up as a Windows Service

To ensure the app starts automatically on server reboot, you can configure the Application Pool:

1. In IIS Manager → **Application Pools** → Select `IronForge`
2. **Advanced Settings**:
   - **Start Mode**: `AlwaysRunning`
   - **Idle Timeout**: `0` (never idle out)
   - **Recycling → Regular Time Interval**: `0` (disable recycling)

---

## Troubleshooting

### View iisnode Logs

Check `C:\inetpub\ironforge-gym\iisnode\` for log files if the site doesn't load.

### Common Issues

| Problem | Solution |
|---------|----------|
| 500 Internal Server Error | Check iisnode logs; ensure `.env` is correct and `npm install` was run |
| Cannot connect to database | Verify PostgreSQL is running: `pg_isready` in Command Prompt |
| Static files (CSS/JS) not loading | Check the `web.config` rewrite rules and folder permissions |
| iisnode not found | Reinstall iisnode and run `iisreset` |
| Port already in use | Stop Default Website or use a different port |
| Permission denied | Re-check folder permissions for `IIS_IUSRS` and `IUSR` |

### Useful Commands

```bash
# Check if PostgreSQL is running
pg_isready

# Restart IIS
iisreset

# Check Node.js version
node --version

# View running IIS sites
%systemroot%\system32\inetsrv\appcmd list site

# Test database connection
psql -U postgres -d ironforge_gym -c "SELECT COUNT(*) FROM users;"
```

---

## Project Structure Reference

```
ironforge-gym/
├── config/
│   └── database.js          # PostgreSQL connection pool
├── middleware/
│   └── auth.js              # Authentication & role guards
├── models/
│   ├── User.js              # User CRUD operations
│   ├── MembershipPlan.js    # Plans queries
│   ├── GymClass.js          # Classes & bookings
│   ├── Trainer.js           # Trainer profiles
│   └── Contact.js           # Contact form messages
├── public/
│   ├── css/style.css        # Main stylesheet
│   └── js/main.js           # Frontend JavaScript
├── routes/
│   ├── pages.js             # Public page routes
│   ├── auth.js              # Login / Register / Logout
│   └── admin.js             # Admin dashboard routes
├── sql/
│   ├── schema.sql           # Database table definitions
│   ├── init-db.js           # Database creation script
│   └── seed-db.js           # Sample data seeder
├── views/
│   ├── partials/            # header.ejs, footer.ejs
│   ├── auth/                # login.ejs, register.ejs
│   ├── admin/               # dashboard, members, messages
│   ├── index.ejs            # Home page
│   ├── about.ejs            # About page
│   ├── classes.ejs          # Classes page
│   ├── trainers.ejs         # Trainers page
│   ├── membership.ejs       # Pricing page
│   ├── contact.ejs          # Contact page
│   ├── 404.ejs              # Not found page
│   └── error.ejs            # Server error page
├── .env                     # Environment variables (do not commit!)
├── .env.example             # Template for .env
├── .gitignore
├── package.json
├── server.js                # Express application entry point
└── web.config               # IIS configuration file
```
