# Mini WebGIS Certification Task

Mini WebGIS developed for the MAPID Mini Certification task using Svelte and MapLibre GL JS.

The application visualizes approximately 50,000 supplier/farm locations using Mapbox Vector Tiles (MVT) and MAPID basemaps.

## Certification Requirements

The main certification tasks are:

1. Replicate the provided WebGIS.
2. Redesign the attribute popup UI.
3. Create an ON/OFF toggle for the point layer.

All main requirements have been implemented.

## Features

### Farm Location Layer

- Approximately 50,000 supplier/farm locations
- MVT-based point rendering
- ON/OFF point layer toggle
- Interactive supplier popup
- Redesigned attribute popup
- Selected feature highlighting

### Attribute Table

- Farm attribute table
- Search functionality
- Pagination
- Single feature selection
- Ctrl + Click multi-selection
- Shift + Click range selection
- Ctrl + Shift range selection
- Clear Selection
- Right-click context menu
- Zoom to selected object
- Synchronization between table selection and map

### Supplier Density Heatmap

- Independent Supplier Density layer
- ON/OFF heatmap toggle
- Green-yellow-red density visualization
- Heatmap and point layers can be displayed simultaneously
- Zoom-dependent visualization:
  - Heatmap is more dominant at smaller map scales
  - Point layer becomes more dominant when zooming in

### Map Navigation

- MAPID Street 2D
- MAPID Light
- MAPID Satellite
- MapLibre fallback basemap
- Zoom and pan controls
- N / E / S / W directional compass
- Map rotation based on geographic orientation

## Technology Stack

- Svelte
- Vite
- MapLibre GL JS
- MAPID Basemap
- Mapbox Vector Tiles (MVT)
- MBTiles
- Python tile server
- CSV attribute data

## Project Structure

```text
webgis-binus-mapid-svelte-answer/
│
├── data_mvt/
│   ├── farm-location-50k.csv
│   └── suppliers.mbtiles
│
├── src/
│   ├── lib/
│   │   ├── config.js
│   │   └── MapView.svelte
│   ├── App.svelte
│   ├── app.css
│   └── main.js
│
├── tiles/
│   └── server.py
│
├── .env
├── .gitignore
├── package.json
├── README.md
└── vite.config.js
```

## Installation

Install the project dependencies:

```bash
npm install
```

## Environment

The MAPID API key provided for this certification task is stored in:

```text
.env
```

The frontend reads the key using:

```env
VITE_MAPID_KEY=...
```

## Run the MVT Tile Server

Open a terminal from the project directory:

```bash
python tiles/server.py
```

The local MVT endpoint is:

```text
http://127.0.0.1:8080/suppliers/{z}/{x}/{y}.mvt
```

Keep this terminal running.

## Run the Web Application

Open another terminal:

```bash
npm run dev
```

The development server is normally available at:

```text
http://localhost:5173
```

## Basic Usage

### Farm Location

Enable the **Farm Location** toggle to display supplier/farm points.

The table icon beside the Farm Location layer opens the attribute table.

### Attribute Table Selection

```text
Click          = Single selection
Ctrl + Click   = Select/unselect individual records
Shift + Click  = Select a continuous range
Ctrl + Shift   = Add another range to the current selection
```

Selected records are highlighted both in the table and on the map.

Right-click a record to access:

```text
Zoom to selected object
Clear selection
```

### Supplier Density

Enable **Supplier Density** to display the heatmap.

Farm Location and Supplier Density are independent layers and may be enabled simultaneously.

## Dataset

The exercise uses approximately 50,000 farm/supplier records.

Available attributes include:

- FID
- Supplier / Entity Name
- Plot Area
- Country
- Region
- Coordinate

MVT is used to efficiently render the large point dataset.

## Certification Output

This repository contains the source code for the Mini WebGIS Certification Task.

The final visualization and demonstration output are submitted separately through Google Drive.