<script lang="ts">
  import { goto } from '$app/navigation';
  import { page } from '$app/stores';
  import IntersectionObserver from '$lib/components/asset-viewer/intersection-observer.svelte';
  import CircleIconButton from '$lib/components/elements/buttons/circle-icon-button.svelte';
  import ControlAppBar from '$lib/components/shared-components/control-app-bar.svelte';
  import GalleryViewer from '$lib/components/shared-components/gallery-viewer/gallery-viewer.svelte';
  import { AppRoute, QueryParameter } from '$lib/constants';
  import type { Viewport } from '$lib/stores/assets.store';
  import { memoryStore } from '$lib/stores/memory.store';
  import { getAssetThumbnailUrl } from '$lib/utils';
  import { fromLocalDateTime } from '$lib/utils/timeline-util';
  import { ThumbnailFormat, getMemoryLane } from '@immich/sdk';
  import { mdiChevronDown, mdiChevronLeft, mdiChevronRight, mdiChevronUp, mdiPause, mdiPlay } from '@mdi/js';
  import { DateTime } from 'luxon';
  import { onMount } from 'svelte';
  import { tweened } from 'svelte/motion';
  import { fade } from 'svelte/transition';

  const parseIndex = (s: string | null, max: number | null) =>
    Math.max(Math.min(Number.parseInt(s ?? '') || 0, max ?? 0), 0);

  $: memoryIndex = parseIndex($page.url.searchParams.get(QueryParameter.MEMORY_INDEX), $memoryStore?.length - 1);
  $: assetIndex = parseIndex($page.url.searchParams.get(QueryParameter.ASSET_INDEX), currentMemory?.assets.length - 1);

  $: previousMemory = $memoryStore?.[memoryIndex - 1];
  $: currentMemory = $memoryStore?.[memoryIndex];
  $: nextMemory = $memoryStore?.[memoryIndex + 1];

  $: previousAsset = currentMemory?.assets[assetIndex - 1];
  $: currentAsset = currentMemory?.assets[assetIndex];
  $: nextAsset = currentMemory?.assets[assetIndex + 1];

  $: canGoForward = !!(nextMemory || nextAsset);
  $: canGoBack = !!(previousMemory || previousAsset);

  const viewport: Viewport = { width: 0, height: 0 };
  const toNextMemory = () => goto(`?${QueryParameter.MEMORY_INDEX}=${memoryIndex + 1}`);
  const toPreviousMemory = () => goto(`?${QueryParameter.MEMORY_INDEX}=${memoryIndex - 1}`);

  const toNextAsset = () =>
    goto(`?${QueryParameter.MEMORY_INDEX}=${memoryIndex}&${QueryParameter.ASSET_INDEX}=${assetIndex + 1}`);
  const toPreviousAsset = () =>
    goto(`?${QueryParameter.MEMORY_INDEX}=${memoryIndex}&${QueryParameter.ASSET_INDEX}=${assetIndex - 1}`);

  const toNext = () => (nextAsset ? toNextAsset() : toNextMemory());
  const toPrevious = () => (previousAsset ? toPreviousAsset() : toPreviousMemory());

  const progress = tweened<number>(0, {
    duration: (from: number, to: number) => (to ? 5000 * (to - from) : 0),
  });

  const play = () => progress.set(1);
  const pause = () => progress.set($progress);

  let resetPromise = Promise.resolve();
  const reset = () => (resetPromise = progress.set(0));

  let paused = false;

  // Play or pause progress when the paused state changes.
  $: paused ? pause() : play();

  // Progress should be paused when it's no longer possible to advance.
  $: paused ||= !canGoForward || galleryInView;

  // Advance to the next asset or memory when progress is complete.
  $: $progress === 1 && toNext();

  // Progress should be resumed when reset and not paused.
  $: !$progress && !paused && play();

  // Progress should be reset when the current memory or asset changes.
  $: memoryIndex, assetIndex, reset();

  const handleKeyDown = (e: KeyboardEvent) => {
    if (e.key === 'ArrowRight' && canGoForward) {
      e.preventDefault();
      toNext();
    } else if (e.key === 'ArrowLeft' && canGoBack) {
      e.preventDefault();
      toPrevious();
    } else if (e.key === 'Escape') {
      e.preventDefault();
      goto(AppRoute.PHOTOS);
    }
  };

  onMount(async () => {
    if (!$memoryStore) {
      const localTime = new Date();
      $memoryStore = await getMemoryLane({
        month: localTime.getMonth() + 1,
        day: localTime.getDate(),
      });
    }
  });

  let memoryGallery: HTMLElement;
  let memoryWrapper: HTMLElement;
  let galleryInView = false;
</script>

<svelte:window on:keydown={handleKeyDown} />

<section id="memory-viewer" class="w-full bg-immich-dark-gray" bind:this={memoryWrapper}>
  {#if currentMemory}
    <ControlAppBar on:close={() => goto(AppRoute.PHOTOS)} forceDark>
      <svelte:fragment slot="leading">
        <p class="text-lg">
          {currentMemory.title}
        </p>
      </svelte:fragment>

      {#if !galleryInView}
        <div class="flex place-content-center place-items-center gap-2 overflow-hidden">
          <CircleIconButton icon={paused ? mdiPlay : mdiPause} forceDark on:click={() => (paused = !paused)} />

          {#each currentMemory.assets as _, index}
            <button
              class="relative w-full py-2"
              on:click={() =>
                goto(`?${QueryParameter.MEMORY_INDEX}=${memoryIndex}&${QueryParameter.ASSET_INDEX}=${index}`)}
            >
              <span class="absolute left-0 h-[2px] w-full bg-gray-500" />
              {#await resetPromise}
                <span class="absolute left-0 h-[2px] bg-white" style:width={`${index < assetIndex ? 100 : 0}%`} />
              {:then}
                <span
                  class="absolute left-0 h-[2px] bg-white"
                  style:width={`${index < assetIndex ? 100 : index > assetIndex ? 0 : $progress * 100}%`}
                />
              {/await}
            </button>
          {/each}

          <div>
            <p class="text-small">
              {assetIndex + 1}/{currentMemory.assets.length}
            </p>
          </div>
        </div>
      {/if}
    </ControlAppBar>

    {#if galleryInView}
      <div
        class="sticky top-20 z-30 flex place-content-center place-items-center transition-opacity"
        class:opacity-0={!galleryInView}
        class:opacity-100={galleryInView}
      >
        <button on:click={() => memoryWrapper.scrollIntoView({ behavior: 'smooth' })} disabled={!galleryInView}>
          <CircleIconButton icon={mdiChevronUp} backgroundColor="white" forceDark />
        </button>
      </div>
    {/if}
    <!-- Viewer -->
    <section class="overflow-hidden pt-20">
      <div
        class="ml-[-100%] box-border flex h-[calc(100vh_-_180px)] w-[300%] items-center justify-center gap-10 overflow-hidden"
      >
        <!-- PREVIOUS MEMORY -->
        <div
          class="h-1/2 w-[20vw] rounded-2xl"
          class:opacity-25={previousMemory}
          class:opacity-0={!previousMemory}
          class:hover:opacity-70={previousMemory}
        >
          <button class="relative h-full w-full rounded-2xl" disabled={!previousMemory} on:click={toPreviousMemory}>
            {#if previousMemory}
              <img
                class="h-full w-full rounded-2xl object-cover"
                src={getAssetThumbnailUrl(previousMemory.assets[0].id, ThumbnailFormat.Jpeg)}
                alt=""
                draggable="false"
              />
            {:else}
              <enhanced:img
                class="h-full w-full rounded-2xl object-cover"
                src="$lib/assets/no-thumbnail.png"
                sizes="min(271px,186px)"
                alt=""
                draggable="false"
              />
            {/if}

            {#if previousMemory}
              <div class="absolute bottom-4 right-4 text-left text-white">
                <p class="text-xs font-semibold text-gray-200">PREVIOUS</p>
                <p class="text-xl">{previousMemory.title}</p>
              </div>
            {/if}
          </button>
        </div>

        <!-- CURRENT MEMORY -->
        <div
          class="main-view relative flex h-full w-[70vw] place-content-center place-items-center rounded-2xl bg-black"
        >
          <div class="relative h-full w-full rounded-2xl bg-black">
            {#key currentAsset.id}
              <img
                transition:fade
                class="h-full w-full rounded-2xl object-contain transition-all"
                src={getAssetThumbnailUrl(currentAsset.id, ThumbnailFormat.Jpeg)}
                alt=""
                draggable="false"
              />
            {/key}
            <!-- CONTROL BUTTONS -->
            {#if canGoBack}
              <div class="absolute top-1/2 left-0 ml-4">
                <CircleIconButton icon={mdiChevronLeft} backgroundColor="#202123" on:click={toPrevious} />
              </div>
            {/if}

            {#if canGoForward}
              <div class="absolute top-1/2 right-0 mr-4">
                <CircleIconButton icon={mdiChevronRight} backgroundColor="#202123" on:click={toNext} />
              </div>
            {/if}

            <div class="absolute left-8 top-4 text-sm font-medium text-white">
              <p>
                {fromLocalDateTime(currentMemory.assets[0].localDateTime).toLocaleString(DateTime.DATE_FULL)}
              </p>
              <p>
                {currentAsset.exifInfo?.city || ''}
                {currentAsset.exifInfo?.country || ''}
              </p>
            </div>
          </div>
        </div>

        <!-- NEXT MEMORY -->
        <div
          class="h-1/2 w-[20vw] rounded-xl"
          class:opacity-25={nextMemory}
          class:opacity-0={!nextMemory}
          class:hover:opacity-70={nextMemory}
        >
          <button class="relative h-full w-full rounded-2xl" on:click={toNextMemory} disabled={!nextMemory}>
            {#if nextMemory}
              <img
                class="h-full w-full rounded-2xl object-cover"
                src={getAssetThumbnailUrl(nextMemory.assets[0].id, ThumbnailFormat.Jpeg)}
                alt=""
                draggable="false"
              />
            {:else}
              <enhanced:img
                class="h-full w-full rounded-2xl object-cover"
                src="$lib/assets/no-thumbnail.png"
                sizes="min(271px,186px)"
                alt=""
                draggable="false"
              />
            {/if}

            {#if nextMemory}
              <div class="absolute bottom-4 left-4 text-left text-white">
                <p class="text-xs font-semibold text-gray-200">UP NEXT</p>
                <p class="text-xl">{nextMemory.title}</p>
              </div>
            {/if}
          </button>
        </div>
      </div>
    </section>

    <!-- GALERY VIEWER -->

    <section class="bg-immich-dark-gray m-4">
      <div
        class="sticky mb-10 mt-4 flex place-content-center place-items-center transition-all"
        class:opacity-0={galleryInView}
        class:opacity-100={!galleryInView}
      >
        <button on:click={() => memoryGallery.scrollIntoView({ behavior: 'smooth' })}>
          <CircleIconButton icon={mdiChevronDown} backgroundColor="white" forceDark />
        </button>
      </div>

      <IntersectionObserver
        once={false}
        on:intersected={() => (galleryInView = true)}
        on:hidden={() => (galleryInView = false)}
        bottom={-200}
      >
        <div
          id="gallery-memory"
          bind:this={memoryGallery}
          bind:clientHeight={viewport.height}
          bind:clientWidth={viewport.width}
        >
          <GalleryViewer assets={currentMemory.assets} {viewport} />
        </div>
      </IntersectionObserver>
    </section>
  {/if}
</section>

<style>
  .main-view {
    box-shadow:
      0 4px 4px 0 rgba(0, 0, 0, 0.3),
      0 8px 12px 6px rgba(0, 0, 0, 0.15);
  }
</style>
