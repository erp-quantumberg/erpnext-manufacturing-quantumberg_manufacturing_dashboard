# Quantumberg Manufacturing Dashboard

Quantumberg Manufacturing Dashboard is a Frappe app that adds a Manufacturing Dashboard page to the Frappe Desk. Its backend reads manufacturing records from the current site, and its Vue frontend presents summary metrics, charts, and a report list.

## Features

- Summary cards for open work orders, work-in-progress work orders, manufactured item value, and items in production.
- Manufacturing charts for work order status and quantity, job card status, completed operations, pending work orders, and quality inspections.
- Company and date-range filters for dashboard data.
- Links from summary cards to related Frappe DocType lists.
- A report library panel. Its current report names and record counts are sample/demo content, not generated reports.
- Graceful empty results for several charts when their underlying DocTypes are not present on the site.
- A configurable Vite/Vue frontend build that writes assets to the Frappe app's `public/dashboard` directory.

The downtime chart currently has no data source. The backend returns an explanatory message and an empty dataset until a machine downtime DocType or integration is selected and implemented.

## Requirements

### Runtime and tools

- A Frappe Bench environment on the `version-16` branch/series.
- Python 3.14 or newer, as declared in `pyproject.toml`.
- Node.js 24 and npm for building the frontend (the CI workflow uses Node.js 24).
- Git and a working Frappe site.

Frappe Bench manages the site's database and Redis services. Use the versions and configuration supported by your Frappe installation.

### App dependencies

| App                                                  | Version | Requirement            | Purpose                                                                                                                                      |
| ---------------------------------------------------- | ------- | ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| [Frappe Framework](https://github.com/frappe/frappe) | 16      | Required               | Provides the Desk page, APIs, and runtime this app is built on.                                                                              |
| [ERPNext](https://github.com/frappe/erpnext)         | 16      | Recommended (optional) | Provides the Work Order, Job Card, and Quality Inspection DocTypes used for manufacturing data, as well as the `Manufacturing Manager` role. |

ERPNext is not declared as a hard dependency in this app's `required_apps`. The dashboard can be installed without ERPNext, but endpoints for unavailable DocTypes return empty results, and the `Manufacturing Manager` role will not be available. Install ERPNext if you want the complete manufacturing dashboard experience.

No separate Python runtime packages are listed in this app's `pyproject.toml`; Frappe is provided by Bench. Frontend packages (Vue, ApexCharts, Tailwind CSS, Axios, and Lucide icons, along with Vite build tooling) are declared in `frontend/package.json` and installed with npm.

## Installation

From the directory containing your Bench, install ERPNext first if you need the manufacturing DocTypes and role used by this dashboard:

```bash
cd ~/frappe-bench
bench get-app --branch version-16 erpnext
bench --site your-site.example install-app erpnext
```

Then fetch and install the dashboard app and build its Desk assets:

```bash
bench get-app --branch version-16 https://github.com/erp-quantumberg/erpnext-manufacturing-quantumberg_manufacturing_dashboard.git
bench --site your-site.example install-app quantumberg_manufacturing_dashboard
bench build --app quantumberg_manufacturing_dashboard
```

Replace `your-site.example` with the site name shown under your Bench's `sites` directory. If ERPNext is already installed, skip its `get-app` and `install-app` commands. If you have cloned this dashboard repository into `apps/` yourself, skip its `bench get-app` command.

## Setup and development

### Build the frontend assets

The frontend is in `frontend/`. Its Vite production build outputs directly to `quantumberg_manufacturing_dashboard/public/dashboard/`, where the Desk page loads `index.js` and `index.css`.

```bash
cd ~/frappe-bench/apps/quantumberg_manufacturing_dashboard/frontend
npm install
npm run build
cd ~/frappe-bench
bench build --app quantumberg_manufacturing_dashboard
```

Commit or deploy the generated assets as appropriate for your release process. The Vite configuration reads `sites/common_site_config.json` relative to the app location, so keep the frontend within a Bench app checkout when using its configured development server.

### Run the frontend development server

With the Frappe Bench and site configured, start the Bench in one terminal:

```bash
cd ~/frappe-bench
bench start
```

In another terminal:

```bash
cd ~/frappe-bench/apps/quantumberg_manufacturing_dashboard/frontend
npm install
npm run dev
```

Vite listens on port `8080` and proxies `/api`, `/assets`, `/files`, and `/private` to the webserver port configured in `sites/common_site_config.json`. This server is for frontend development; the dashboard's Frappe API client uses `window.frappe`, which is supplied by Desk. Use the installed Desk page for an authenticated end-to-end check.

### Open the dashboard

Sign in to the Frappe Desk and open **Manufacturing Dashboard** (route: `/app/manufacturing-dashboard`). The page is configured for users with the **Manufacturing Manager** or **System Manager** role. Users also need appropriate site permissions to read the relevant companies and manufacturing records.

## Backend API

The frontend calls whitelisted methods in `quantumberg_manufacturing_dashboard.api`. Each method accepts optional `from_date`, `to_date`, `company`, and `period` arguments. The available methods are:

- `get_dashboard_summary`
- `get_work_order_analysis`
- `get_job_card_analysis`
- `get_produced_quantity`
- `get_completed_operations`
- `get_work_order_quantity`
- `get_pending_work_orders`
- `get_quality_inspection_analysis`
- `get_downtime_analysis`
- `get_manufacturing_reports`

The `period` argument supports `Last 7 Days`, `Last 30 Days`, `Last 90 Days`, and `Last Year`; the default is `Last 30 Days`. Supplying both `from_date` and `to_date` uses that explicit date range. If a queried DocType is missing, its endpoint returns an empty dataset rather than failing the whole dashboard.

## Configuration notes

- The dashboard page's allowed roles are defined in `quantumberg_manufacturing_dashboard/quantumberg_manufacturing_dashboard/page/manufacturing_dashboard/manufacturing_dashboard.json`.
- The Vite development server port is `8080`; its backend proxy target comes from the Bench site's `webserver_port` setting, defaulting to `8000`.
- Production assets are built into `quantumberg_manufacturing_dashboard/public/dashboard/`. Rebuild them after changing frontend source files.
- The manufactured item value is calculated from produced quantities and each item's current `valuation_rate`; it is not a historical valuation snapshot.
- The report panel is currently demo content. Its items display sample counts and do not open generated Frappe reports.
- Downtime analysis requires a site-specific source and is not implemented yet.
- This app does not create manufacturing records or configure ERPNext workflows; it displays information already present on the site.

## Troubleshooting

### The dashboard page is missing

- Confirm the app is installed: `bench --site your-site.example list-apps`.
- Confirm you are using a compatible Frappe version and that the signed-in user has the **Manufacturing Manager** or **System Manager** role.
- Run `bench build --app quantumberg_manufacturing_dashboard`, then reload the Desk page.

### The page reports that dashboard assets failed to load

- From `frontend/`, run `npm install` and `npm run build`.
- Run `bench build --app quantumberg_manufacturing_dashboard` from the Bench root.
- Check that `public/dashboard/index.js` and `public/dashboard/index.css` exist and that the Bench web process is running.
- Inspect the browser Console and the Frappe web process logs for the asset request error.

### Charts are empty or contain no records

- Confirm ERPNext is installed and that the site contains Work Orders, Job Cards, and/or Quality Inspections for the selected date range.
- Check that the selected company and date range match the records, and that the user can access them.
- Some endpoints intentionally return empty datasets when their DocType is unavailable. Downtime data and the report list are not live integrations in the current version.

### Vite cannot start

- Run the frontend commands from the `frontend/` directory within the Bench app checkout.
- Confirm Node.js 24 and npm are available.
- Confirm `sites/common_site_config.json` exists at the path expected by `frontend/vite.config.js`, and that its webserver port matches the running Bench.
- Check whether port `8080` is already in use; the Vite configuration uses `strictPort`.

## Contributing

Contributions are welcome. Before opening a pull request:

1. Make changes in the relevant backend or `frontend/` source files.
2. For frontend changes, run `npm install` and `npm run build` from `frontend/`.
3. Run the app tests from the Bench root:

   ```bash
   bench --site your-site.example run-tests --app quantumberg_manufacturing_dashboard
   ```

4. Install and enable the repository's pre-commit hooks:

   ```bash
   pip install pre-commit
   pre-commit install
   ```

The configured checks include Ruff (with pyupgrade rules), ESLint, and Prettier. Please include a concise description of the change and any relevant test results in your pull request.

## License

The Frappe app metadata declares the MIT License. See [license.txt](license.txt). That file currently retains template placeholders for the copyright year and holder; maintainers should fill those in before distribution.
