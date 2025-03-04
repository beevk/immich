<script lang="ts">
  import { browser } from '$app/environment';
  import { goto } from '$app/navigation';
  import { AppRoute, AssetAction } from '$lib/constants';
  import type { AssetInteractionStore } from '$lib/stores/asset-interaction.store';
  import { assetViewingStore } from '$lib/stores/asset-viewing.store';
  import { BucketPosition, type AssetStore, type Viewport } from '$lib/stores/assets.store';
  import { locale } from '$lib/stores/preferences.store';
  import { isSearchEnabled } from '$lib/stores/search.store';
  import { formatGroupTitle, splitBucketIntoDateGroups } from '$lib/utils/timeline-util';
  import type { AssetResponseDto } from '@api';
  import { DateTime } from 'luxon';
  import { createEventDispatcher, onDestroy, onMount } from 'svelte';
  import AssetViewer from '../asset-viewer/asset-viewer.svelte';
  import IntersectionObserver from '../asset-viewer/intersection-observer.svelte';
  import Portal from '../shared-components/portal/portal.svelte';
  import Scrollbar from '../shared-components/scrollbar/scrollbar.svelte';
  import ShowShortcuts from '../shared-components/show-shortcuts.svelte';
  import AssetDateGroup from './asset-date-group.svelte';
  import { shouldIgnoreShortcut } from '$lib/utils/shortcut';

  export let isSelectionMode = false;
  export let singleSelect = false;
  export let assetStore: AssetStore;
  export let assetInteractionStore: AssetInteractionStore;
  export let removeAction: AssetAction | null = null;

  const { assetSelectionCandidates, assetSelectionStart, selectedGroup, selectedAssets, isMultiSelectState } =
    assetInteractionStore;
  const viewport: Viewport = { width: 0, height: 0 };
  let { isViewing: showAssetViewer, asset: viewingAsset } = assetViewingStore;
  let element: HTMLElement;
  let showShortcuts = false;
  let showSkeleton = true;

  $: timelineY = element?.scrollTop || 0;

  const onKeyboardPress = (event: KeyboardEvent) => handleKeyboardPress(event);
  const dispatch = createEventDispatcher<{ select: AssetResponseDto; escape: void }>();

  onMount(async () => {
    showSkeleton = false;
    document.addEventListener('keydown', onKeyboardPress);
    await assetStore.init(viewport);
  });

  onDestroy(() => {
    if (browser) {
      document.removeEventListener('keydown', onKeyboardPress);
    }

    if ($showAssetViewer) {
      $showAssetViewer = false;
    }
  });

  const handleKeyboardPress = (event: KeyboardEvent) => {
    if ($isSearchEnabled || shouldIgnoreShortcut(event)) {
      return;
    }

    if (!$showAssetViewer) {
      switch (event.key) {
        case 'Escape':
          dispatch('escape');
          return;
        case '?':
          if (event.shiftKey) {
            event.preventDefault();
            showShortcuts = !showShortcuts;
          }
          return;
        case '/':
          event.preventDefault();
          goto(AppRoute.EXPLORE);
          return;
      }
    }
  };

  const handleSelectAsset = (asset: AssetResponseDto) => {
    if (!assetStore.albumAssets.has(asset.id)) {
      assetInteractionStore.selectAsset(asset);
    }
  };

  function intersectedHandler(event: CustomEvent) {
    const el = event.detail.container as HTMLElement;
    const target = el.firstChild as HTMLElement;
    if (target) {
      const bucketDate = target.id.split('_')[1];
      assetStore.loadBucket(bucketDate, event.detail.position);
    }
  }

  function handleScrollTimeline(event: CustomEvent) {
    element.scrollBy(0, event.detail.heightDelta);
  }

  const handlePrevious = async () => {
    const previousAsset = await assetStore.getPreviousAssetId($viewingAsset.id);
    if (previousAsset) {
      assetViewingStore.setAssetId(previousAsset);
    }

    return !!previousAsset;
  };

  const handleNext = async () => {
    const nextAsset = await assetStore.getNextAssetId($viewingAsset.id);
    if (nextAsset) {
      assetViewingStore.setAssetId(nextAsset);
    }

    return !!nextAsset;
  };

  const handleClose = () => assetViewingStore.showAssetViewer(false);

  const handleAction = async (asset: AssetResponseDto, action: AssetAction) => {
    if (removeAction === action) {
      // find the next asset to show or close the viewer
      (await handleNext()) || (await handlePrevious()) || handleClose();

      // delete after find the next one
      assetStore.removeAsset(asset.id);
    }
  };

  let animationTick = false;

  const handleTimelineScroll = () => {
    if (animationTick) {
      return;
    }

    animationTick = true;
    window.requestAnimationFrame(() => {
      timelineY = element?.scrollTop || 0;
      animationTick = false;
    });
  };

  let lastAssetMouseEvent: AssetResponseDto | null = null;

  $: if (!lastAssetMouseEvent) {
    assetInteractionStore.clearAssetSelectionCandidates();
  }

  let shiftKeyIsDown = false;

  const onKeyDown = (e: KeyboardEvent) => {
    if ($isSearchEnabled) {
      return;
    }

    if (e.key == 'Shift') {
      e.preventDefault();
      shiftKeyIsDown = true;
    }
  };

  const onKeyUp = (e: KeyboardEvent) => {
    if ($isSearchEnabled) {
      return;
    }

    if (e.key == 'Shift') {
      e.preventDefault();
      shiftKeyIsDown = false;
    }
  };

  $: if (!shiftKeyIsDown) {
    assetInteractionStore.clearAssetSelectionCandidates();
  }

  $: if (shiftKeyIsDown && lastAssetMouseEvent) {
    selectAssetCandidates(lastAssetMouseEvent);
  }

  const handleSelectAssetCandidates = (asset: AssetResponseDto | null) => {
    if (asset) {
      selectAssetCandidates(asset);
    }
    lastAssetMouseEvent = asset;
  };

  const handleGroupSelect = (group: string, assets: AssetResponseDto[]) => {
    if ($selectedGroup.has(group)) {
      assetInteractionStore.removeGroupFromMultiselectGroup(group);
      for (const asset of assets) {
        assetInteractionStore.removeAssetFromMultiselectGroup(asset);
      }
    } else {
      assetInteractionStore.addGroupToMultiselectGroup(group);
      for (const asset of assets) {
        handleSelectAsset(asset);
      }
    }
  };

  const handleSelectAssets = async (asset: AssetResponseDto) => {
    if (!asset) {
      return;
    }

    dispatch('select', asset);

    if (singleSelect) {
      element.scrollTop = 0;
      return;
    }

    const rangeSelection = $assetSelectionCandidates.size > 0;
    const deselect = $selectedAssets.has(asset);

    // Select/deselect already loaded assets
    if (deselect) {
      for (const candidate of $assetSelectionCandidates || []) {
        assetInteractionStore.removeAssetFromMultiselectGroup(candidate);
      }
      assetInteractionStore.removeAssetFromMultiselectGroup(asset);
    } else {
      for (const candidate of $assetSelectionCandidates || []) {
        handleSelectAsset(candidate);
      }
      handleSelectAsset(asset);
    }

    assetInteractionStore.clearAssetSelectionCandidates();

    if ($assetSelectionStart && rangeSelection) {
      let startBucketIndex = $assetStore.getBucketIndexByAssetId($assetSelectionStart.id);
      let endBucketIndex = $assetStore.getBucketIndexByAssetId(asset.id);

      if (startBucketIndex === null || endBucketIndex === null) {
        return;
      }

      if (endBucketIndex < startBucketIndex) {
        [startBucketIndex, endBucketIndex] = [endBucketIndex, startBucketIndex];
      }

      // Select/deselect assets in all intermediate buckets
      for (let bucketIndex = startBucketIndex + 1; bucketIndex < endBucketIndex; bucketIndex++) {
        const bucket = $assetStore.buckets[bucketIndex];
        await assetStore.loadBucket(bucket.bucketDate, BucketPosition.Unknown);
        for (const asset of bucket.assets) {
          if (deselect) {
            assetInteractionStore.removeAssetFromMultiselectGroup(asset);
          } else {
            handleSelectAsset(asset);
          }
        }
      }

      // Update date group selection
      for (let bucketIndex = startBucketIndex; bucketIndex <= endBucketIndex; bucketIndex++) {
        const bucket = $assetStore.buckets[bucketIndex];

        // Split bucket into date groups and check each group
        const assetsGroupByDate = splitBucketIntoDateGroups(bucket.assets, $locale);

        for (const dateGroup of assetsGroupByDate) {
          const dateGroupTitle = formatGroupTitle(DateTime.fromISO(dateGroup[0].fileCreatedAt).startOf('day'));
          if (dateGroup.every((a) => $selectedAssets.has(a))) {
            assetInteractionStore.addGroupToMultiselectGroup(dateGroupTitle);
          } else {
            assetInteractionStore.removeGroupFromMultiselectGroup(dateGroupTitle);
          }
        }
      }
    }

    assetInteractionStore.setAssetSelectionStart(deselect ? null : asset);
  };

  const selectAssetCandidates = (asset: AssetResponseDto) => {
    if (!shiftKeyIsDown) {
      return;
    }

    const rangeStart = $assetSelectionStart;
    if (!rangeStart) {
      return;
    }

    let start = $assetStore.assets.indexOf(rangeStart);
    let end = $assetStore.assets.indexOf(asset);

    if (start > end) {
      [start, end] = [end, start];
    }

    assetInteractionStore.setAssetSelectionCandidates($assetStore.assets.slice(start, end + 1));
  };

  const onSelectStart = (e: Event) => {
    if ($isMultiSelectState && shiftKeyIsDown) {
      e.preventDefault();
    }
  };
</script>

<svelte:window on:keydown={onKeyDown} on:keyup={onKeyUp} on:selectstart={onSelectStart} />

{#if showShortcuts}
  <ShowShortcuts on:close={() => (showShortcuts = !showShortcuts)} />
{/if}

<Scrollbar
  {assetStore}
  height={viewport.height}
  {timelineY}
  on:scrollTimeline={({ detail }) => (element.scrollTop = detail)}
/>

<!-- Right margin MUST be equal to the width of immich-scrubbable-scrollbar -->
<section
  id="asset-grid"
  class="scrollbar-hidden ml-4 mr-[60px] h-full overflow-y-auto pb-[60px]"
  bind:clientHeight={viewport.height}
  bind:clientWidth={viewport.width}
  bind:this={element}
  on:scroll={handleTimelineScroll}
>
  <!-- skeleton -->
  {#if showSkeleton}
    <div class="mt-8 animate-pulse">
      <div class="mb-2 h-4 w-24 rounded-full bg-immich-primary/20 dark:bg-immich-dark-primary/20" />
      <div class="flex w-[120%] flex-wrap">
        {#each Array(100) as _}
          <div class="m-[1px] h-[10em] w-[16em] bg-immich-primary/20 dark:bg-immich-dark-primary/20" />
        {/each}
      </div>
    </div>
  {/if}

  {#if element}
    <slot />

    <!-- (optional) empty placeholder -->
    {#if $assetStore.initialized && $assetStore.buckets.length === 0}
      <slot name="empty" />
    {/if}
    <section id="virtual-timeline" style:height={$assetStore.timelineHeight + 'px'}>
      {#each $assetStore.buckets as bucket, bucketIndex (bucketIndex)}
        <IntersectionObserver
          on:intersected={intersectedHandler}
          on:hidden={() => assetStore.cancelBucket(bucket)}
          let:intersecting
          top={750}
          bottom={750}
          root={element}
        >
          <div id={'bucket_' + bucket.bucketDate} style:height={bucket.bucketHeight + 'px'}>
            {#if intersecting}
              <AssetDateGroup
                {assetStore}
                {assetInteractionStore}
                {isSelectionMode}
                {singleSelect}
                on:select={({ detail: group }) => handleGroupSelect(group.title, group.assets)}
                on:shift={handleScrollTimeline}
                on:selectAssetCandidates={({ detail: asset }) => handleSelectAssetCandidates(asset)}
                on:selectAssets={({ detail: asset }) => handleSelectAssets(asset)}
                assets={bucket.assets}
                bucketDate={bucket.bucketDate}
                bucketHeight={bucket.bucketHeight}
                {viewport}
              />
            {/if}
          </div>
        </IntersectionObserver>
      {/each}
    </section>
  {/if}
</section>

<Portal target="body">
  {#if $showAssetViewer}
    <AssetViewer
      {assetStore}
      asset={$viewingAsset}
      on:previous={() => handlePrevious()}
      on:next={() => handleNext()}
      on:close={() => handleClose()}
      on:archived={({ detail: asset }) => handleAction(asset, AssetAction.ARCHIVE)}
      on:unarchived={({ detail: asset }) => handleAction(asset, AssetAction.UNARCHIVE)}
      on:favorite={({ detail: asset }) => handleAction(asset, AssetAction.FAVORITE)}
      on:unfavorite={({ detail: asset }) => handleAction(asset, AssetAction.UNFAVORITE)}
    />
  {/if}
</Portal>

<style>
  #asset-grid {
    contain: layout;
    scrollbar-width: none;
  }
</style>
