<script>
  import { onMount, tick } from 'svelte';
  import MapView from './lib/MapView.svelte';

  import farmCsvUrl
    from '../data_mvt/farm-location-50k.csv?url';


  // =========================================================
  // APPLICATION STATE
  // =========================================================

  let basemap = 'satellite';
  let mapView;

  // DEFAULT OFF supaya UI dan map konsisten saat pertama dibuka.
  let mvtEnabled = false;

  let heatmapEnabled = false;

  let tableOpen = false;


  // =========================================================
  // FARM DATA
  // =========================================================

  let farms = [];
  let farmByFid = new Map();

  let loading = true;
  let loadError = '';


  // =========================================================
  // SELECTION
  // =========================================================

  let selectedFid = null;
  let selectedFarm = null;

  let selectedFids = new Set();

  let selectionAnchorIndex = null;

  let selectedFarms = [];

  /* semua selected FID → object farm */
  $: selectedFarms =
    Array.from(selectedFids)
      .map((fid) =>
        farmByFid.get(String(fid))
      )
      .filter(Boolean);

  // =========================================================
  // ATTRIBUTE TABLE
  // =========================================================

  let searchTerm = '';
  let page = 1;

  const pageSize = 100;


  // =========================================================
  // RIGHT CLICK MENU
  // =========================================================

  let contextMenuOpen = false;
  let contextX = 0;
  let contextY = 0;
  let contextFarm = null;


  // =========================================================
  // BASEMAP OPTIONS
  // =========================================================

  const options = [
    {
      id: 'street',
      label: 'MAPID Street 2D'
    },
    {
      id: 'light',
      label: 'MAPID Light'
    },
    {
      id: 'satellite',
      label: 'MAPID Satellite'
    },
    {
      id: 'demo',
      label: 'MapLibre Demo (fallback)'
    }
  ];


  // =========================================================
  // CSV PARSER
  // =========================================================

  function parseCsv(text) {
    const rows = [];

    let row = [];
    let field = '';
    let inQuotes = false;

    for (let i = 0; i < text.length; i += 1) {
      const char = text[i];
      const next = text[i + 1];

      if (inQuotes) {
        if (char === '"' && next === '"') {
          field += '"';
          i += 1;
        } else if (char === '"') {
          inQuotes = false;
        } else {
          field += char;
        }

        continue;
      }

      if (char === '"') {
        inQuotes = true;
      } else if (char === ',') {
        row.push(field);
        field = '';
      } else if (char === '\n') {
        row.push(field.replace(/\r$/, ''));

        rows.push(row);

        row = [];
        field = '';
      } else {
        field += char;
      }
    }

    if (field.length || row.length) {
      row.push(field.replace(/\r$/, ''));
      rows.push(row);
    }

    if (!rows.length) {
      return [];
    }

    const headers = rows[0].map(
      (header, index) =>
        (
          index === 0
            ? header.replace(/^\uFEFF/, '')
            : header
        ).trim()
    );

    return rows
      .slice(1)
      .filter((values) =>
        values.some(
          (value) =>
            String(value).trim() !== ''
        )
      )
      .map((values) => {
        const record = {};

        headers.forEach(
          (header, index) => {
            record[header] =
              values[index] ?? '';
          }
        );

        return record;
      });
  }


  function parsePointWkt(value) {
    const text =
      String(value ?? '').trim();

    const match =
      /^POINT\s*\(\s*(-?\d+(?:\.\d+)?)\s+(-?\d+(?:\.\d+)?)\s*\)$/i.exec(
        text
      );

    if (!match) {
      return null;
    }

    const lng = Number(match[1]);
    const lat = Number(match[2]);

    if (
      !Number.isFinite(lng) ||
      !Number.isFinite(lat)
    ) {
      return null;
    }

    return {
      lng,
      lat
    };
  }


  // =========================================================
  // LOAD CSV
  // =========================================================

  onMount(async () => {
    try {
      loading = true;

      const response =
        await fetch(farmCsvUrl);

      if (!response.ok) {
        throw new Error(
          `CSV HTTP ${response.status}`
        );
      }

      const text =
        await response.text();

      farms = parseCsv(text)
        .map((row) => {
          const point =
            parsePointWkt(row.latlong);

          return {
            ...row,

            FID:
              String(row.FID ?? ''),

            plotareaha:
              Number(row.plotareaha),

            lng:
              point?.lng ?? null,

            lat:
              point?.lat ?? null
          };
        })
        .filter(
          (row) =>
            Number.isFinite(row.lng) &&
            Number.isFinite(row.lat)
        );

      farmByFid =
        new Map(
          farms.map((farm) => [
            String(farm.FID),
            farm
          ])
        );

      console.log(
        'Farm CSV loaded:',
        farms.length
      );

    } catch (error) {
      console.error(
        'CSV load error:',
        error
      );

      loadError =
        error?.message ??
        'Unable to load CSV';

    } finally {
      loading = false;
    }
  });


  // =========================================================
  // TABLE FILTERING
  // =========================================================

  $: normalizedSearch =
    searchTerm
      .trim()
      .toLowerCase();


  $: filteredRows =
    normalizedSearch
      ? farms.filter((row) =>
          [
            row.FID,
            row.entityname,
            row.plotareaha,
            row.countryname,
            row.regionlabel
          ].some((value) =>
            String(value ?? '')
              .toLowerCase()
              .includes(
                normalizedSearch
              )
          )
        )
      : farms;


  $: totalPages =
    Math.max(
      1,
      Math.ceil(
        filteredRows.length /
        pageSize
      )
    );


  $: if (page > totalPages) {
    page = totalPages;
  }


  $: startIndex =
    (page - 1) * pageSize;


  $: visibleRows =
    filteredRows.slice(
      startIndex,
      startIndex + pageSize
    );


  // =========================================================
  // LAYER TOGGLE
  // =========================================================

  async function toggleMvt() {
    mvtEnabled =
      !mvtEnabled;

    if (!mvtEnabled) {
      clearSelection();
      return;
    }

    // Tunggu Svelte kirim mvtEnabled=true
    // ke MapView terlebih dahulu.
    await tick();

    // Saat ON, langsung tampilkan
    // seluruh extent farm.
    if (farms.length > 0) {
      mapView?.fitToFarms(farms);
    }
  }

  async function toggleHeatmap() {
    heatmapEnabled = !heatmapEnabled;

    if (heatmapEnabled) {
      await tick();

      if (farms.length > 0) {
        mapView?.fitToFarms(farms);
      }
    }
  }

  // =========================================================
  // ATTRIBUTE TABLE
  // =========================================================

  function toggleAttributeTable() {
    tableOpen =
      !tableOpen;

    contextMenuOpen = false;
  }


  function handleSearch() {
    page = 1;

    contextMenuOpen = false;
  }


  // =========================================================
  // SELECTION
  // =========================================================

  function setSingleSelection(
    farm,
    absoluteIndex = null
  ) {
    if (!farm) {
      return;
    }

    const fid =
      String(farm.FID);

    selectedFids =
      new Set([fid]);

    selectedFid = fid;

    selectedFarm = farm;

    if (absoluteIndex !== null) {
      selectionAnchorIndex =
        absoluteIndex;
    }
  }

  function selectFarmFromTable(
    event,
    farm,
    absoluteIndex
  ) {
    if (!farm) {
      return;
    }

    const fid =
      String(farm.FID);

    const ctrlPressed =
      event.ctrlKey ||
      event.metaKey;

    const shiftPressed =
      event.shiftKey;


    // ==================================
    // SHIFT
    // Pilih range dari anchor
    // ==================================

    if (
      shiftPressed &&
      selectionAnchorIndex !== null
    ) {

      const start =
        Math.min(
          selectionAnchorIndex,
          absoluteIndex
        );

      const end =
        Math.max(
          selectionAnchorIndex,
          absoluteIndex
        );


      const rangeFids =
        filteredRows
          .slice(
            start,
            end + 1
          )
          .map(
            (row) =>
              String(row.FID)
          );


      // CTRL + SHIFT
      // Tambahkan range ke selection lama
      if (ctrlPressed) {

        selectedFids =
          new Set([
            ...selectedFids,
            ...rangeFids
          ]);

      } else {

        // SHIFT biasa:
        // replace selection dengan range

        selectedFids =
          new Set(rangeFids);

      }


      selectedFid = fid;
      selectedFarm = farm;

      return;
    }


    // ==================================
    // CTRL CLICK
    // Toggle row satu per satu
    // ==================================

    if (ctrlPressed) {

      const nextSelection =
        new Set(selectedFids);


      if (
        nextSelection.has(fid)
      ) {

        nextSelection.delete(fid);

      } else {

        nextSelection.add(fid);

      }


      selectedFids =
        nextSelection;


      selectionAnchorIndex =
        absoluteIndex;


      // Kalau row yang diklik masih selected,
      // jadikan active farm.
      if (
        selectedFids.has(fid)
      ) {

        selectedFid = fid;
        selectedFarm = farm;

      } else {

        // Kalau row tadi di-unselect,
        // ambil selected row terakhir sebagai active.

        const remaining =
          Array.from(
            selectedFids
          );

        const lastFid =
          remaining.length
            ? remaining[
                remaining.length - 1
              ]
            : null;


        selectedFid =
          lastFid;


        selectedFarm =
          lastFid
            ? farmByFid.get(
                lastFid
              ) ?? null
            : null;
      }

      return;
    }


    // ==================================
    // NORMAL CLICK
    // Single selection
    // ==================================

    setSingleSelection(
      farm,
      absoluteIndex
    );
  }


  function clearSelection() {
    selectedFid = null;
    selectedFarm = null;

    selectedFids =
      new Set();

    selectionAnchorIndex =
      null;

    contextFarm = null;

    contextMenuOpen = false;

    mapView?.clearSelectionView();
  }


  // =========================================================
  // MAP CLICK → TABLE SELECTION
  // =========================================================

  function handleMapSupplierSelect(event) {
    const fid =
      String(
        event.detail?.FID ?? ''
      );

    const farm =
      farmByFid.get(fid);

    if (!farm) {
      return;
    }


    const index =
      filteredRows.findIndex(
        (row) =>
          String(row.FID) === fid
      );


    setSingleSelection(
      farm,
      index >= 0
        ? index
        : null
    );


    if (
      tableOpen &&
      index >= 0
    ) {

      page =
        Math.floor(
          index / pageSize
        ) + 1;
    }
  }


  // =========================================================
  // RIGHT CLICK
  // =========================================================

  function openContextMenu(event, farm) {
    event.preventDefault();

    const fid = String(farm.FID);

    const absoluteIndex =
      filteredRows.findIndex(
        (row) =>
          String(row.FID) === fid
      );


    // Kalau row yang di-right-click belum termasuk selection,
    // jadikan dia single selection.
    if (!selectedFids.has(fid)) {
      setSingleSelection(
        farm,
        absoluteIndex >= 0
          ? absoluteIndex
          : null
      );
    } else {
      // Kalau sudah selected,
      // jangan buang multi-selection lainnya.
      selectedFid = fid;
      selectedFarm = farm;
    }


    contextFarm = farm;


    const menuWidth = 230;
    const menuHeight = 90;


    contextX = Math.min(
      event.clientX,
      window.innerWidth -
        menuWidth -
        10
    );


    contextY = Math.min(
      event.clientY,
      window.innerHeight -
        menuHeight -
        10
    );


    contextMenuOpen = true;
  }


  function closeContextMenu() {
    contextMenuOpen = false;
  }


  function zoomContextFarm() {
    if (!contextFarm) {
      return;
    }


    selectedFid =
      String(
        contextFarm.FID
      );

    selectedFarm =
      contextFarm;


    mapView?.zoomToFarm(
      contextFarm
    );


    contextMenuOpen = false;
  }
</script>


<svelte:window
  on:click={closeContextMenu}
/>


<div class="app">

  <!-- ===================================================== -->
  <!-- SIDEBAR -->
  <!-- ===================================================== -->

  <aside>

    <div>

      <p class="kicker">
        Mini WebGIS Certification Task
      </p>

      <h1>
        Supplier WebGIS
      </h1>


      <!-- LAYER CARD -->

      <!-- FARM LOCATION LAYER -->

      <div class="layer-control">

        <div class="layer-info">

          <span class="layer-title">
            Farm Location
          </span>

          <span class="layer-type">
            MVT
          </span>

        </div>


        <div class="layer-actions">

          <!-- ATTRIBUTE TABLE ICON -->

          <button
            type="button"
            class="table-icon-button"
            class:active={tableOpen}
            title={
              tableOpen
                ? 'Close Attribute Table'
                : 'Open Attribute Table'
            }
            aria-label="Open Attribute Table"
            on:click={toggleAttributeTable}
          >

            <svg
              viewBox="0 0 24 24"
              width="17"
              height="17"
              aria-hidden="true"
            >

              <rect
                x="3"
                y="4"
                width="18"
                height="16"
                rx="2"
                fill="none"
                stroke="currentColor"
                stroke-width="1.8"
              />

              <path
                d="M3 9h18M9 9v11M15 9v11"
                fill="none"
                stroke="currentColor"
                stroke-width="1.8"
              />

            </svg>

          </button>


          <!-- FARM POINT TOGGLE -->

          <button
            type="button"
            class="mvt-switch"
            class:active={mvtEnabled}
            role="switch"
            aria-checked={mvtEnabled}
            aria-label="Toggle Farm MVT Layer"
            on:click={toggleMvt}
          >

            <span class="switch-knob"></span>

          </button>

        </div>

      </div>


      <!-- HEATMAP LAYER -->

      <div class="layer-control heatmap-control">

        <div class="layer-info">

          <span class="layer-title">
            Supplier Density
          </span>

          <span class="layer-type">
            HEATMAP
          </span>

        </div>


        <div class="layer-actions">

          <button
            type="button"
            class="mvt-switch"
            class:active={heatmapEnabled}
            role="switch"
            aria-checked={heatmapEnabled}
            aria-label="Toggle Supplier Density Heatmap"
            on:click={toggleHeatmap}
          >

            <span class="switch-knob"></span>

          </button>

        </div>

      </div>


      <p class="status">
        Farm layer is
        {mvtEnabled ? 'ON' : 'OFF'}.
      </p>


      <p class="note">
        Click a farm point to view
        its supplier attributes.
      </p>


      {#if loading}

        <p class="data-status">
          Loading farm attributes...
        </p>

      {:else if loadError}

        <p class="data-status error">
          CSV error: {loadError}
        </p>

      {:else}

        <p class="data-status">
          {farms.length.toLocaleString()}
          farm records loaded
        </p>

      {/if}

    </div>


    <!-- BASEMAP AT BOTTOM -->

    <div class="basemap-section">

      <label for="basemap">
        Basemap
      </label>

      <select
        id="basemap"
        bind:value={basemap}
      >

        {#each options as option}

          <option value={option.id}>
            {option.label}
          </option>

        {/each}

      </select>

    </div>

  </aside>


  <!-- ===================================================== -->
  <!-- MAP + TABLE -->
  <!-- ===================================================== -->

  <main class="workspace">

    <div class="map-shell">

      <MapView
        bind:this={mapView}
        {basemap}
        {mvtEnabled}
        {heatmapEnabled}
        {selectedFarm}
        {selectedFarms}
        on:supplierselect={
          handleMapSupplierSelect
        }
      />

    </div>


    <!-- =================================================== -->
    <!-- ATTRIBUTE TABLE -->
    <!-- =================================================== -->

    {#if tableOpen}

      {#if selectedFid}

        <div class="selection-action-bar">

          <button
            type="button"
            class="clear-selection-top"
            on:click={clearSelection}
          >
            Clear Selection
          </button>

        </div>

      {/if}

      <section class="attribute-panel">

        <header class="table-toolbar">

          <div>

            <strong>
              Farm Attribute Table
            </strong>

            <span class="record-count">
              {filteredRows.length.toLocaleString()}
              records
            </span>

          </div>


          <div class="table-toolbar-actions">

            <input
              type="search"
              placeholder="Search FID, supplier, country or region..."
              bind:value={searchTerm}
              on:input={handleSearch}
            />

          </div>

        </header>


        <div class="table-scroll">

          <table>

            <thead>

              <tr>
                <th>FID</th>
                <th>Supplier</th>
                <th>Plot Area (ha)</th>
                <th>Country</th>
                <th>Region</th>
                <th>Coordinate</th>
              </tr>

            </thead>


            <tbody>

              {#each visibleRows as row, rowIndex (row.FID)}

                <tr
                  class:selected={
                    selectedFids.has(
                      String(row.FID)
                    )
                  }

                  class:active-row={
                    String(row.FID) ===
                    String(selectedFid)
                  }

                  on:mousedown={(event) => {
                    if (
                      event.shiftKey ||
                      event.ctrlKey ||
                      event.metaKey
                    ) {
                      event.preventDefault();
                    }
                  }}

                  on:click={(event) =>
                    selectFarmFromTable(
                      event,
                      row,
                      startIndex + rowIndex
                    )
                  }

                  on:contextmenu={(event) =>
                    openContextMenu(
                      event,
                      row
                    )
                  }
                >

                  <td>{row.FID}</td>

                  <td>
                    {row.entityname}
                  </td>

                  <td>
                    {
                      Number.isFinite(
                        row.plotareaha
                      )
                        ? row.plotareaha
                        : '—'
                    }
                  </td>

                  <td>
                    {row.countryname}
                  </td>

                  <td>
                    {row.regionlabel}
                  </td>

                  <td>
                    {row.latlong}
                  </td>

                </tr>

              {/each}

            </tbody>

          </table>

        </div>


        <footer class="table-footer">

          <span>

            Showing

            {filteredRows.length
              ? startIndex + 1
              : 0}

            –

            {Math.min(
              startIndex + pageSize,
              filteredRows.length
            )}

            of

            {filteredRows.length.toLocaleString()}

          </span>


          <div class="pagination">

            <button
              type="button"
              disabled={page === 1}
              on:click={() =>
                page -= 1
              }
            >
              Previous
            </button>


            <span>
              Page {page}
              of {totalPages}
            </span>


            <button
              type="button"
              disabled={
                page === totalPages
              }
              on:click={() =>
                page += 1
              }
            >
              Next
            </button>

          </div>

        </footer>

      </section>

    {/if}

  </main>


  <!-- ===================================================== -->
  <!-- CONTEXT MENU -->
  <!-- ===================================================== -->

  {#if contextMenuOpen}

    <div
      class="context-menu"
      style="
        left: {contextX}px;
        top: {contextY}px;
      "
      on:click={(event) =>
        event.stopPropagation()
      }
    >

      <button
        type="button"
        on:click={zoomContextFarm}
      >
        Zoom to selected object
      </button>


      <button
        type="button"
        on:click={clearSelection}
      >
        Clear selection
      </button>

    </div>

  {/if}

</div>


<style>

  /* ======================================================= */
  /* GLOBAL */
  /* ======================================================= */

  :global(body) {
    margin: 0;
    overflow: hidden;

    font-family:
      Inter,
      system-ui,
      -apple-system,
      BlinkMacSystemFont,
      "Segoe UI",
      sans-serif;
  }


  .app {
    display: flex;

    width: 100vw;
    height: 100vh;

    overflow: hidden;

    background: #f4f1ea;
  }


  /* ======================================================= */
  /* SIDEBAR */
  /* ======================================================= */

  aside {
    width: 280px;
    flex-shrink: 0;

    padding: 20px 18px;

    display: flex;
    flex-direction: column;

    justify-content: space-between;

    box-sizing: border-box;

    background: #f4f1ea;

    border-right:
      1px solid #ddd6c8;

    z-index: 10;
  }


  .kicker {
    margin: 0 0 8px;

    font-size: 11px;

    letter-spacing: 0.06em;

    text-transform: uppercase;

    color: #6b6458;
  }


  h1 {
    margin: 0 0 24px;

    font-size: 22px;

    line-height: 1.2;

    font-weight: 700;

    color: #171717;
  }


  /* ======================================================= */
  /* LAYER CARD */
  /* ======================================================= */

  .layer-control {
    display: flex;

    align-items: center;

    justify-content:
      space-between;

    gap: 12px;

    padding: 10px 12px;

    background: white;

    border:
      1px solid #d5d0c6;

    border-radius: 8px;
  }


  .layer-info {
    display: flex;

    flex-direction: column;
  }


  .layer-title {
    font-size: 13px;

    font-weight: 650;

    color: #222;
  }


  .layer-type {
    margin-top: 2px;

    font-size: 10px;

    color: #777;
  }


  .layer-actions {
    display: flex;

    align-items: center;

    gap: 8px;

    flex-shrink: 0;
  }


  /* ======================================================= */
  /* ATTRIBUTE TABLE ICON */
  /* ======================================================= */

  .table-icon-button {
    width: 30px;
    height: 30px;

    padding: 0;
    margin: 0;

    display: flex;

    align-items: center;
    justify-content: center;

    border:
      1px solid #d1d5db;

    border-radius: 6px;

    background: white;

    color: #4b5563;

    cursor: pointer;

    transition:
      background 0.15s ease,
      color 0.15s ease,
      border-color 0.15s ease;
  }


  .table-icon-button:hover {
    background: #f3f4f6;

    color: #111827;
  }


  .table-icon-button.active {
    background: #edf7f2;

    border-color: #0f6f4a;

    color: #0f6f4a;
  }


  /* ======================================================= */
  /* MVT TOGGLE */
  /* ======================================================= */

  .mvt-switch {
    position: relative;

    width: 48px;
    height: 26px;

    padding: 0;
    margin: 0;

    border: none;

    border-radius: 999px;

    background: #d1d5db;

    cursor: pointer;

    flex-shrink: 0;

    transition:
      background 0.2s ease;
  }


  .mvt-switch.active {
    background: #22c55e;
  }


  .switch-knob {
    position: absolute;

    width: 20px;
    height: 20px;

    left: 3px;
    top: 3px;

    border-radius: 50%;

    background: white;

    box-shadow:
      0 1px 4px
      rgba(0, 0, 0, 0.3);

    transition:
      transform 0.2s ease;
  }


  .mvt-switch.active
  .switch-knob {
    transform:
      translateX(22px);
  }


  /* ======================================================= */
  /* SIDEBAR TEXT */
  /* ======================================================= */

  .status {
    margin:
      14px 0 0;

    font-size: 13px;

    color: #34312d;
  }


  .note {
    margin:
      7px 0 0;

    font-size: 12px;

    line-height: 1.45;

    color: #6b6458;
  }


  .data-status {
    margin:
      9px 0 0;

    font-size: 11px;

    color: #777;
  }


  .data-status.error {
    color: #b42318;
  }


  /* ======================================================= */
  /* BASEMAP */
  /* ======================================================= */

  .basemap-section {
    margin-top: auto;
  }


  label {
    display: block;

    margin-bottom: 6px;

    font-size: 12px;

    font-weight: 650;
  }


  select {
    width: 100%;

    padding: 8px 10px;

    box-sizing: border-box;

    border:
      1px solid #c9c1b2;

    border-radius: 6px;

    background: white;

    font: inherit;

    font-size: 13px;
  }


  /* ======================================================= */
  /* MAP WORKSPACE */
  /* ======================================================= */

  .workspace {
    flex: 1;

    min-width: 0;
    min-height: 0;

    display: flex;

    flex-direction: column;
  }


  .map-shell {
    flex: 1;

    min-height: 0;

    display: flex;
  }


  /* ======================================================= */
  /* ATTRIBUTE TABLE */
  /* ======================================================= */

  .attribute-panel {
    height: 310px;

    flex-shrink: 0;

    display: flex;

    flex-direction: column;

    background: white;

    border-top:
      1px solid #d9dde2;

    box-shadow:
      0 -4px 14px
      rgba(0, 0, 0, 0.08);
  }


  .table-toolbar {
    display: flex;

    align-items: center;

    justify-content:
      space-between;

    gap: 12px;

    padding: 9px 12px;

    border-bottom:
      1px solid #e2e5e8;

    background: #f7f8fa;
  }


  .record-count {
    margin-left: 10px;

    font-size: 11px;

    color: #6b7280;
  }


  .table-toolbar-actions {
    display: flex;

    align-items: center;

    gap: 8px;
  }



  .table-toolbar input {
    width: 330px;

    padding: 7px 9px;

    border:
      1px solid #cfd4da;

    border-radius: 6px;

    font: inherit;

    font-size: 12px;
  }


  .table-scroll {
    flex: 1;

    min-height: 0;

    overflow: auto;
  }


  table {
    width: 100%;

    border-collapse:
      collapse;

    white-space: nowrap;

    font-size: 12px;
  }


  th,
  td {
    padding: 7px 10px;

    text-align: left;

    border-right:
      1px solid #eceef0;

    border-bottom:
      1px solid #eceef0;
  }


  th {
    position: sticky;

    top: 0;

    z-index: 3;

    background: #eef1f4;

    font-weight: 650;
  }


  tbody tr {
    cursor: pointer;
  }


  tbody tr:hover {
    background: #f2f7fb;
  }


  tbody tr.selected {
    background: #dceeff;
  }

  tbody tr.active-row {
    box-shadow:
      inset 4px 0 0
      #1677c8;
  }


  /* ======================================================= */
  /* TABLE FOOTER */
  /* ======================================================= */

  .table-footer {
    display: flex;

    align-items: center;

    justify-content:
      space-between;

    gap: 12px;

    padding: 7px 12px;

    border-top:
      1px solid #e2e5e8;

    background: #f7f8fa;

    font-size: 11px;
  }


  .pagination {
    display: flex;

    align-items: center;

    gap: 8px;
  }


  .pagination button {
    padding: 5px 9px;

    border:
      1px solid #cfd4da;

    border-radius: 5px;

    background: white;

    cursor: pointer;
  }


  .pagination button:disabled {
    opacity: 0.4;

    cursor: default;
  }


  /* ======================================================= */
  /* RIGHT CLICK MENU */
  /* ======================================================= */

  .context-menu {
    position: fixed;

    z-index: 9999;

    min-width: 215px;

    padding: 5px;

    background: white;

    border:
      1px solid #d1d5db;

    border-radius: 7px;

    box-shadow:
      0 8px 24px
      rgba(0, 0, 0, 0.18);
  }


  .context-menu button {
    width: 100%;

    padding: 8px 10px;

    border: none;

    border-radius: 5px;

    background: white;

    text-align: left;

    cursor: pointer;

    font: inherit;

    font-size: 12px;
  }


  .context-menu button:hover {
    background: #eef4f8;
  }

.selection-action-bar {
  height: 38px;

  flex-shrink: 0;

  display: flex;
  align-items: center;

  padding: 0 10px;

  background: #f7f8fa;

  border-top: 1px solid #d9dde2;
  border-bottom: 1px solid #e2e5e8;
}


.clear-selection-top {
  width: auto;

  padding: 5px 10px;

  margin: 0;

  border: 1px solid #cfd4da;
  border-radius: 4px;

  background: #ffffff;

  color: #222;

  font: inherit;
  font-size: 12px;
  font-weight: 500;

  cursor: pointer;
}


.clear-selection-top:hover {
  background: #f1f3f5;

  border-color: #aeb4ba;
}

.table-scroll tbody,
.table-scroll tbody td {
  user-select: none;
  -webkit-user-select: none;
}

.heatmap-control {
  margin-top: 8px;
}
</style>