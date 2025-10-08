# 🌐 .NET 9 Web API & ASP.NET Core Web App – IIS Hosting Guide

This document explains how to deploy and host your **.NET 9 Web API** and **ASP.NET Core Web Application** on **IIS (Internet Information Services)** in Windows.  
It covers installation, publishing, configuration, HTTPS setup, and troubleshooting.

---

# Quick plan

1. Install IIS (GUI or PowerShell/DISM).
2. Install the .NET 9 **Hosting Bundle** (adds ASP.NET Core Module for IIS).
3. Publish your apps (framework-dependent or self-contained).
4. Create App Pool + Site in IIS and point to published folder.
5. Configure permissions, bindings (HTTP/HTTPS) and test.
6. Troubleshoot if something breaks.

---

# 1) Install IIS (Windows GUI or Command line)

**GUI (Windows 10 / 11 / Server with Desktop):**

1. Open **Control Panel → Programs → Turn Windows features on or off**.
2. Check **Internet Information Services**. Under it make sure:

   * **Web Management Tools → IIS Management Console** is checked.
   * **World Wide Web Services → Common HTTP Features** (Static Content, Default Document, HTTP Errors) is checked.
   * (Optional but useful) **Application Development Features** (CGI, ISAPI Extensions) and **Health and Diagnostics → Request Monitor** and **Security → Request Filtering**.
3. Click **OK** and wait for installation to finish.
4. Start IIS Manager: press `Win + R`, type `inetmgr` and Enter.

**PowerShell / DISM (headless or scriptable):**

* On **Windows Server** (run PowerShell as Administrator):

```powershell
Install-WindowsFeature -name Web-Server -IncludeManagementTools
````

* On **Windows 10/11** (use DISM in an elevated cmd/powershell):

```powershell
dism /online /enable-feature /featurename:IIS-WebServer /all
```

---

# 2) Install the .NET 9 Hosting Bundle (required for IIS hosting, unless you publish self-contained)

* Download and **run the ASP.NET Core Hosting Bundle** installer that matches your .NET 9 runtime (from Microsoft). Run the installer **as Administrator**.
* After install: run `iisreset` (or restart machine).

**Why:** the Hosting Bundle installs the .NET runtime (if needed) and the **ASP.NET Core Module (ANCM)** which lets IIS act as a reverse proxy to Kestrel.

> Alternative: Publish **self-contained** (see publish section). That removes the need for the hosting bundle on the server.

---

# 3) Publish your app (two common approaches)

**A. Framework-dependent (recommended, smaller):**
From your project folder:

```bash
dotnet publish -c Release -o "C:\inetpub\wwwroot\MyApi"
```

This produces `MyApi.dll` and supporting files. IIS + Hosting Bundle must be present.

**B. Self-contained (no Hosting Bundle required):**

```bash
dotnet publish -c Release -r win-x64 --self-contained true -o "C:\inetpub\wwwroot\MyApi"
```

This produces an exe (MyApi.exe) and everything bundled — heavier but portable.

**Visual Studio:**

* Right-click project → **Publish** → choose **Folder** target and set the output folder (e.g. `C:\inetpub\wwwroot\MyApi`). Click Publish.

---

# 4) Create App Pool and Website in IIS

1. Open **IIS Manager (inetmgr)**.
2. **App Pool**:

   * Right click **Application Pools → Add Application Pool**.
   * Name: `MyApiPool`. **.NET CLR version** = **No Managed Code**. (ASP.NET Core runs under the .NET Core runtime, not the .NET Framework CLR.)
   * Managed pipeline: default.
3. **Create Site**:

   * Right click **Sites → Add Website**.
   * **Site name**: `MyApiSite`.
   * **Physical path**: `C:\inetpub\wwwroot\MyApi` (published folder).
   * **Application pool**: choose `MyApiPool`.
   * **Binding**: choose IP (All Unassigned or specific IP), port (80 for http or 8071 if you want), hostname if using host header. Click OK.

If you need to host both API and web app: create two sites on different bindings (different ports or hostnames), or create one site and add the other app as an **Application** under it.

---

# 5) Folder permissions (important)

Give the app pool identity read (and write if you need logging) access to the folder:

1. Right click published folder → Properties → Security → Edit → Add.
2. In object name box type: `IIS AppPool\MyApiPool` and click **Check Names** then OK.
3. Grant **Read & execute, List folder, Read** (or Modify if app writes files/logs).

---

# 6) `web.config` (IIS + ANCM) — examples

**Framework-dependent (typical)**

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

**Self-contained (processPath points to exe):**

```xml
<aspNetCore processPath=".\MyApi.exe" arguments="" stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout" hostingModel="inprocess" />
```

> Visual Studio publish usually generates `web.config` for you. Set `stdoutLogEnabled="true"` only while debugging; create the `logs` folder and give IIS write permission if you enable it.

---

# 7) HTTPS — certificates & redirect from HTTP to HTTPS

**Get a cert**

* For production: obtain certificate from a CA (or use Let’s Encrypt with tools like win-acme).
* For quick testing: in IIS Manager → Server Certificates → **Create Self-Signed Certificate** (not for production).

**Bind cert**

* In IIS Manager → Sites → select site → **Bindings → Add** → Type `https`, port `443`, select certificate.

**Redirect HTTP → HTTPS**

* Option A: Add middleware in your app (`app.UseHttpsRedirection()` and `app.UseHsts()` in `Program.cs`), then ensure IIS forwards requests (default).
* Option B: Use **URL Rewrite** module in IIS and create a rule to redirect http to https (needs URL Rewrite module installed).

---

# 8) Windows Firewall & external access

If you want the site accessible from other machines:

```powershell
# allow HTTP
netsh advfirewall firewall add rule name="IIS HTTP" dir=in action=allow protocol=TCP localport=80

# allow HTTPS
netsh advfirewall firewall add rule name="IIS HTTPS" dir=in action=allow protocol=TCP localport=443
```

# 9)🚑 Troubleshooting & Deployment Guide for .NET 9 Web API / Web App on IIS

## 9️⃣ Troubleshooting

| **Error** | **Cause** | **Fix** |
|------------|-----------|----------|
| `500.19 - Cannot read configuration file (0x80070005)` | IIS lacks permission to access folder | Grant read permissions to `IIS AppPool\<AppPoolName>` |
| `500.30 - ANCM In-Process Start Failure` | App crashed or runtime missing | Run `dotnet MyApp.dll` manually to see the error |
| `404 - Not Found` | Wrong binding or physical path | Verify physical path and bindings in IIS |
| `Cleartext HTTP not permitted (Android)` | MAUI app calling HTTP instead of HTTPS | Use **HTTPS** or configure `network_security_config.xml` (for development only) |

---

### 🔍 Check Detailed Logs

- Open **Event Viewer → Windows Logs → Application**
- In your app’s `web.config`, temporarily enable logging:
  ```xml
  <aspNetCore ... stdoutLogEnabled="true" stdoutLogFile=".\logs\stdout" />
````

* Restart IIS after any change:

  ```powershell
  iisreset
  ```

---

## 🔧 Useful Commands

| **Purpose**              | **Command**                                               |
| ------------------------ | --------------------------------------------------------- |
| Restart IIS              | `iisreset`                                                |
| List all listening ports | `netstat -ano`                                            |
| Publish project          | `dotnet publish -c Release -o "C:\inetpub\wwwroot\MyApi"` |
| Check installed runtimes | `dotnet --list-runtimes`                                  |

---

## ✅ Deployment Checklist

* [x] IIS Installed and running (`inetmgr`)
* [x] .NET 9 Hosting Bundle installed (unless published self-contained)
* [x] Published folder accessible by IIS
* [x] Application Pool uses **No Managed Code**
* [x] Folder permissions set for `IIS AppPool\<AppPoolName>`
* [x] HTTP/HTTPS bindings configured properly
* [x] Firewall allows chosen port (80 / 443 / custom)
* [x] HTTPS certificate configured (recommended for production)

---

## 🧠 Notes

* Always publish to a neutral path (C:\inetpub\wwwroot or D:\WebApps), not Desktop or Documents.

* Use dedicated App Pools for each site.

* Disable stdoutLogEnabled in production.

* Run apps under Production environment.

* Ensure `ASPNETCORE_ENVIRONMENT` = **Production** in your config

---

## 🛠 Example: Two Apps on IIS

| **App Type** | **Folder Path**               | **Port** | **App Pool**   |
| ------------ | ----------------------------- | -------- | -------------- |
| Web API      | `C:\inetpub\wwwroot\MyApi`    | `8071`   | `MyApiPool`    |
| Web App      | `C:\inetpub\wwwroot\MyWebApp` | `8080`   | `MyWebAppPool` |

**Access URLs:**

* API → [http://localhost:8071](http://localhost:8071)
* Web App → [http://localhost:8080](http://localhost:8080)

---

## 🏁 Done!

🎉 Your **.NET 9 Web API** and **ASP.NET Core Web Application** are now successfully hosted on IIS.
You can access them via:

```
http://<Your-IP>:<Port>
https://<Your-Domain>
```

---

### 🧩 Author

**Name:** Bhakta Charan Rout
**Environment:** Windows + IIS + .NET 9 + SQL Server

```

