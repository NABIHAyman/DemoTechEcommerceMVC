# Deployment guide — TechEcommerce

Step-by-step deployment of the ASP.NET Core MVC application, either on Windows
Server with IIS or on Linux with Kestrel behind Nginx.

> **Reminder.** This is academic work. The guide describes a sound deployment,
> not a critical production setup: no high availability, no monitoring, no
> disaster recovery plan.

---

## Contents

1. [Publishing the application](#1-publishing-the-application)
2. [Database](#2-database)
3. [Configuration](#3-configuration)
4. [Option A — Windows Server with IIS](#4-option-a--windows-server-with-iis)
5. [Option B — Linux with Kestrel behind Nginx](#5-option-b--linux-with-kestrel-behind-nginx)
6. [HTTPS](#6-https)
7. [Verification](#7-verification)
8. [Updating](#8-updating)
9. [Backups](#9-backups)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Publishing the application

On the build machine, with the .NET 9 SDK installed:

```bash
git clone https://github.com/NABIHAyman/DemoTechEcommerceMVC.git
cd DemoTechEcommerceMVC
```

```bash
dotnet restore
dotnet build --configuration Release --no-restore
```

### Framework-dependent publish

Requires the ASP.NET Core 9 runtime on the target server. Smaller output.

```bash
dotnet publish DemoTechEcommerceMVC/DemoTechEcommerceMVC.csproj \
  --configuration Release \
  --output ./publish
```

### Self-contained publish

Ships its own runtime — nothing to install on the server.

```bash
dotnet publish DemoTechEcommerceMVC/DemoTechEcommerceMVC.csproj \
  --configuration Release \
  --runtime linux-x64 \
  --self-contained true \
  --output ./publish
```

Replace `linux-x64` with `win-x64` for Windows.

> The project enables Razor runtime compilation, which is a development
> convenience. It works in production but costs a little startup time and keeps
> the `.cshtml` files on disk. Removing the
> `Microsoft.AspNetCore.Mvc.Razor.RuntimeCompilation` package and the
> `.AddRazorRuntimeCompilation()` call before publishing is the cleaner option.

---

## 2. Database

### Creating the schema

The application ships 21 EF Core migrations. Two ways to apply them.

**From a machine that has the SDK**, pointing at the production server:

```bash
dotnet ef database update \
  --project DemoTechEcommerceMVC \
  --connection "<production connection string>"
```

**By generating a SQL script** — preferable when a DBA reviews changes, or when
the server is unreachable from the build machine:

```bash
dotnet ef migrations script \
  --project DemoTechEcommerceMVC \
  --idempotent \
  --output migrations.sql
```

The `--idempotent` flag makes the script safe to re-run: each migration is
guarded by a check.

### Application account

Do not let the application connect as `sa`. Create a dedicated login:

```sql
CREATE LOGIN app_user WITH PASSWORD = '<strong password>';
CREATE DATABASE TechEcommerceDb;
GO

USE TechEcommerceDb;
CREATE USER app_user FOR LOGIN app_user;
ALTER ROLE db_datareader ADD MEMBER app_user;
ALTER ROLE db_datawriter ADD MEMBER app_user;
GO
```

> `db_datareader` and `db_datawriter` are enough at runtime. Migrations need
> schema rights (`db_ddladmin`): grant them for the migration, then revoke.

---

## 3. Configuration

**Never edit `appsettings.json` on the server.** Configuration comes from
environment variables, which take precedence.

ASP.NET Core maps nesting with a double underscore: `ConnectionStrings:DefaultConnection`
becomes `ConnectionStrings__DefaultConnection`.

| Variable | Value |
|---|---|
| `ASPNETCORE_ENVIRONMENT` | `Production` |
| `ASPNETCORE_URLS` | Addresses Kestrel binds to, e.g. `http://127.0.0.1:5000` |
| `ConnectionStrings__DefaultConnection` | SQL Server connection string with the application account |
| `Logging__LogLevel__Default` | `Warning` is a reasonable production default |

A connection string takes this shape — values omitted on purpose:

```
Server=<host>;Database=<name>;User Id=<user>;Password=<password>;Encrypt=True;TrustServerCertificate=False
```

> `TrustServerCertificate=True`, which the development file uses, disables
> certificate validation. Set it to `False` in production and install a valid
> certificate on SQL Server.

---

## 4. Option A — Windows Server with IIS

### Prerequisites

```powershell
Install-WindowsFeature -Name Web-Server -IncludeManagementTools
```

Then install the **ASP.NET Core 9 Hosting Bundle** from
`https://dotnet.microsoft.com/download/dotnet/9.0` and restart IIS:

```powershell
net stop was /y
net start w3svc
```

### Deploying the files

```powershell
New-Item -ItemType Directory -Path C:\inetpub\techecommerce -Force
Copy-Item -Path .\publish\* -Destination C:\inetpub\techecommerce -Recurse -Force
```

### Creating the site

```powershell
Import-Module WebAdministration

New-WebAppPool -Name "TechEcommercePool"
Set-ItemProperty IIS:\AppPools\TechEcommercePool -Name managedRuntimeVersion -Value ""

New-Website -Name "TechEcommerce" `
            -PhysicalPath "C:\inetpub\techecommerce" `
            -ApplicationPool "TechEcommercePool" `
            -Port 80
```

`managedRuntimeVersion` must be empty: ASP.NET Core runs out of process, the
pool must not load the .NET Framework CLR.

### Environment variables

```powershell
Set-WebConfigurationProperty -PSPath "IIS:\Sites\TechEcommerce" `
  -Filter "system.webServer/aspNetCore/environmentVariables" `
  -Name "." `
  -Value @{name='ASPNETCORE_ENVIRONMENT';value='Production'}
```

Repeat for `ConnectionStrings__DefaultConnection`.

### Folder permissions

```powershell
icacls "C:\inetpub\techecommerce" /grant "IIS AppPool\TechEcommercePool:(OI)(CI)RX"
icacls "C:\inetpub\techecommerce\logs" /grant "IIS AppPool\TechEcommercePool:(OI)(CI)M"
```

Read and execute on the application, write only where logs go.

---

## 5. Option B — Linux with Kestrel behind Nginx

### Installing the runtime

Debian 12 / Ubuntu 22.04, framework-dependent publish:

```bash
sudo apt update
sudo apt install -y aspnetcore-runtime-9.0 nginx
```

A self-contained publish needs no runtime package.

### Deploying the files

```bash
sudo mkdir -p /var/www/techecommerce
sudo cp -r ./publish/* /var/www/techecommerce/
sudo chown -R www-data:www-data /var/www/techecommerce
```

### systemd unit

```bash
sudo nano /etc/systemd/system/techecommerce.service
```

```ini
[Unit]
Description=TechEcommerce ASP.NET Core application
After=network.target

[Service]
WorkingDirectory=/var/www/techecommerce
ExecStart=/usr/bin/dotnet /var/www/techecommerce/DemoTechEcommerceMVC.dll
Restart=always
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier=techecommerce
User=www-data

Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=ASPNETCORE_URLS=http://127.0.0.1:5000
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false
EnvironmentFile=/etc/techecommerce.env

[Install]
WantedBy=multi-user.target
```

Keep the connection string out of the unit file, which is world-readable:

```bash
sudo nano /etc/techecommerce.env
```

```
ConnectionStrings__DefaultConnection=<connection string>
```

```bash
sudo chmod 600 /etc/techecommerce.env
sudo chown root:root /etc/techecommerce.env
```

For a **self-contained** publish, replace `ExecStart` with the native binary:

```ini
ExecStart=/var/www/techecommerce/DemoTechEcommerceMVC
```

Enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable techecommerce
sudo systemctl start techecommerce
sudo systemctl status techecommerce
```

### Nginx reverse proxy

```bash
sudo nano /etc/nginx/sites-available/techecommerce
```

```nginx
server {
    listen 80;
    server_name example.tld www.example.tld;

    client_max_body_size 10M;

    location / {
        proxy_pass         http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection keep-alive;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/techecommerce /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

> Behind a proxy, the application must trust the forwarded headers, otherwise
> `Program.cs` sees plain HTTP and `UseHttpsRedirection()` loops. Add
> `app.UseForwardedHeaders()` with `ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto`
> **before** `UseHttpsRedirection()`.

### Firewall

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

Port 5000 must stay bound to `127.0.0.1` and never be opened.

---

## 6. HTTPS

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d example.tld -d www.example.tld
sudo certbot renew --dry-run
```

On IIS, bind the certificate through the management console or:

```powershell
New-WebBinding -Name "TechEcommerce" -Protocol https -Port 443
```

The application already calls `UseHsts()` outside the development environment.

---

## 7. Verification

```bash
sudo systemctl status techecommerce
curl -I http://127.0.0.1:5000
sudo journalctl -u techecommerce -f --lines=100
```

### Checklist

- [ ] The site answers over HTTPS on the public domain
- [ ] `ASPNETCORE_ENVIRONMENT=Production`
- [ ] The developer exception page never appears
- [ ] The connection string uses the application account, not `sa`
- [ ] `TrustServerCertificate=False`
- [ ] SQL Server is not reachable from the internet
- [ ] `/etc/techecommerce.env` is `chmod 600`
- [ ] All 21 migrations applied — check the `__EFMigrationsHistory` table
- [ ] An administrator account exists and `/Dashboard` is reachable

---

## 8. Updating

```bash
git pull origin master
dotnet publish DemoTechEcommerceMVC/DemoTechEcommerceMVC.csproj -c Release -o ./publish
```

```bash
sudo systemctl stop techecommerce
sudo cp -r ./publish/* /var/www/techecommerce/
sudo chown -R www-data:www-data /var/www/techecommerce
```

Apply any new migration, then:

```bash
sudo systemctl start techecommerce
```

On IIS, dropping an `app_offline.htm` file at the site root stops the
application cleanly; remove it once the copy is done.

---

## 9. Backups

```bash
sqlcmd -S <server> -U <user> -Q "BACKUP DATABASE TechEcommerceDb TO DISK = '/var/opt/mssql/backup/techecommerce-$(date +%F).bak' WITH FORMAT, COMPRESSION"
```

Restore:

```sql
RESTORE DATABASE TechEcommerceDb
FROM DISK = '/var/opt/mssql/backup/techecommerce-2026-09-19.bak'
WITH REPLACE;
```

Back up `/etc/techecommerce.env` separately, and store it encrypted.

---

## 10. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| HTTP 500.30 on IIS | The application fails at startup, usually a bad connection string | Read the Windows event log, section *Application* |
| HTTP 502.5 on IIS | Runtime missing, or wrong `processPath` | Install the Hosting Bundle, check `web.config` |
| Endless redirect loop | `UseHttpsRedirection()` behind a proxy that does not forward the scheme | Add `UseForwardedHeaders()` before it |
| `Login failed for user` | Wrong credentials, or login not mapped in the database | Replay section 2 |
| `A network-related error occurred` | SQL Server unreachable, or TCP/IP disabled | Enable TCP/IP in SQL Server Configuration Manager, open port 1433 internally |
| Missing CSS and JS | `wwwroot` not copied on publish | Check the contents of `./publish/wwwroot` |
| `Pending model changes` | A migration exists in code but not in the database | `dotnet ef database update` |
| Service restarting in a loop | Startup exception | `journalctl -u techecommerce -n 200` |

---

## Author

**Ayman NABIH**
[github.com/NABIHAyman](https://github.com/NABIHAyman) ·
[linkedin.com/in/nabihayman](https://linkedin.com/in/nabihayman)
