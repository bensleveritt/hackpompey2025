<script lang="ts">
  import MapComponent from '$lib/components/Map.svelte';

  const mapHeight = '100vh';
</script>

<main>
  <div class="map-wrapper">
    <MapComponent height={mapHeight} />
  </div>
</main>

<style>
  :global(body) {
    margin: 0;
  }

  main {
    height: 100vh;
    width: 100vw;
    position: relative;
    overflow: hidden;
  }

  .title-overlay {
    position: absolute;
    top: 1rem;
    left: 50%;
    transform: translateX(-50%);
    z-index: 1;
    background: rgba(255, 255, 255, 0.9);
    padding: 0.5rem 1.5rem;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  }

  h1 {
    color: #333;
    font-size: 2.5rem;
    margin: 0;
    text-align: center;
  }

  .map-wrapper {
    height: 100%;
    width: 100%;
  }
</style>
