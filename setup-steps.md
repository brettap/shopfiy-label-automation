Here’s a Git-ready KB draft you can save as:

`kb/projects/label-printing-agent.md`

# Label Printing Agent — Shopify to Polono Label Printer

## Purpose

Build a local **Label Printing Agent** that simplifies shipping label printing from Shopify orders.

The first version will not fully automate Shopify. Instead, it will watch a local folder for label PDF files and automatically print them to the Polono label printer. Later versions can connect to Shopify webhooks and a shipping label provider.

---

## Target Workflow

### Phase 1 — Local Folder Watcher

```text
Label PDF downloaded → saved to C:\Labels\ToPrint → agent prints label → file moves to Printed
```

### Phase 2 — Shopify Assisted Workflow

```text
Shopify order received → shipping label created → label PDF saved locally → agent prints label
```

### Phase 3 — Full Automation

```text
Shopify order webhook → agent receives order → shipping API creates label → label prints → Shopify order marked fulfilled
```

Shopify supports order webhooks such as `orders/create`, and newer Shopify app work should use the GraphQL Admin API because the REST Admin API is legacy for new public apps.
Sources: Shopify Webhooks, Shopify REST Admin API legacy notice.

---

## Lab Requirements

### Hardware

* Lab PC
* Polono thermal label printer
* USB connection or network print access
* Internet access
* Shopify admin access

### Software

* Windows 10/11
* Polono printer driver installed
* Python 3 or PowerShell
* PDF reader/print handler
* Git
* VS Code

---

## Folder Structure

Create these folders on the lab PC:

```powershell
mkdir C:\Labels
mkdir C:\Labels\ToPrint
mkdir C:\Labels\Printed
mkdir C:\Labels\Failed
mkdir C:\Labels\Logs
```

Recommended project repo structure:

```text
label-printing-agent/
│
├── README.md
├── docs/
│   ├── workflow.md
│   ├── setup-windows.md
│   └── troubleshooting.md
│
├── scripts/
│   ├── watch-label-folder.ps1
│   └── print-test-label.ps1
│
├── config/
│   └── config.example.json
│
└── logs/
    └── .gitkeep
```

---

## Phase 1 Build — Folder Watcher

### Goal

Automatically print any PDF label placed in:

```text
C:\Labels\ToPrint
```

After printing:

* Successful print → move to `C:\Labels\Printed`
* Failed print → move to `C:\Labels\Failed`
* Write log entry to `C:\Labels\Logs`

---

## PowerShell Starter Script

Save as:

```text
scripts\watch-label-folder.ps1
```

```powershell
$WatchFolder = "C:\Labels\ToPrint"
$PrintedFolder = "C:\Labels\Printed"
$FailedFolder = "C:\Labels\Failed"
$LogFile = "C:\Labels\Logs\label-agent.log"
$PrinterName = "POLONO"

function Write-Log {
    param([string]$Message)
    $Timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    "$Timestamp - $Message" | Out-File -FilePath $LogFile -Append
}

Write-Log "Label Printing Agent started."

while ($true) {
    $Files = Get-ChildItem -Path $WatchFolder -Filter *.pdf -File

    foreach ($File in $Files) {
        try {
            Write-Log "Found label: $($File.FullName)"

            Start-Process -FilePath $File.FullName -Verb Print -PassThru | Out-Null

            Start-Sleep -Seconds 5

            Move-Item -Path $File.FullName -Destination $PrintedFolder -Force

            Write-Log "Printed and moved to Printed: $($File.Name)"
        }
        catch {
            Write-Log "FAILED printing $($File.Name): $($_.Exception.Message)"

            Move-Item -Path $File.FullName -Destination $FailedFolder -Force
        }
    }

    Start-Sleep -Seconds 3
}
```

---

## Important Note About Printer Name

Run this command to list installed printers:

```powershell
Get-Printer | Select-Object Name, DriverName, PortName
```

Update this line if needed:

```powershell
$PrinterName = "POLONO"
```

The basic `Start-Process -Verb Print` method may use the Windows default printer. For best results, set the Polono printer as the default printer on the lab PC.

---

## Manual Test

1. Download or create a test shipping label PDF.
2. Save it into:

```text
C:\Labels\ToPrint
```

3. Confirm:

   * Label prints
   * File moves to `Printed`
   * Log file updates

View logs:

```powershell
Get-Content C:\Labels\Logs\label-agent.log -Tail 20
```

---

## Optional: Run Agent at Startup

Create a scheduled task:

```powershell
$Action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-ExecutionPolicy Bypass -File C:\Path\To\label-printing-agent\scripts\watch-label-folder.ps1"

$Trigger = New-ScheduledTaskTrigger -AtStartup

Register-ScheduledTask -TaskName "Label Printing Agent" -Action $Action -Trigger $Trigger -RunLevel Highest -Description "Watches label folder and prints shipping labels"
```

Verify task:

```powershell
Get-ScheduledTask -TaskName "Label Printing Agent"
```

Start task manually:

```powershell
Start-ScheduledTask -TaskName "Label Printing Agent"
```

---

## Phase 2 — Shopify Assisted Workflow

This phase keeps label creation mostly manual but improves the process.

### Workflow

```text
Shopify order arrives
↓
Create shipping label in Shopify, Pirate Ship, ShipStation, Shippo, EasyPost, UPS, USPS, or FedEx
↓
Download label as PDF
↓
Save PDF to C:\Labels\ToPrint
↓
Agent prints automatically
```

This is the safest first production workflow because it avoids accidentally buying labels or fulfilling orders incorrectly.

---

## Phase 3 — Full Shopify Automation

### Future Workflow

```text
Shopify orders/create webhook
↓
Agent receives order data
↓
Agent validates shipping address and order items
↓
Agent sends shipment request to label provider
↓
Provider returns PDF/ZPL label and tracking number
↓
Agent prints label
↓
Agent updates Shopify fulfillment with tracking
```

### Required Components

* Shopify custom app
* Webhook endpoint
* HTTPS tunnel or hosted endpoint
* Shipping label provider API
* Local print service
* Order database or log file
* Retry system
* Failed-print queue

### Shopify Events to Research

* `orders/create`
* `orders/paid`
* fulfillment orders
* fulfillment tracking info

Use `orders/paid` instead of `orders/create` if labels should only print after payment is confirmed.

---

## Recommended Shipping API Options

### Easy Start

* Pirate Ship
* ShipStation
* Shippo
* EasyPost

### More Advanced

* UPS API
* USPS API
* FedEx API

For this project, a third-party shipping API is easier than direct carrier integration.

---

## Safety Rules

Do not fully automate paid label purchasing until these checks exist:

* Confirm order is paid
* Confirm item is shippable
* Confirm address is valid
* Confirm shipping method
* Confirm printer is online
* Confirm no duplicate label was already printed
* Confirm tracking number was saved
* Confirm fulfillment update succeeded

---

## Duplicate Print Protection

The agent should eventually track printed labels by:

* Shopify order number
* Label filename
* Tracking number
* Timestamp
* Print status

Example log fields:

```text
timestamp, order_number, label_file, tracking_number, status
```

---

## Troubleshooting

### Label does not print

Check printer status:

```powershell
Get-Printer | Select-Object Name, PrinterStatus, WorkOffline
```

Check print jobs:

```powershell
Get-PrintJob -PrinterName "POLONO"
```

Clear stuck jobs:

```powershell
Get-PrintJob -PrinterName "POLONO" | Remove-PrintJob
```

Restart spooler:

```powershell
Restart-Service Spooler
```

---

### File prints but label is wrong size

Check:

* Printer driver paper size
* Label size, usually 4x6
* PDF scaling
* Default print preferences
* Shopify/shipping provider label format

Recommended label format:

```text
4x6 PDF
```

If the printer supports it later, ZPL may be better than PDF.

---

### File does not move to Printed

Check permissions:

```powershell
Test-Path C:\Labels\ToPrint
Test-Path C:\Labels\Printed
Test-Path C:\Labels\Failed
```

Run PowerShell as admin if needed.

---

## Future Enhancements

* Web dashboard
* Print history page
* Reprint button
* Shopify order lookup
* Slack/email notification on failure
* Barcode scan verification
* Auto-archive printed labels
* ZPL support
* Dockerized API service
* SQLite database
* Windows service instead of scheduled task

---

## MVP Definition

The first successful version is complete when:

* Polono printer works from Windows
* Agent starts manually
* PDF label placed in `C:\Labels\ToPrint` prints automatically
* Printed file moves to `C:\Labels\Printed`
* Errors move to `C:\Labels\Failed`
* Logs are written

---

## Next Build Step

Start with the local watcher only.

Do not connect Shopify until local printing is stable.

Sources used for the Shopify pieces: Shopify documents webhooks for event notifications, including order creation, and states that REST Admin API is legacy while new public apps should use GraphQL Admin API. ([Shopify][1]) ([Shopify][2])

[1]: https://shopify.dev/docs/apps/build/webhooks?utm_source=chatgpt.com "About webhooks"
[2]: https://shopify.dev/docs/api/admin-rest/latest/resources/webhook?utm_source=chatgpt.com "Webhook"
