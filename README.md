# 🌐 .NET 9 Web API & ASP.NET Core Web App – IIS Hosting Guide

This document explains how to deploy and host your **.NET 9 Web API** and **ASP.NET Core Web Application** on **IIS (Internet Information Services)** in Windows.  
It covers installation, publishing, configuration, HTTPS setup, and troubleshooting.

---

## 🧭 Quick Setup Plan

1. ✅ Install IIS (GUI or Command Line)
2. ✅ Install the **.NET 9 Hosting Bundle**
3. ✅ Publish your applications
4. ✅ Create Application Pool & Site in IIS
5. ✅ Configure folder permissions
6. ✅ Bind HTTP/HTTPS and test
7. ✅ Troubleshoot if necessary

---

## 1️⃣ Install IIS

### Option A: Windows GUI (Windows 10 / 11 / Server)
1. Open **Control Panel → Programs → Turn Windows features on or off**
2. Enable these options:
   - **Internet Information Services**
   - **Web Management Tools → IIS Management Console**
   - **World Wide Web Services → Common HTTP Features**
   -   (Optional but helpful)    **Application Development Features → CGI, ISAPI Extensions**             and
      **Health and Diagnostics → Request Monitor** and**Security → Request Filtering**
3. Click **OK**, wait for installation, then run `inetmgr` to open IIS Manager.

### Option B: Command Line

**Windows Server (PowerShell as Administrator):**
```powershell
Install-WindowsFeature -name Web-Server -IncludeManagementTools
````

**Windows 10/11 (DISM):**

```powershell
dism /online /enable-feature /featurename:IIS-WebServer /all
```

---

## 2️⃣ Install the .NET 9 Hosting Bundle

Download and install the **ASP.NET Core Hosting Bundle** for .NET 9 from Microsoft:

👉 [Download Hosting Bundle](https://dotnet.microsoft.com/en-us/download/dotnet/9.0)

Run the installer **as Administrator**, then restart IIS:

```powershell
iisreset
```

> 💡 The Hosting Bundle installs the **ASP.NET Core Module (ANCM)** that enables IIS to host .NET Core/9 applications via Kestrel.

---

## 3️⃣ Publish Your Application

You can publish via **CLI** or **Visual Studio**.

### Option A: Framework-dependent (Recommended)

Requires Hosting Bundle on the server.

```bash
dotnet publish -c Release -o "C:\inetpub\wwwroot\MyApi"
```

### Option B: Self-contained (No Hosting Bundle required)

```bash
dotnet publish -c Release -r win-x64 --self-contained true -o "C:\inetpub\wwwroot\MyApi"
```

### Option C: Visual Studio

1. Right-click your project → **Publish**
2. Choose **Folder**
3. Set target path (e.g., `C:\inetpub\wwwroot\MyApi`)
4. Click **Publish**

---

## 4️⃣ Create IIS Application Pool & Website

### Create App Pool

1. Open **IIS Manager**
2. Right-click **Application Pools → Add Application Pool**
3. Name: `MyApiPool`
4. **.NET CLR Version**: `No Managed Code`
5. Click **OK**

### Create Website

1. Right-click **Sites → Add Website**
2. **Site name:** `MyApiSite`
3. **Physical path:** `C:\inetpub\wwwroot\MyApi`
4. **Application pool:** `MyApiPool`
5. **Binding:** Choose IP/Port (e.g., 80, 8071) and hostname (optional)
6. Click **OK**

> 🔁 To host both Web API and Web App:
> Create separate sites with different ports or add one as an **Application** under another.

---

## 5️⃣ Configure Folder Permissions

Grant access to the App Pool identity so IIS can read files.

1. Right-click published folder → **Properties → Security → Edit → Add**
2. Enter:

   ```
   IIS AppPool\MyApiPool
   ```
3. Click **Check Names → OK**
4. Grant **Read & Execute**, **List folder**, **Read**
5. Click **Apply → OK**

> 🧱 If app writes logs/files, grant **Modify** permissions.

---

## 6️⃣ `web.config` Examples

### Framework-dependent

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <handlers>
      <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModuleV2" resourceType="Unspecified" />
    </handlers>
    <aspNetCore processPath="dotnet" arguments=".\MyApi.dll" stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout" hostingModel="inprocess">
      <environmentVariables>
        <environmentVariable name="ASPNETCORE_ENVIRONMENT" value="Production" />
      </environmentVariables>
    </aspNetCore>
  </system.webServer>
</configuration>
```

### Self-contained

```xml
<aspNetCore processPath=".\MyApi.exe" arguments="" stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout" hostingModel="inprocess" />
```

---

## 7️⃣ Enable HTTPS (Recommended)

### Generate a Certificate

In IIS:

1. Select the server node → **Server Certificates**
2. Click **Create Self-Signed Certificate**

### Bind HTTPS

1. Select your site → **Bindings → Add**
2. Type: `https`, Port: `443`
3. Select your certificate

### Redirect HTTP → HTTPS

Option 1 – via Code:

```csharp
app.UseHttpsRedirection();
app.UseHsts();
```

Option 2 – via URL Rewrite (IIS module)

---

## 8️⃣ Allow IIS Through Firewall

```powershell
# Allow HTTP
netsh advfirewall firewall add rule name="IIS HTTP" dir=in action=allow protocol=TCP localport=80

# Allow HTTPS
netsh advfirewall firewall add rule name="IIS HTTPS" dir=in action=allow protocol=TCP localport=443
```

For custom port (e.g. 8071):

```powershell
netsh advfirewall firewall add rule name="IIS Custom Port" dir=in action=allow protocol=TCP localport=8071
```

---

## 9️⃣ Troubleshooting

| Error                                                    | Cause                                  | Fix                                                             |
| -------------------------------------------------------- | -------------------------------------- | --------------------------------------------------------------- |
| **500.19 - Cannot read configuration file (0x80070005)** | IIS lacks permission to access folder  | Grant read permissions to `IIS AppPool\<AppPoolName>`           |
| **500.30 - ANCM In-Process Start Failure**               | App crashed or runtime missing         | Run `dotnet MyApp.dll` manually to see the error                |
| **404 - Not Found**                                      | Wrong binding or path                  | Verify physical path and bindings in IIS                        |
| **Cleartext HTTP not permitted (Android)**               | MAUI app calling HTTP instead of HTTPS | Use HTTPS or configure `network_security_config.xml` (dev only) |

Check detailed logs:

* **Event Viewer → Windows Logs → Application**
* `web.config` → set `stdoutLogEnabled="true"`

Restart IIS after changes:

```powershell
iisreset
```

---

## 🔍 Useful Commands

| Purpose                 | Command                                                   |              |
| ----------------------- | --------------------------------------------------------- | ------------ |
| Restart IIS             | `iisreset`                                                |              |
| List ports              | `netstat -ano                                             | findstr :80` |
| Publish project         | `dotnet publish -c Release -o "C:\inetpub\wwwroot\MyApi"` |              |
| Check installed runtime | `dotnet --list-runtimes`                                  |              |

---

## ✅ Deployment Checklist

* [ ] IIS Installed and running (`inetmgr`)
* [ ] .NET 9 Hosting Bundle installed (unless self-contained)
* [ ] Published folder accessible by IIS
* [ ] App Pool uses “No Managed Code”
* [ ] Folder permissions set (`IIS AppPool\<AppPoolName>`)
* [ ] HTTP/HTTPS bindings configured
* [ ] Firewall allows chosen port
* [ ] HTTPS certificate configured (optional but recommended)

---

## 🧠 Notes

* Always publish to a neutral path (`C:\inetpub\wwwroot` or `D:\WebApps`), not Desktop or Documents.
* Use dedicated App Pools for each site.
* Disable `stdoutLogEnabled` in production.
* Run apps under **Production** environment.

---

### 🛠 Example: Two Apps on IIS

| App Type | Folder Path                   | Port | App Pool       |
| -------- | ----------------------------- | ---- | -------------- |
| Web API  | `C:\inetpub\wwwroot\MyApi`    | 8071 | `MyApiPool`    |
| Web App  | `C:\inetpub\wwwroot\MyWebApp` | 8080 | `MyWebAppPool` |

Access:

* API → [http://localhost:8071](http://localhost:8071)
* Web App → [http://localhost:8080](http://localhost:8080)

---

## 🏁 Done!

Your **.NET 9 Web API** and **Web Application** are now hosted on IIS 🎉
You can access them via `http://<Your-IP>:<Port>` or `https://<Your-Domain>`.

---

> 🧩 *Author: Bhakta Charan Rout*
> *Environment: Windows + IIS + .NET 9 *

```
