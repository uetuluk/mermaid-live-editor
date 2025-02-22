<script lang="ts">
  import { inputStateStore, stateStore, updateCodeStore } from '$lib/util/state';
  import { onMount } from 'svelte';
  import panzoom from 'svg-pan-zoom';
  import type { State, ValidatedState } from '$lib/types';
  import { logEvent } from '$lib/util/stats';
  import { cmdKey } from '$lib/util/util';
  import { render as renderDiagram } from '$lib/util/mermaid';
  import type { MermaidConfig } from 'mermaid';

  let code = '';
  let config = '';
  let container: HTMLDivElement;
  let view: HTMLDivElement;
  let error = false;
  let outOfSync = false;
  let hide = false;
  let manualUpdate = true;
  let panZoomEnabled = $stateStore.panZoom;
  let pzoom: typeof panzoom | undefined;

  const handlePanZoomChange = () => {
    if (!pzoom) {
      return;
    }
    const pan = pzoom.getPan();
    const zoom = pzoom.getZoom();
    updateCodeStore({ pan, zoom });
    logEvent('panZoom');
  };

  const handlePanZoom = (state: State) => {
    if (!state.panZoom) {
      return;
    }
    hide = true;
    pzoom?.destroy();
    pzoom = undefined;
    void Promise.resolve().then(() => {
      const graphDiv = document.getElementById('graph-div');
      if (!graphDiv) {
        return;
      }
      pzoom = panzoom(graphDiv, {
        onPan: handlePanZoomChange,
        onZoom: handlePanZoomChange,
        controlIconsEnabled: true,
        fit: true,
        center: true
      });
      const { pan, zoom } = state;
      if (pan !== undefined && zoom !== undefined && Number.isFinite(zoom)) {
        pzoom.zoom(zoom);
        pzoom.pan(pan);
      }
      hide = false;
    });
  };

  const handleStateChange = async (state: ValidatedState) => {
    if (state.error !== undefined) {
      error = true;
      return;
    }
    error = false;
    try {
      if (container && state && (state.updateDiagram || state.autoSync)) {
        if (!state.autoSync) {
          $inputStateStore.updateDiagram = false;
        }
        outOfSync = false;
        manualUpdate = true;
        // Do not render if there is no change in Code/Config/PanZoom
        if (code === state.code && config === state.mermaid && panZoomEnabled === state.panZoom) {
          return;
        }
        code = state.code;
        config = state.mermaid;
        panZoomEnabled = state.panZoom;
        const scroll = view.parentElement!.scrollTop;
        delete container.dataset.processed;
        const { svg, bindFunctions } = await renderDiagram(
          Object.assign({}, JSON.parse(state.mermaid)) as MermaidConfig,
          code,
          'graph-div'
        );

        if (svg.length > 0) {
          handlePanZoom(state);
          container.innerHTML = svg;
          console.log({ svg });
          const graphDiv = document.getElementById('graph-div');
          if (!graphDiv) {
            throw new Error('graph-div not found');
          }
          graphDiv.setAttribute('height', '100%');
          graphDiv.style.maxWidth = '100%';
          if (bindFunctions) {
            bindFunctions(graphDiv);
          }
        }

        view.parentElement!.scrollTop = scroll;
        error = false;
      } else if (manualUpdate) {
        manualUpdate = false;
      } else if (code !== state.code || config !== state.mermaid) {
        outOfSync = true;
      }
    } catch (e) {
      console.error('view fail', e);
      error = true;
    }
  };

  onMount(() => {
    stateStore.subscribe((state) => {
      void handleStateChange(state);
    });
    window.addEventListener('resize', () => {
      if ($stateStore.panZoom && pzoom) {
        pzoom.resize();
      }
    });
  });
</script>

{#if (error && $stateStore.error instanceof Error) || outOfSync}
  <div
    class="absolute w-full p-2 z-10 font-mono {error
      ? 'text-red-600'
      : 'text-yellow-600'} bg-base-100 bg-opacity-80 text-left"
    id="errorContainer">
    {#if error}
      {@html $stateStore.error?.toString().replace(/\n/g, '<br />')}
    {:else}
      Diagram out of sync. <br />
      Press <i class="fas fa-sync" /> (Sync button) or <kbd>{cmdKey} + Enter</kbd> to sync.
    {/if}
  </div>
{/if}

<div id="view" bind:this={view} class="p-2 h-full" class:error class:outOfSync>
  <div id="container" bind:this={container} class="h-full overflow-auto" class:hide />
</div>

<style>
  #view {
    flex: 1;
  }

  #container {
    transition: visibility 0.3s;
  }

  .error,
  .outOfSync {
    opacity: 0.5;
  }

  .hide {
    visibility: hidden;
  }
</style>
