# ☁️ Weather Tracker

A ServiceNow scoped application for tracking and looking up real-time city weather data, built with the ServiceNow Fluent SDK.

## 📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Data Model](#data-model)
- [Integration](#integration)
- [Security](#security)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Usage](#usage)
- [Validation & Troubleshooting](#validation--troubleshooting)
- [Configuration](#configuration)
- [License](#license)

## Overview

Weather Tracker is a ServiceNow scoped application (x_1850489_weathe_0) that provides two core capabilities:

**Weather Data Management** — A custom table for manually storing city weather observations with temperature, unit, and summaries.

**Live Weather Lookup** — A one-click integration with WeatherAPI.com that fetches real-time temperature and weather conditions via REST + GlideAjax.

Built entirely with the Fluent DSL (.now.ts files), the app includes tables, business rules, client scripts, REST messages, script includes, UI actions, ACLs, form/list layouts, field styles, and navigation modules.

## Features

| Feature | Description |
|---------|-------------|
| 🗂️ Weather Table | Store city weather data with country, city, temperature, unit (C/F/K), and summary |
| 🔍 Weather Lookup | Enter a city name, click "Fetch Weather", and get live temperature + condition from WeatherAPI.com |
| ⏱️ Auto Timestamps | last_updated field auto-populates on every insert/update via a before business rule |
| ✅ Client Validation | onSubmit validation ensures country/city ≥ 2 characters and temperature is numeric |
| 🔒 Role-Based Security | Custom admin role with read/write/create/delete ACLs on the weather table |
| 🌐 REST Integration | Outbound REST Message to api.weatherapi.com with parameterized GET requests |
| 🎨 Field Styling | Bold blue temperature and italic condition text via declarative Field Styles |
| 📡 GlideAjax | Async client-to-server communication using a scoped AbstractAjaxProcessor script include |

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Browser (UI)                         │
│                                                             │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────────┐  │
│  │ Weather Form │   │ Lookup Form  │   │ Client Scripts │  │
│  │ (CRUD)       │   │ + Fetch Btn  │   │ (Validation)   │  │
│  └──────────────┘   └──────┬───────┘   └────────────────┘  │
│                            │ GlideAjax                      │
├────────────────────────────┼────────────────────────────────┤
│                   Server   │                                │
│                            ▼                                │
│  ┌─────────────────────────────────────┐                    │
│  │ WeatherAjax (Script Include)        │                    │
│  │ extends global.AbstractAjaxProcessor│                    │
│  │                                     │                    │
│  │  ┌───────────────────────────────┐  │                    │
│  │  │ sn_ws.RESTMessageV2           │  │                    │
│  │  │ ('WeatherAPI','getCurrentWx') │──┼──► WeatherAPI.com  │
│  │  └───────────────────────────────┘  │     (External)     │
│  └─────────────────────────────────────┘                    │
│                                                             │
│  ┌─────────────────┐  ┌──────────┐  ┌───────────────────┐  │
│  │ Business Rules   │  │   ACLs   │  │  Field Styles     │  │
│  │ (auto timestamp) │  │ (RWCD)   │  │  (temperature/    │  │
│  └─────────────────┘  └──────────┘  │   condition)       │  │
│                                      └───────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## Project Structure

```
x_1850489_weathe_0/
├── now.config.json                          # Scope configuration
├── package.json                             # Dependencies (SDK 4.11.0)
├── tsconfig.json                            # TypeScript config
│
├── src/
│   ├── client/                              # Client-side scripts (browser)
│   │   ├── validate-weather-input.js        #   onSubmit validation for weather table
│   │   └── unit-change-handler.js           #   onChange handler for temperature unit
│   │
│   ├── scripts/                             # Server-side scripts
│   │   └── weather-ajax.js                  #   GlideAjax processor (AbstractAjaxProcessor)
│   │
│   └── fluent/                              # Fluent metadata definitions (.now.ts)
│       ├── index.now.ts                     #   App entry point
│       │
│       ├── weather-table.now.ts             #   Weather table + admin role
│       ├── weather-lookup-table.now.ts      #   Weather Lookup table
│       │
│       ├── business-rules.now.ts            #   "Set Last Updated" before rule
│       ├── client-scripts.now.ts            #   Client script definitions
│       ├── ui-action.now.ts                 #   "Refresh Data" button (weather table)
│       ├── weather-lookup-ui-action.now.ts  #   "Fetch Weather" button (lookup table)
│       │
│       ├── rest-message.now.ts              #   WeatherAPI REST Message definition
│       ├── weather-ajax-si.now.ts           #   WeatherAjax Script Include definition
│       ├── weather-ajax-acl.now.ts          #   client_callable_script_include ACL
│       │
│       ├── form-layout.now.ts               #   Weather table form layout
│       ├── list-layout.now.ts               #   Weather table list layout
│       ├── weather-lookup-form.now.ts       #   Lookup table form layout
│       ├── weather-lookup-list.now.ts       #   Lookup table list layout
│       ├── weather-lookup-styles.now.ts     #   Field Styles for read-only fields
│       │
│       ├── navigation.now.ts                #   Application menu + all modules
│       └── security.now.ts                  #   ACL rules (read/write/create/delete)
│
└── target/                                  # Build output
    └── x_1850489_weathe_0_1_0_0.zip
```

## Data Model

### Weather Table (x_1850489_weathe_0_weather)

Manual weather data storage with auto-timestamps.

| Field | Type | Required | Read-Only | Default | Description |
|-------|------|----------|-----------|---------|-------------|
| country | String (100) | ✅ | — | — | Country name |
| city | String (100) | ✅ | — | — | City name (display column) |
| temperature | Decimal | — | — | — | Temperature reading |
| unit | String (40) | — | — | C | Unit: Celsius / Fahrenheit / Kelvin |
| weather_summary | String (500) | — | — | — | Free-text weather description |
| last_updated | DateTime | — | ✅ | — | Auto-set on insert/update by business rule |

### Weather Lookup Table (x_1850489_weathe_0_lookup)

API-powered weather lookups with read-only result fields.

| Field | Type | Required | Read-Only | Description |
|-------|------|----------|-----------|-------------|
| city | String (100) | ✅ | — | City to look up |
| temperature | Decimal | — | ✅ | Temperature in °C (populated by API) |
| condition | String (200) | — | ✅ | Weather condition text (populated by API) |

## Integration

### WeatherAPI.com REST Message

| Property | Value |
|----------|-------|
| Name | WeatherAPI |
| Endpoint | https://api.weatherapi.com/v1/current.json |
| Method | GET |
| Query Params | key (API key), q (city name) |

### GlideAjax Flow

```
Browser                          Server                         External
───────                          ──────                         ────────
  │                                │                               │
  │ GlideAjax('x_1850489_        │                               │
  │   weathe_0.WeatherAjax')     │                               │
  │ sysparm_name: 'getWeather'   │                               │
  │ sysparm_city: 'London'       │                               │
  │ ──────────────────────────►  │                               │
  │                               │  RESTMessageV2                │
  │                               │  ('WeatherAPI',               │
  │                               │   'getCurrentWeather')        │
  │                               │  ────────────────────────►    │
  │                               │                               │
  │                               │  ◄──── HTTP 200 + JSON ────  │
  │                               │                               │
  │  ◄─── JSON {success,         │                               │
  │        temp_c, condition} ──  │                               │
  │                               │                               │
  │ g_form.setValue('temperature') │                               │
  │ g_form.setValue('condition')   │                               │
```

## Security

### Role

| Role | Scope | Description |
|------|-------|-------------|
| x_1850489_weathe_0.admin | Application | Full CRUD access to weather data |

### ACLs — Weather Table

| Operation | Role Required | Admin Override |
|-----------|---------------|----------------|
| Read | x_1850489_weathe_0.admin | ✅ |
| Write | x_1850489_weathe_0.admin | ✅ |
| Create | x_1850489_weathe_0.admin | ✅ |
| Delete | x_1850489_weathe_0.admin | ✅ |

### ACL — Script Include

| Type | Name | Operation | Attribute |
|------|------|-----------|-----------|
| client_callable_script_include | WeatherAjax | execute | user_is_authenticated |

⚠️ This ACL is critical. Without it, scoped-app GlideAjax calls fail silently with no server-side log entry.

## Prerequisites

- ServiceNow Instance — PDI or developer instance
- ServiceNow SDK — v4.11.0 or higher
- Node.js — Compatible version for the SDK
- WeatherAPI.com Key — Free tier at weatherapi.com

## Installation

### From Source (SDK CLI)

```bash
# 1. Clone the repository
git clone https://github.com/<your-org>/weather-tracker.git
cd weather-tracker/x_1850489_weathe_0

# 2. Install dependencies
npm install

# 3. Build the application
now-sdk build

# 4. Deploy to your instance
now-sdk install
```

### From Update Set

Navigate to System Update Sets → Retrieved Update Sets
Import the XML update set from the target/ directory
Preview and commit the update set

## Usage

### Weather Data (Manual Entry)

Open the Weather Tracker menu in the application navigator
Click Create New to open a new weather record
Fill in country, city, temperature, unit, and weather summary
Click Save — the last_updated field auto-populates
Click Refresh Data to manually update the timestamp

### Weather Lookup (Live API)

Open the Weather Tracker menu in the application navigator
Click New Weather Lookup
Enter a city name (e.g., "London", "Tokyo", "New York")
Click the Fetch Weather button
Temperature and condition fields populate automatically from WeatherAPI.com
Click Save to persist the record

## Validation & Troubleshooting

### Step-by-Step Validation

| # | Check | How |
|---|-------|-----|
| 1 | REST Message | Navigate to the WeatherAPI REST Message → Test getCurrentWeather with key=<your_key> and city=London → Confirm HTTP 200 |
| 2 | Script Include logs | After clicking Fetch Weather, check System Logs → All for WeatherAjax: API response status=200 |
| 3 | Silent failure | If no logs appear, inspect browser DevTools → Network tab → find the xmlhttp.do request → check the raw response for an empty <xml> tag (indicates ACL denial) |
| 4 | Field styling | Open a Weather Lookup record with data → temperature should be bold blue, condition should be italic gray |

### Common Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| "Fetch Weather" does nothing | Missing client_callable_script_include ACL | Verify ACL exists: type=client_callable_script_include, name=WeatherAjax, operation=execute |
| Empty response from GlideAjax | Script include name mismatch | Ensure GlideAjax uses full scoped name: x_1850489_weathe_0.WeatherAjax |
| API returns non-200 | Invalid API key or city | Check the API key in weather-ajax.js and test the city name directly at api.weatherapi.com |
| Styles not applying | Field not read-only | Confirm temperature and condition have readOnly: true in the table definition |
| sysparm_name error | Function name mismatch | sysparm_name must be getWeather (exact match to the Script Include method) |

## Configuration

### API Key

The WeatherAPI.com key is currently embedded in src/scripts/weather-ajax.js. For production use, store it in a system property:

```javascript
// Replace this:
rm.setStringParameterNoEscape('key', 'your-api-key-here');

// With this:
rm.setStringParameterNoEscape('key', gs.getProperty('x_1850489_weathe_0.weatherapi.key'));
```

Then create a system property x_1850489_weathe_0.weatherapi.key with your API key value.

### Temperature Unit

The Weather table supports three temperature units via a choice field:

| Value | Label |
|-------|-------|
| C | Celsius (default) |
| F | Fahrenheit |
| K | Kelvin |

## Built With

- ServiceNow Fluent SDK 4.11.0
- WeatherAPI.com — Free weather data API
- TypeScript — Fluent DSL metadata definitions
- JavaScript — Client and server scripts

## License

*[License information not provided in the original document.]*
