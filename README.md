# OS End of Life (EOL) Notifier for n8n

A professional-grade, state-aware n8n automation workflow designed to query the [endoflife.date API](https://endoflife.date) daily and send warnings **60 days**, **30 days**, and **7 days** in advance before operating systems reach their End of Life (EOL). 

Notifications are delivered via visually rich Discord embeds, colored by urgency.

---

## 🌟 Key Features

1. **Multi-OS Tracking by Default**:
   - Monitors popular systems: **Windows Desktop**, **Windows Server**, **macOS**, **Ubuntu Linux**, **Debian Linux**, **Oracle Linux**, and **Rocky Linux**.
   - Fully customizable to track any software lifecycle supported by `endoflife.date` (e.g. Python, Node.js, PostgreSQL).

2. **State-Aware Alerts (No Spam)**:
   - Uses n8n workflow static data storage (`getWorkflowStaticData('global')`).
   - Dynamically remembers which threshold alerts have already been sent per version/cycle (e.g. `windows_11-24h2-w_60`).
   - Guarantees **exactly one notification per threshold**, even if the workflow is run manually multiple times.

3. **Dynamic Urgency Visuals (Discord Embeds)**:
   - **60-Day warning**: 🔵 Blue theme (`#3498DB`) - Information/Early Planning.
   - **30-Day warning**: 🟠 Orange theme (`#E67E22`) - Action Recommended.
   - **7-Day warning**: 🔴 Red theme (`#E74C3C`) - Immediate Action Required.
   - Includes direct links to official vendor release pages/documentation for troubleshooting and upgrades.

---

## 📁 Project Structure

*   [OS_EOL_Notifier.json](file:///c:/KacperWoźniak/Cloud_enchancement/Antigravity/n8n/EOL%20Workflow/OS_EOL_Notifier.json): The clean, credentials-free JSON file ready to be imported into n8n.
*   [End of life idea.md](file:///c:/KacperWoźniak/Cloud_enchancement/Antigravity/n8n/EOL%20Workflow/End%20of%20life%20idea.md): The original Polish concept brief.
*   [LICENSE](file:///c:/KacperWoźniak/Cloud_enchancement/Antigravity/n8n/EOL%20Workflow/LICENSE): MIT License for open sharing.
*   [.gitignore](file:///c:/KacperWoźniak/Cloud_enchancement/Antigravity/n8n/EOL%20Workflow/.gitignore): Ignores local workflow variations and private settings.

---

## 🚀 Setup & Installation Instructions

### Step 1: Import into n8n
1. Open your n8n dashboard and click **Add Workflow** (or create a new blank canvas).
2. Copy the entire contents of [OS_EOL_Notifier.json](file:///c:/KacperWoźniak/Cloud_enchancement/Antigravity/n8n/EOL%20Workflow/OS_EOL_Notifier.json).
3. Paste the contents directly into your n8n workspace canvas (or select **Import from File** in the top-right settings menu).

### Step 2: Configure the Discord Webhook
1. Open the **Send Discord Notification** HTTP Request node (the last node in the flow).
2. Replace `YOUR_DISCORD_WEBHOOK_URL_HERE` in the **URL** parameter with your target Discord channel's webhook integration URL.
3. Save the workflow.

### Step 3: Enable the Schedule Trigger
1. The **Schedule Trigger** node is configured to run daily (e.g., at midnight or 8:00 AM). You can double-click it to customize execution timing.
2. Toggle the workflow to **Active** in the top-right corner of the n8n interface.

---

## 🛠️ Customizing Monitored Products

You can easily modify the list of tracked operating systems or software products.

1. Open the **Init Products** Code node (the third node in the flow).
2. Edit the returned JSON array to add or remove items. The `id` must match the product path on [endoflife.date](https://endoflife.date) (e.g., `https://endoflife.date/node` corresponds to `node`).
   
```javascript
return [
  { json: { id: "windows", name: "Windows Desktop" } },
  { json: { id: "windows-server", name: "Windows Server" } },
  { json: { id: "macos", name: "macOS" } },
  { json: { id: "ubuntu", name: "Ubuntu Linux" } },
  { json: { id: "debian", name: "Debian Linux" } },
  { json: { id: "oracle-linux", name: "Oracle Linux" } },
  { json: { id: "rocky-linux", name: "Rocky Linux" } }
];
```

---

## ⚙️ How it Works under the Hood

### EOL Logic & State Management
Inside the **Evaluate EOL State** Code node, the following JavaScript evaluates each cycle and coordinates the tracking state:

```javascript
const items = $input.all();
const staticData = $getWorkflowStaticData('global');
staticData.sentAlerts = staticData.sentAlerts || {};

const alerts = [];
const today = new Date();

let initProducts = [];
try {
  initProducts = $('Init Products').all();
} catch (e) {
  // Safe fallback
}

for (let i = 0; i < items.length; i++) {
  const item = items[i];
  const cycleInfo = item.json;
  
  let productInfo = { id: "unknown", name: "Unknown Product" };
  try {
    // 1. Try finding by pairedItem index (most reliable when HTTP Request splits array)
    if (item.pairedItem && typeof item.pairedItem.item === 'number') {
      const idx = item.pairedItem.item;
      if (idx >= 0 && idx < initProducts.length) {
        productInfo = initProducts[idx].json;
      }
    } else {
      // 2. Fallback: Try to map 1-to-1 if lengths match, or default to first product
      if (items.length === initProducts.length && i < initProducts.length) {
        productInfo = initProducts[i].json;
      } else if (initProducts.length > 0) {
        productInfo = initProducts[0].json;
      }
    }
  } catch (e) {
    // Retain default fallback
  }
  
  const productId = productInfo.id;
  const productName = productInfo.name;
  
  const cycle = cycleInfo.cycle;
  const releaseLabel = cycleInfo.releaseLabel || cycle;
  const eol = cycleInfo.eol;
  
  if (!eol || typeof eol !== 'string') continue;
  
  const eolDate = new Date(eol);
  const timeDiff = eolDate.getTime() - today.getTime();
  const daysRemaining = Math.ceil(timeDiff / (1000 * 60 * 60 * 24));
  
  // Classify active alert threshold (only 60, 30, or 7 days)
  let activeThreshold = null;
  if (daysRemaining <= 7 && daysRemaining >= 0) {
    activeThreshold = 7;
  } else if (daysRemaining <= 30 && daysRemaining > 7) {
    activeThreshold = 30;
  } else if (daysRemaining <= 60 && daysRemaining > 30) {
    activeThreshold = 60;
  }
  
  if (activeThreshold !== null) {
    const alertKey = `${productId}_${cycle}_${activeThreshold}`;
    
    // Trigger if not sent previously for this threshold
    if (!staticData.sentAlerts[alertKey]) {
      alerts.push({
        json: {
          productId,
          productName,
          cycle,
          releaseLabel,
          eolDate: eol,
          daysRemaining,
          threshold: activeThreshold,
          alertKey,
          link: cycleInfo.link || `https://endoflife.date/${productId}`,
          // Urgency colors (Decimal format)
          color: activeThreshold === 7 ? 15158332 : (activeThreshold === 30 ? 15105570 : 3447003),
          timestamp: new Date().toISOString()
        }
      });
      staticData.sentAlerts[alertKey] = true;
    }
  }
}

return alerts;
```

### Discord Notification Embed Structure
When alerts are present, n8n sends a POST request with the following JSON embed payload to your Discord webhook channel:

```json
{
  "embeds": [
    {
      "title": "⚠️ OS End of Life Alert: {{ $json.productName }} ({{ $json.releaseLabel }})",
      "description": "The cycle **{{ $json.releaseLabel }}** of **{{ $json.productName }}** is approaching its End of Life (EOL) in **{{ $json.daysRemaining }} days**.",
      "color": {{ $json.color }},
      "fields": [
        { "name": "Product", "value": "{{ $json.productName }}", "inline": true },
        { "name": "Version Cycle", "value": "{{ $json.releaseLabel }}", "inline": true },
        { "name": "EOL Date", "value": "`{{ $json.eolDate }}`", "inline": true },
        { "name": "Days Remaining", "value": "**{{ $json.daysRemaining }} days** ({{ $json.threshold }}-day warning)", "inline": false },
        { "name": "Reference Link", "value": "[Official Lifecycle / Release Info]({{ $json.link }})", "inline": false }
      ],
      "footer": {
        "text": "n8n Automated OS EOL Notifier"
      },
      "timestamp": "{{ $json.timestamp }}"
    }
  ]
}
```
