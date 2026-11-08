# NetIntel (.NET 8 MVC)

ASP.NET Core MVC network-traffic dashboard: daily per-device traffic reports, app usage, social media users, WAN quotas and device ownership.

## Run

```bash
dotnet run --project NetIntel.Web --launch-profile http
```

Open http://localhost:5297. In Development, an admin account is created on first start using `Seed:AdminEmail` / `Seed:AdminPassword`
in `NetIntel.Web/appsettings.Development.json`. **Change that password after first login** and never ship it to production;
for production, provide `Seed__AdminEmail` / `Seed__AdminPassword` as environment variables (or user-secrets) once, then remove them.

## Database

SQLite by default (`NetIntel.Web/App_Data/netintel.db`, created automatically). To use SQL Server set:

```json
"Database": { "Provider": "SqlServer" },
"ConnectionStrings": { "SqlServer": "Server=...;Database=NetIntel;..." }
```

The schema is created with `EnsureCreated()`; on start-up, tables added in newer versions are created automatically
(`Data/SchemaUpgrader.cs`). New *columns* on existing tables are not migrated — move to EF Core migrations before production.

## Roles

| Role | Access |
|---|---|
| Admin | Everything, including Users & Roles, extra GB / base WAN allocations and alert recalculation |
| IT Staff | Upload / re-process / delete reports, device-owner master, WAN usage readings, acknowledge social alerts |
| Viewer | Dashboard, Social Media Users (view only), WAN & Quota (view only), CSV exports |

Accounts are created by an Admin under **Users & Roles** (`NetIntel:AllowSelfRegistration` is off).

## Features

- **Dashboard** – today by default (or the latest report day), or any From–To range. Each device appears **once** with its usage
  summed across the range and across networks (a laptop seen on Staff LAN and Guest Wi-Fi is one device), sorted by total, highest first.
- **Device table** – 5 columns (Device Name, Employee Name, Download, Upload, Total), split into `LKH2LP` Devices, Mobile Devices and
  Other Devices. Click a row for the app breakdown (bar/pie chart + usage and % per app).
- **MB / GB toggle** in the header switches every traffic figure and chart; the choice is remembered per browser. WAN quotas are always shown in GB.
- **Weekly trend** – Monday→Sunday chart with Total, Download and Upload lines: selected week, or average / total per weekday.
- **Social Media Users** (all roles) – summary of devices over `NetIntel:SocialMediaThresholdMB` (default 1024 MB) of social traffic in the
  selected period, plus daily alerts that Admin / IT Staff can acknowledge. Platform keywords: `NetIntel:AppCategories:Social`.
- **CSV exports** – *Maximized data users* (all devices or one group, largest first) and *Social media users* (all or over-limit only),
  from the device table's **Export** menu and the Social Media Users page.
- **WAN & Quota** (all roles) – WAN 1 (SLT) 5163 GB anytime and WAN 2 1000 GB anytime + 1000 GB night per calendar month.
  Admin / IT Staff record the GB used so far this month from the ISP portal; Admins add **extra GB** top-ups for a month and edit base
  allocations. View WAN 1, WAN 2 or Total; remaining quota also shows on the dashboard.
- **Device Owners** – import an Excel/CSV master with columns **Device Name** (`LKH2LP…`), **Name**, **Department**
  (IP / MAC columns optional). Mapping is re-applied to every stored report whenever the master changes.
- **Uploaded Reports** – file name, report date, upload time, uploader, records parsed, total traffic, status; download, re-process, delete.
  Re-uploading a date replaces it.
- **Inbox folder** – files named `ip_top_apps_*.csv` dropped into `NetIntel.Web/inbox` are ingested every `NetIntel:InboxScanMinutes`.

Device grouping rules (`NetIntel:DeviceGroups`): names starting with `CompanyPrefix` (`LKH2LP`) are company devices; iOS / Android OS or
phone-like names are mobile; everything else is Other. Devices are merged by device name, except phones, which are keyed by MAC because
many share a generic name such as "iPhone".

## CSV formats accepted

Headers are case/spacing-insensitive. The firewall export format is supported directly:
`Network, IP, Device, User, MAC, OS, Upload_MB, Download_MB, Total_MB, App1, App1_MB, App1_Pct, … App5_Pct, Other_MB`
(`-` means empty; `Other_MB` becomes an "Other apps" entry). Also accepted: a `Top Applications` text column
(`YouTube (45%), Teams 120 MB`) or App columns with only `_pct`. The report date comes from `YYYY-MM-DD` in the file name
(or the date picker for a single upload).
