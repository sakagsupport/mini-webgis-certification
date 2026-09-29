<script>
  import {
    onMount,
    onDestroy,
    createEventDispatcher
  } from 'svelte';

  import maplibregl
    from 'maplibre-gl';

  import {
    basemaps,
    MVT_TILES,
    MVT_SOURCE_LAYER,
    MAP_CENTER,
    MAP_ZOOM
  } from './config.js';


  export let basemap = 'street';

  export let mvtEnabled = false;

  export let heatmapEnabled = false;

  export let selectedFarm = null;

  export let selectedFarms = [];


  const dispatch =
    createEventDispatcher();


  let container;
  let map;
  let popup;

  let currentBearing = 0;

  let appliedBasemap =
    basemap;


  const SOURCE_ID =
    'suppliers';

  const LAYER_ID =
    'suppliers-circles';

  const HEATMAP_LAYER_ID =
  'suppliers-heatmap';

  const SELECTED_SOURCE_ID =
    'selected-farm-source';

  const SELECTED_HALO_LAYER_ID =
    'selected-farm-halo';

  const SELECTED_POINT_LAYER_ID =
    'selected-farm-point';


  const circlePaint = {

    'circle-radius': [
      'interpolate',
      ['linear'],
      ['zoom'],

      6, 3,
      12, 6,
      16, 9
    ],

    'circle-color':
      '#0f6f4a',

    'circle-stroke-width':
      1,

    'circle-stroke-color':
      '#ffffff'
  };


  function escapeHtml(value) {
    return String(
      value ?? '—'
    )
      .replaceAll('&', '&amp;')
      .replaceAll('<', '&lt;')
      .replaceAll('>', '&gt;')
      .replaceAll('"', '&quot;');
  }


  function showFarmPopup(
    lng,
    lat,
    farm
  ) {
    if (
      !map ||
      !popup ||
      !farm
    ) {
      return;
    }


    const supplier =
      farm.entityname ??
      'Unknown Supplier';


    const fid =
      farm.FID ??
      '—';


    const plotArea =
      farm.plotareaha !== undefined &&
      farm.plotareaha !== null &&
      farm.plotareaha !== ''
        ? `${farm.plotareaha} ha`
        : '—';


    const country =
      farm.countryname ??
      '—';


    const region =
      farm.regionlabel ??
      '—';


    const coordinateText =
      Number.isFinite(Number(lat)) &&
      Number.isFinite(Number(lng))
        ? `${Number(lat).toFixed(5)}, ${Number(lng).toFixed(5)}`
        : '—';


    popup
      .setLngLat([
        lng,
        lat
      ])
      .setHTML(
        `
          <div class="farm-popup-card">

            <div class="farm-popup-header">

              <div>

                <div class="farm-popup-eyebrow">
                  FARM DETAILS
                </div>

                <div class="farm-popup-title">
                  ${escapeHtml(supplier)}
                </div>

              </div>

              <span class="farm-popup-fid">
                FID ${escapeHtml(fid)}
              </span>

            </div>


            <div class="farm-popup-divider"></div>


            <div class="farm-popup-grid">

              <div class="farm-popup-label">
                Plot Area
              </div>

              <div class="farm-popup-value">
                ${escapeHtml(plotArea)}
              </div>


              <div class="farm-popup-label">
                Country
              </div>

              <div class="farm-popup-value">
                ${escapeHtml(country)}
              </div>


              <div class="farm-popup-label">
                Region
              </div>

              <div class="farm-popup-value">
                ${escapeHtml(region)}
              </div>

            </div>


            <div class="farm-popup-coordinate">

              <span class="coordinate-dot"></span>

              ${escapeHtml(coordinateText)}

            </div>

          </div>
        `
      )
      .addTo(map);
  }


  function onSupplierClick(
    event
  ) {
    const feature =
      event.features?.[0];


    const props =
      feature?.properties;


    if (!props) {
      return;
    }


    showFarmPopup(
      event.lngLat.lng,
      event.lngLat.lat,
      props
    );


    dispatch(
      'supplierselect',
      {
        ...props,

        FID:
          String(
            props.FID ?? ''
          )
      }
    );
  }


  function onPointer() {
    if (!map) {
      return;
    }

    map
      .getCanvas()
      .style
      .cursor =
        'pointer';
  }


  function onPointerOut() {
    if (!map) {
      return;
    }

    map
      .getCanvas()
      .style
      .cursor =
        '';
  }


  function clearSupplierLayers() {
    if (!map?.getStyle()) {
      return;
    }


    // POINT LAYER

    if (
      map.getLayer(
        LAYER_ID
      )
    ) {

      map.off(
        'click',
        LAYER_ID,
        onSupplierClick
      );

      map.off(
        'mouseenter',
        LAYER_ID,
        onPointer
      );

      map.off(
        'mouseleave',
        LAYER_ID,
        onPointerOut
      );

      map.removeLayer(
        LAYER_ID
      );
    }


    // HEATMAP LAYER

    if (
      map.getLayer(
        HEATMAP_LAYER_ID
      )
    ) {

      map.removeLayer(
        HEATMAP_LAYER_ID
      );
    }


    // SHARED MVT SOURCE

    if (
      map.getSource(
        SOURCE_ID
      )
    ) {

      map.removeSource(
        SOURCE_ID
      );
    }
  }

function ensureSupplierSource() {
  if (
    !map?.getStyle() ||
    map.getSource(SOURCE_ID)
  ) {
    return;
  }


  map.addSource(
    SOURCE_ID,
    {
      type: 'vector',

      tiles: [
        MVT_TILES
      ],

      minzoom: 0,

      maxzoom: 14
    }
  );
}

  function addSupplierPointLayer() {
    if (
      !map?.getStyle() ||
      !mvtEnabled
    ) {
      return;
    }


    /*
    * Kalau heatmap juga ON:
    * zoom jauh → points dibuat ringan
    * zoom dekat → points semakin dominan.
    */

    const pointOpacity =
      heatmapEnabled
        ? [
            'interpolate',
            ['linear'],
            ['zoom'],

            4, 0.08,

            7, 0.18,

            9, 0.45,

            11, 0.80,

            13, 1
          ]
        : 1;


    map.addLayer({
      id:
        LAYER_ID,

      type:
        'circle',

      source:
        SOURCE_ID,

      'source-layer':
        MVT_SOURCE_LAYER,

      paint: {

        ...circlePaint,

        'circle-opacity':
          pointOpacity,

        'circle-stroke-opacity':
          pointOpacity
      }
    });


    map.on(
      'click',
      LAYER_ID,
      onSupplierClick
    );

    map.on(
      'mouseenter',
      LAYER_ID,
      onPointer
    );

    map.on(
      'mouseleave',
      LAYER_ID,
      onPointerOut
    );
  }


  function addSupplierHeatmapLayer() {
    if (
      !map?.getStyle() ||
      !heatmapEnabled
    ) {
      return;
    }


    /*
    * Kalau point layer juga ON:
    * heatmap fade-out saat zoom mendekat.
    *
    * Kalau point OFF:
    * heatmap tetap terlihat kuat.
    */

    const heatmapOpacity =
      mvtEnabled
        ? [
            'interpolate',
            ['linear'],
            ['zoom'],

            4, 0.92,

            7, 0.88,

            9, 0.72,

            11, 0.42,

            13, 0.16,

            15, 0.05
          ]
        : 0.90;


    map.addLayer({
      id:
        HEATMAP_LAYER_ID,

      type:
        'heatmap',

      source:
        SOURCE_ID,

      'source-layer':
        MVT_SOURCE_LAYER,

      paint: {

        'heatmap-weight':
          1,


        'heatmap-intensity': [
          'interpolate',
          ['linear'],
          ['zoom'],

          4, 0.7,

          7, 1.0,

          10, 1.35,

          13, 1.65
        ],


        'heatmap-radius': [
          'interpolate',
          ['linear'],
          ['zoom'],

          4, 12,

          7, 20,

          10, 28,

          13, 34
        ],


        'heatmap-color': [
          'interpolate',
          ['linear'],
          ['heatmap-density'],

          0,
          'rgba(0, 0, 0, 0)',

          0.15,
          'rgba(46, 204, 113, 0.25)',

          0.35,
          '#2ecc71',

          0.55,
          '#ffe066',

          0.75,
          '#ff9f1c',

          1,
          '#e63946'
        ],


        'heatmap-opacity':
          heatmapOpacity
      }
    });
  }

  function clearSelectedFarm() {
    if (!map?.getStyle()) {
      return;
    }


    if (
      map.getLayer(
        SELECTED_POINT_LAYER_ID
      )
    ) {
      map.removeLayer(
        SELECTED_POINT_LAYER_ID
      );
    }


    if (
      map.getLayer(
        SELECTED_HALO_LAYER_ID
      )
    ) {
      map.removeLayer(
        SELECTED_HALO_LAYER_ID
      );
    }


    if (
      map.getSource(
        SELECTED_SOURCE_ID
      )
    ) {
      map.removeSource(
        SELECTED_SOURCE_ID
      );
    }
  }


  function addSelectedFarm() {
    if (
      !map?.getStyle() ||
      !Array.isArray(selectedFarms) ||
      selectedFarms.length === 0
    ) {
      return;
    }


    const features =
      selectedFarms
        .map((farm) => {
          const lng =
            Number(farm.lng);

          const lat =
            Number(farm.lat);

          if (
            !Number.isFinite(lng) ||
            !Number.isFinite(lat)
          ) {
            return null;
          }

          return {
            type: 'Feature',

            geometry: {
              type: 'Point',

              coordinates: [
                lng,
                lat
              ]
            },

            properties: {
              FID:
                String(farm.FID)
            }
          };
        })
        .filter(Boolean);


    if (!features.length) {
      return;
    }


    map.addSource(
      SELECTED_SOURCE_ID,
      {
        type: 'geojson',

        data: {
          type:
            'FeatureCollection',

          features
        }
      }
    );


    // Halo semua selected point

    map.addLayer({
      id:
        SELECTED_HALO_LAYER_ID,

      type:
        'circle',

      source:
        SELECTED_SOURCE_ID,

      paint: {
        'circle-radius':
          10,

        'circle-color':
          '#ffd43b',

        'circle-opacity':
          0.30
      }
    });


    // Semua selected point jadi kuning

    map.addLayer({
      id:
        SELECTED_POINT_LAYER_ID,

      type:
        'circle',

      source:
        SELECTED_SOURCE_ID,

      paint: {
        'circle-radius': [
          'interpolate',
          ['linear'],
          ['zoom'],

          6, 4,
          12, 7,
          16, 10
        ],

        'circle-color':
          '#ffb703',

        'circle-stroke-width':
          2,

        'circle-stroke-color':
          '#3a3000'
      }
    });
  }


  function applyDynamicLayers() {
    if (
      !map ||
      !map.isStyleLoaded()
    ) {
      return;
    }


    // Bersihkan layer visual lama
    clearSupplierLayers();


    // Source dibutuhkan kalau
    // salah satu layer ON
    if (
      mvtEnabled ||
      heatmapEnabled
    ) {

      ensureSupplierSource();
    }


    // Heatmap selalu dibuat dulu
    // supaya berada di bawah points.
    if (heatmapEnabled) {

      addSupplierHeatmapLayer();
    }


    // Point dibuat terakhir
    // sehingga berada di atas heatmap.
    if (mvtEnabled) {

      addSupplierPointLayer();
    }


    // Selected objects selalu
    // berada paling atas.
    clearSelectedFarm();


    if (
      (mvtEnabled ||
        heatmapEnabled) &&
      selectedFarms.length > 0
    ) {

      addSelectedFarm();
    }
  }


  // =========================================================
  // PUBLIC FUNCTIONS
  // =========================================================

  export function fitToFarms(farms) {
    if (
      !map ||
      !Array.isArray(farms) ||
      farms.length === 0
    ) {
      return;
    }

    const bounds =
      new maplibregl.LngLatBounds();

    let validCount = 0;

    for (const farm of farms) {
      const lng = Number(farm.lng);
      const lat = Number(farm.lat);

      if (
        !Number.isFinite(lng) ||
        !Number.isFinite(lat)
      ) {
        continue;
      }

      /*
      * Valid extent untuk dataset Côte d'Ivoire
      * sekaligus membuang coordinate outlier.
      */
      const insideStudyArea =
        lng >= -9 &&
        lng <= -2 &&
        lat >= 4 &&
        lat <= 11;

      if (!insideStudyArea) {
        continue;
      }

      bounds.extend([
        lng,
        lat
      ]);

      validCount += 1;
    }

    if (validCount === 0) {
      return;
    }

    map.fitBounds(bounds, {
      padding: {
        top: 35,
        bottom: 35,
        left: 35,
        right: 35
      },

      duration: 1000,

      maxZoom: 9
    });
  }


  export function clearSelectionView() {
    popup?.remove();
  }


  export function zoomToFarm(
    farm
  ) {
    if (
      !map ||
      !farm
    ) {
      return;
    }


    const lng =
      Number(farm.lng);

    const lat =
      Number(farm.lat);


    if (
      !Number.isFinite(lng) ||
      !Number.isFinite(lat)
    ) {
      return;
    }


    // BUKA POPUP TERLEBIH DAHULU

    showFarmPopup(
      lng,
      lat,
      farm
    );


    // LALU CAMERA FLY

    map.flyTo({
      center: [
        lng,
        lat
      ],

      zoom: 15,

      speed: 1.2,

      curve: 1.4,

      essential: true
    });
  }


  // =========================================================
  // INITIAL MAP
  // =========================================================

  function faceDirection(direction) {
    if (!map) {
      return;
    }


    const bearings = {
      N: 0,
      E: 90,
      S: 180,
      W: -90
    };


    const bearing =
      bearings[direction];


    map.rotateTo(
      bearing,
      {
        duration: 600
      }
    );
  }


  function syncBearing() {
    if (!map) {
      return;
    }

    currentBearing =
      map.getBearing();
  }

  onMount(() => {

    popup =
      new maplibregl.Popup({
        closeButton:
          true,

        closeOnClick:
          true
      });


    map =
      new maplibregl.Map({

        container,

        style:
          basemaps[
            basemap
          ],

        center:
          MAP_CENTER,

        zoom:
          MAP_ZOOM
      });


    map.addControl(
      new maplibregl.NavigationControl({
        showZoom: true,
        showCompass: false
      }),

      'top-right'
    );

  });


  // =========================================================
  // BASEMAP CHANGE
  // =========================================================

  $: if (
    map &&
    basemap !== appliedBasemap
  ) {

    appliedBasemap =
      basemap;

    popup?.remove();


    map.setStyle(
      basemaps[
        basemap
      ],
      {
        diff: false
      }
    );
  }


  // =========================================================
  // MVT / SELECTION REACTIVITY
  // =========================================================

  $: if (map) {

    mvtEnabled;
    heatmapEnabled;

    selectedFarm;
    selectedFarms;

    if (
      map.isStyleLoaded()
    ) {
      applyDynamicLayers();
    }
  }


  onDestroy(() => {

    popup?.remove();

    map?.remove();

  });

</script>


<div class="map-wrapper">

  <div
    class="map"
    bind:this={container}
  ></div>


  <div
    class="direction-pad"
    aria-label="Map orientation control"
  >

    <button
      type="button"
      class="direction-button north"
      title="Face North"
      on:click={() =>
        faceDirection('N')
      }
    >
      N
    </button>


    <button
      type="button"
      class="direction-button west"
      title="Face West"
      on:click={() =>
        faceDirection('W')
      }
    >
      W
    </button>


    <div
      class="compass-center"
      title="True North indicator"
    >

      <span
        class="compass-needle"
        style="
          transform:
          rotate({-currentBearing}deg);
        "
      >
        ▲
      </span>

    </div>


    <button
      type="button"
      class="direction-button east"
      title="Face East"
      on:click={() =>
        faceDirection('E')
      }
    >
      E
    </button>


    <button
      type="button"
      class="direction-button south"
      title="Face South"
      on:click={() =>
        faceDirection('S')
      }
    >
      S
    </button>

  </div>

</div>


<style>

  .map-wrapper {
    position: relative;

    flex: 1;

    width: 100%;
    height: 100%;

    min-width: 0;
    min-height: 0;
  }


  .map {
    position: absolute;

    inset: 0;

    width: 100%;
    height: 100%;
  }


  /* =========================
     FLOATING COMPASS
     ========================= */

  .direction-pad {
    position: absolute;

    top: 82px;
    right: 10px;

    z-index: 20;

    display: grid;

    grid-template-columns:
      28px 28px 28px;

    grid-template-rows:
      28px 28px 28px;

    gap: 2px;

    padding: 5px;

    background:
      rgba(255, 255, 255, 0.94);

    border:
      1px solid rgba(
        0,
        0,
        0,
        0.15
      );

    border-radius: 8px;

    box-shadow:
      0 2px 8px
      rgba(0, 0, 0, 0.18);
  }


  .direction-button {
    width: 28px;
    height: 28px;

    padding: 0;

    border:
      1px solid #d1d5db;

    border-radius: 5px;

    background: white;

    font-size: 11px;
    font-weight: 700;

    cursor: pointer;

    color: #374151;
  }


  .direction-button:hover {
    background: #edf7f2;

    border-color: #0f6f4a;

    color: #0f6f4a;
  }


  .north {
    grid-column: 2;
    grid-row: 1;
  }


  .west {
    grid-column: 1;
    grid-row: 2;
  }


  .compass-center {
    grid-column: 2;
    grid-row: 2;

    display: flex;

    align-items: center;
    justify-content: center;

    width: 28px;
    height: 28px;

    border-radius: 50%;

    background: #f3f4f6;

    overflow: hidden;
  }


  .compass-needle {
    display: block;

    color: #dc2626;

    font-size: 14px;

    transform-origin: center;

    transition:
      transform 0.15s linear;
  }


  .east {
    grid-column: 3;
    grid-row: 2;
  }


  .south {
    grid-column: 2;
    grid-row: 3;
  }

/* =========================================================
   FARM ATTRIBUTE POPUP
   ========================================================= */

:global(.maplibregl-popup-content) {
  min-width: 260px;

  padding: 0;

  overflow: hidden;

  border-radius: 10px;

  background: #ffffff;

  box-shadow:
    0 8px 28px
    rgba(0, 0, 0, 0.22);

  font-family:
    Inter,
    system-ui,
    -apple-system,
    BlinkMacSystemFont,
    "Segoe UI",
    sans-serif;
}


:global(.maplibregl-popup-close-button) {
  top: 6px;
  right: 7px;

  width: 26px;
  height: 26px;

  padding: 0;

  border-radius: 6px;

  color: #6b7280;

  font-size: 20px;
  line-height: 24px;
}


:global(.maplibregl-popup-close-button:hover) {
  background: #edf7f2;

  color: #0f6f4a;
}


:global(.farm-popup-card) {
  width: 270px;

  box-sizing: border-box;

  color: #1f2937;
}


/* HEADER */

:global(.farm-popup-header) {
  display: flex;

  align-items: flex-start;
  justify-content: space-between;

  gap: 12px;

  padding:
    15px 38px 13px 16px;

  background:
    linear-gradient(
      135deg,
      #f1f8f4,
      #ffffff
    );
}


:global(.farm-popup-eyebrow) {
  margin-bottom: 4px;

  font-size: 9px;

  font-weight: 700;

  letter-spacing: 0.10em;

  color: #0f6f4a;
}


:global(.farm-popup-title) {
  max-width: 160px;

  overflow: hidden;

  text-overflow: ellipsis;

  white-space: nowrap;

  font-size: 14px;

  font-weight: 700;

  color: #111827;
}


:global(.farm-popup-fid) {
  margin-top: 16px;

  padding: 3px 6px;

  border: 1px solid #b7dbc9;

  border-radius: 5px;

  background: #edf7f2;

  color: #0f6f4a;

  font-size: 9px;

  font-weight: 700;
}


/* SEPARATOR */

:global(.farm-popup-divider) {
  height: 1px;

  background: #e5e7eb;
}


/* ATTRIBUTE GRID */

:global(.farm-popup-grid) {
  display: grid;

  grid-template-columns:
    82px 1fr;

  row-gap: 9px;

  padding: 13px 16px;
}


:global(.farm-popup-label) {
  font-size: 10px;

  color: #6b7280;
}


:global(.farm-popup-value) {
  min-width: 0;

  font-size: 11px;

  font-weight: 600;

  color: #252b33;

  word-break: break-word;
}


/* COORDINATE FOOTER */

:global(.farm-popup-coordinate) {
  display: flex;

  align-items: center;

  gap: 6px;

  padding:
    8px 16px;

  border-top:
    1px solid #edf0f2;

  background: #f8faf9;

  color: #6b7280;

  font-size: 9px;

  font-family:
    ui-monospace,
    SFMono-Regular,
    Consolas,
    monospace;
}


:global(.coordinate-dot) {
  width: 6px;
  height: 6px;

  flex-shrink: 0;

  border-radius: 50%;

  background: #22c55e;
}


/* POPUP ARROW */

:global(.maplibregl-popup-tip) {
  filter:
    drop-shadow(
      0 2px 2px
      rgba(0, 0, 0, 0.08)
    );
}

</style>