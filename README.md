# 🛒 RetailFlow v3

> **A zero-dependency, single-file invoicing & business management app for Indian retail shops.**  
> No backend. No npm install. No cloud subscription. Just open the HTML file and go.

---

## ✨ What even is this?

You know how most billing software is either:
- 💸 expensive SaaS with 47 features you'll never use
- 🦕 some legacy desktop app from 2009 that crashes on Windows 11
- 📡 requires an internet connection just to print an invoice

**RetailFlow is neither.** It's one `RetailFlow_v3.html` file. Open it in Chrome. Done. Your whole business lives in your browser's localStorage — no account needed, no internet needed, no monthly fee.

Built for Indian kirana stores, general merchants, freelancers, and anyone tired of paying ₹2000/month for Tally.

---

## 🚀 Getting Started (seriously, it's this easy)

```bash
# Option 1 — just double click the file
open RetailFlow_v3.html   # macOS
# or just drag it into Chrome

# Option 2 — serve it locally if you want
npx serve .
# then open http://localhost:3000/RetailFlow_v3.html
```

That's it. First launch shows a one-time setup form for your business details. Fill it in → you're live.

---

## 🗂️ Project Structure

```
RetailFlow_v3.html          ← the entire app (yes, one file)
README.md                   ← you're here
retailflow_backup_YYYY-MM-DD.json   ← auto-generated backups (your data)
```

Yep. That's the whole project. One HTML file = ~3000 lines of vanilla JS, CSS, and HTML. No React, no Vue, no Tailwind compiler, no build step, no node_modules folder eating your SSD.

---

## 🎯 Features

### 🧾 Invoice Mode
| Feature | What it does |
|---|---|
| Tax Invoice / Bill of Supply | Switches automatically based on GST registration status |
| Auto invoice numbering | Format: `I-YY/MM/001`, resets monthly within each FY |
| Per-item GST rates | Each line item can have 0%, 5%, 18%, or 40% GST independently |
| CGST + SGST split | Auto-calculated per GST Act Section 15(3) — discount applied before tax |
| HSN/SAC codes | Per-item, auto-fills from stock master |
| Partial payments | Visual progress bar, tracks paid vs balance |
| Due dates | Per invoice, highlighted in red when overdue |
| UPI QR code | Auto-generated from your UPI ID on every invoice |
| Print / PDF | Opens a clean print window — just Ctrl+P |
| Thermal receipt | 58mm/80mm thermal printer format |
| WhatsApp sharing | Pre-filled message with invoice summary + copy button |
| Customer autocomplete | Remembers names, phones, and payment preferences |
| Duplicate invoice | Clone any invoice as a starting point |
| Invoice search | Real-time search by name, invoice #, or amount |
| Filter by status | All / Cash / UPI / Credit / Partial / Paid / Unpaid |

### 💼 Business Mode
| Tab | What it does |
|---|---|
| 📊 Dashboard | P&L overview, this month's numbers, low-stock alert, recent activity |
| 📦 Purchases | Log supplier purchases, track credit/paid/cheque, input GST |
| 💰 Sales | Auto-aggregated from invoices + manual quick-entry sales |
| 💸 Expenses | Categorised (Rent, Salary, Transport, etc.), date-tracked |
| 🔔 Credit | Customers with outstanding balances, one-click payment collection |
| 💳 Payments | Full payment ledger with invoice linking |
| 📈 Analytics | Charts, trends, breakdowns — see below |

### 📈 Analytics (the good stuff)

Switch between **Monthly / Quarterly / Half-Yearly / Annual** views across 3 tabs:

- **Performance** — Grouped bar chart (Revenue vs Cost vs Profit per period), Revenue trend line chart, and a detailed period summary table with margins
- **Breakdown** — Donut charts for payment method split, expense category breakdown, and cost structure (Purchases vs Expenses)
- **Customers** — Horizontal bar rankings by revenue and order count, top customer trend line

All charts are **pure SVG** — no Chart.js, no D3, no external dependencies. Renders offline.

### 📊 GST Mode
- GSTR-1 style monthly report with CGST + SGST split
- Input GST credit tracking on purchases
- FY filter across all reports
- Export to CSV for your CA

### 📦 Stock / Inventory
- Item master with name, unit, rate, GST%, HSN code, and stock quantity
- Auto-deducts stock when invoices are saved
- Auto-fills item details (rate, GST, HSN) when adding to invoice
- Low-stock threshold alert (≤5 units) shown on dashboard

### 👥 Customers
- Auto-created from invoices, manually manageable
- Filter by outstanding / payment method
- 📋 Customer Statement — full running-balance ledger (every invoice, payment, and sale in one view)
- WhatsApp reminder with pre-filled message
- Export to CSV

### 🔒 Backup & Restore
- **⬇️ Backup** — downloads a `.json` snapshot of your entire database (header of every page)
- **⬆️ Restore** — upload any backup JSON to restore everything, with a confirmation step before overwriting

---

## 🏗️ Architecture (for the curious devs)

This is deliberately **no-framework vanilla JS**. Here's how it works:

```
State (S object)
    └── LS (localStorage wrapper)
          ├── rf_firm
          ├── rf_invoices
          ├── rf_customers
          ├── rf_purchases
          ├── rf_sales
          ├── rf_expenses
          ├── rf_payments
          ├── rf_inventory
          └── rf_inv_counters

render()  ←── called after every state change
    └── builds HTML string
    └── sets app.innerHTML
    └── attaches event listeners via onclick="window.xyz()"
```

**State management:** There's a single `S` object. Every user action mutates `S`, calls `save()` (which writes to localStorage), then calls `render()`. That's the entire framework. No virtual DOM, no diffing, no reactivity. Just string templates and innerHTML.

**Rendering pattern:**
```javascript
// Every render function returns an HTML string
function renderInvList() {
  return `<div class="card">
    ${S.invoices.map(inv => `<div>${inv.invoice_no}</div>`).join('')}
  </div>`;
}

// render() assembles them all
function render() {
  const app = document.getElementById('app');
  app.innerHTML = renderTopbar() + renderMain();
}
```

**Chart engine:** Pure SVG generated as strings. No dependencies. `svgGroupedBar()`, `svgDonut()`, `svgLine()` — all return SVG markup. Responsive via `viewBox` + `width:100%`.

**GST math:**
```javascript
// Per-item tax calculation (GST Act Section 15(3) compliant)
function calcItem(it, isGST) {
  const rate = Number(it.rate || 0);
  const qty  = Number(it.qty  || 1);
  const disc = Number(it.discount || 0);
  const gst  = Number(it.gst_rate || 0);
  
  const gross   = r2(rate * qty);
  const discAmt = r2(gross * disc / 100);
  const taxable = r2(gross - discAmt);         // discount BEFORE tax
  const gstAmt  = r2(taxable * gst / 100);
  const cgst    = r2(taxable * gst / 200);     // penny-safe split
  const sgst    = r2(gstAmt - cgst);           // remainder to sgst
  return { gross, discAmt, taxable, gstAmt, cgst, sgst, total: r2(taxable + gstAmt) };
}
```

---

## 🧑‍💻 How to Extend It

Want to add a new feature? Here's the pattern:

### 1. Add state
```javascript
// In the S object (line ~218)
const S = {
  // ... existing state
  myNewFeature: LS.get('rf_my_feature', null),  // ← add this
};

// In save() — add your key to persist it
['firm','customers','invoices', /* ... */ 'my_feature'].forEach(k => LS.set('rf_'+k, S[k]));
```

### 2. Add a render function
```javascript
function renderMyFeature() {
  return `<div class="card">
    <div class="ctitle">My Feature</div>
    ${S.myNewFeature ? `<p>${esc(S.myNewFeature)}</p>` : '<p>Nothing yet</p>'}
    <button class="btn bpri" onclick="doMyThing()">Do it</button>
  </div>`;
}
```

### 3. Wire up window actions
```javascript
// All onclick handlers must be on window (because innerHTML re-renders lose closures)
window.doMyThing = () => {
  S.myNewFeature = 'done!';
  save();
  render();
  toast('Done!', 'success');
};
```

### 4. Hook into the router
```javascript
// In renderBusiness() — add to the tab map
{dash: renderDash, myfeature: renderMyFeature, /* ... */}[S.bizTab]
```

---

## 🎨 Design System

All colours are CSS custom properties on `:root`. Override anything in the `<style>` block.

```css
:root {
  --acc:  #BF4A2B;   /* primary accent (burnt orange) */
  --grn:  #2B6E4A;   /* positive / success */
  --red:  #BF4A2B;   /* danger / negative */
  --blu:  #2655A0;   /* info / UPI */
  --ylw:  #9A6D1E;   /* warning */
  --pur:  #6B3FA0;   /* partial / special */
  --tx:   #1C1915;   /* primary text */
  --tx2:  #7A746C;   /* secondary text */
  --surf: #FFF;      /* card surface */
  --bg:   #F6F3EE;   /* page background */
}

/* dark mode just overrides the same vars */
[data-dark] {
  --bg:   #131110;
  --surf: #1D1A17;
  /* ... */
}
```

**Utility classes you'll use constantly:**
```
.g2 .g3 .g4 .g5   → CSS grid layouts
.card              → white card with shadow
.sc                → stat card (for KPI numbers)
.btn .bpri .bgho .bdng .bwa .bgrn .bblu → button variants
.bdg .bca .bu .bc .bch .bpu .bp → badge variants
.fw6 .fs11 .fs12 .tx2 .txacc .txgrn .txred → text utils
.tabs .tab .tab.on → tab bar
.fld label input   → form field
.empty .eico .etit → empty state
.alert .aerr .aok .awrn .ainfo → alert boxes
```

---

## 📦 Data Schema

All data lives in `localStorage`. Here's what each key holds:

```javascript
rf_firm: {
  firm_name, proprietor, address, phone,
  gst_registered: bool,
  gstin, state, default_gst_rate,
  payment_method, upi_id,
  bank_name, bank_acc, bank_ifsc, bank_branch,
  notes
}

rf_invoices: [{
  id, invoice_no, fy, date, due_date,
  customer_name, customer_phone,
  payment_method, status,       // 'paid' | 'unpaid' | 'partial'
  is_partial, partial_amount,
  items: [{ id, name, unit, qty, rate, discount, gst_rate, hsn, total }],
  gst_amount, total_taxable, total,
  notes
}]

rf_customers: [{
  id, name, phone, payment_method,
  total_business, outstanding, last_date
}]

rf_purchases: [{ id, supplier, item, amount, date, fy, status, gst_rate }]

rf_sales:     [{ id, customer, desc, amount, date, fy, payment_type }]

rf_expenses:  [{ id, category, description, amount, date, fy }]

rf_payments:  [{ id, cust_id, invoice_id, amount, date, notes, fy }]

rf_inventory: [{ id, name, unit, rate, gst_rate, hsn, stock }]

rf_inv_counters: { "YYMM": lastNumber }   // for invoice auto-numbering
```

---

## 🤝 Contributing

No build setup needed. Just edit the file.

```bash
# 1. Make your changes in RetailFlow_v3.html

# 2. Test in Chrome (localStorage-dependent features need a real browser)
# Open DevTools → Application → Local Storage to inspect state

# 3. Check for syntax errors before shipping
node --check RetailFlow_v3.html   # won't work directly, but:

# Extract just the JS block and check it
python3 -c "
h=open('RetailFlow_v3.html').read()
js=h[h.index('<script>\n/*')+8:h.rindex('</script>')]
open('/tmp/rf_check.js','w').write(js)
"
node --check /tmp/rf_check.js
```

**Ground rules:**
- Keep it **single-file**. No external JS files.
- No npm packages. If you need a library, load it from a CDN inside the HTML.
- All onclick handlers go on `window.*` — not inline closures.
- Always call `save(); render();` after mutating state.
- Use `r2()` for all money math. Never do raw floating point arithmetic on currency.
- Use `esc()` for any user-generated string going into HTML. Always.
- Toast for feedback — `toast('message', 'success'|'error'|'info'|'warn')`.

---

## 🚫 Known Limitations

| Limitation | Why |
|---|---|
| Data is browser-local | localStorage is per-browser, per-device. Use the Backup/Restore feature to move data. |
| ~5MB storage limit | localStorage cap. Enough for years of typical retail use. |
| No multi-device sync | By design — this is an offline-first app. |
| No invoice editing | Existing invoices are immutable. Duplicate → edit → save as new. |
| CGST+SGST only (no IGST) | B2C intra-state transactions only. Inter-state support not yet built. |
| Printing layout varies | Depends on browser print settings. Chrome recommended. |

---

## 🗺️ What Could Come Next

PRs welcome for any of these:

- [ ] **Multi-device sync** via a lightweight backend (Supabase/PocketBase)
- [ ] **Invoice editing** — mutate existing invoices with audit trail
- [ ] **IGST support** — for inter-state B2B transactions
- [ ] **Proforma invoice** — send estimates before final invoice
- [ ] **Recurring invoice** — auto-generate monthly invoices
- [ ] **Stock purchase linking** — link a purchase entry to stock increment
- [ ] **Multi-currency** — for businesses dealing in USD/AED
- [ ] **User accounts** — multiple logins, role-based access

---

## 📄 License

MIT. Do whatever you want with it. Attribution appreciated but not required.

---

<div align="center">

Built with ☕ and zero npm installs.

**[Open RetailFlow →](RetailFlow_v3.html)**

</div>
