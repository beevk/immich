<script lang="ts">
  import { afterNavigate, goto, invalidateAll } from '$app/navigation';
  import { page } from '$app/stores';
  import ImageThumbnail from '$lib/components/assets/thumbnail/image-thumbnail.svelte';
  import EditNameInput from '$lib/components/faces-page/edit-name-input.svelte';
  import MergeFaceSelector from '$lib/components/faces-page/merge-face-selector.svelte';
  import MergeSuggestionModal from '$lib/components/faces-page/merge-suggestion-modal.svelte';
  import SetBirthDateModal from '$lib/components/faces-page/set-birth-date-modal.svelte';
  import AddToAlbum from '$lib/components/photos-page/actions/add-to-album.svelte';
  import ArchiveAction from '$lib/components/photos-page/actions/archive-action.svelte';
  import CreateSharedLink from '$lib/components/photos-page/actions/create-shared-link.svelte';
  import DeleteAssets from '$lib/components/photos-page/actions/delete-assets.svelte';
  import DownloadAction from '$lib/components/photos-page/actions/download-action.svelte';
  import FavoriteAction from '$lib/components/photos-page/actions/favorite-action.svelte';
  import SelectAllAssets from '$lib/components/photos-page/actions/select-all-assets.svelte';
  import AssetGrid from '$lib/components/photos-page/asset-grid.svelte';
  import AssetSelectContextMenu from '$lib/components/photos-page/asset-select-context-menu.svelte';
  import AssetSelectControlBar from '$lib/components/photos-page/asset-select-control-bar.svelte';
  import MenuOption from '$lib/components/shared-components/context-menu/menu-option.svelte';
  import ControlAppBar from '$lib/components/shared-components/control-app-bar.svelte';
  import {
    NotificationType,
    notificationController,
  } from '$lib/components/shared-components/notification/notification';
  import { AppRoute } from '$lib/constants';
  import { createAssetInteractionStore } from '$lib/stores/asset-interaction.store';
  import { AssetStore } from '$lib/stores/assets.store';
  import { handleError } from '$lib/utils/handle-error';
  import { AssetResponseDto, PersonResponseDto, TimeBucketSize, api } from '@api';
  import { onMount } from 'svelte';
  import ArrowLeft from 'svelte-material-icons/ArrowLeft.svelte';
  import DotsVertical from 'svelte-material-icons/DotsVertical.svelte';
  import Plus from 'svelte-material-icons/Plus.svelte';
  import type { PageData } from './$types';
  import { clickOutside } from '$lib/utils/click-outside';
  import { assetViewingStore } from '$lib/stores/asset-viewing.store';

  export let data: PageData;

  let { isViewing: showAssetViewer } = assetViewingStore;

  enum ViewMode {
    VIEW_ASSETS = 'view-assets',
    SELECT_FACE = 'select-face',
    MERGE_FACES = 'merge-faces',
    SUGGEST_MERGE = 'suggest-merge',
    BIRTH_DATE = 'birth-date',
  }

  let assetStore = new AssetStore({
    size: TimeBucketSize.Month,
    isArchived: false,
    personId: data.person.id,
  });
  const assetInteractionStore = createAssetInteractionStore();
  const { selectedAssets, isMultiSelectState } = assetInteractionStore;

  let viewMode: ViewMode = ViewMode.VIEW_ASSETS;
  let isEditingName = false;
  let previousRoute: string = AppRoute.EXPLORE;
  let previousPersonId: string = data.person.id;
  let people = data.people.people;
  let personMerge1: PersonResponseDto;
  let personMerge2: PersonResponseDto;
  let potentialMergePeople: PersonResponseDto[] = [];

  let personName = '';

  let name: string = data.person.name;
  let suggestedPeople: PersonResponseDto[] = [];

  $: isAllArchive = Array.from($selectedAssets).every((asset) => asset.isArchived);
  $: isAllFavorite = Array.from($selectedAssets).every((asset) => asset.isFavorite);

  $: {
    suggestedPeople = !name
      ? []
      : people
          .filter(
            (person: PersonResponseDto) =>
              person.name.toLowerCase().startsWith(name.toLowerCase()) && person.id !== data.person.id,
          )
          .slice(0, 5);
  }

  onMount(() => {
    const action = $page.url.searchParams.get('action');
    if (action == 'merge') {
      viewMode = ViewMode.MERGE_FACES;
    }
  });
  const handleEscape = () => {
    if ($showAssetViewer) {
      return;
    }
    if ($isMultiSelectState) {
      assetInteractionStore.clearMultiselect();
      return;
    } else {
      goto(previousRoute);
      return;
    }
  };
  afterNavigate(({ from }) => {
    // Prevent setting previousRoute to the current page.
    if (from && from.route.id !== $page.route.id) {
      previousRoute = from.url.href;
    }
    if (previousPersonId !== data.person.id) {
      assetStore = new AssetStore({
        size: TimeBucketSize.Month,
        isArchived: false,
        personId: data.person.id,
      });
      previousPersonId = data.person.id;
    }
  });

  const hideFace = async () => {
    try {
      await api.personApi.updatePerson({
        id: data.person.id,
        personUpdateDto: { isHidden: true },
      });

      notificationController.show({
        message: 'Changed visibility succesfully',
        type: NotificationType.Info,
      });

      goto(AppRoute.EXPLORE, { replaceState: true });
    } catch (error) {
      handleError(error, 'Unable to hide person');
    }
  };

  const handleSelectFeaturePhoto = async (asset: AssetResponseDto) => {
    if (viewMode !== ViewMode.SELECT_FACE) {
      return;
    }

    await api.personApi.updatePerson({ id: data.person.id, personUpdateDto: { featureFaceAssetId: asset.id } });

    // TODO: Replace by Websocket in the future
    notificationController.show({
      message: 'Feature photo updated, refresh page to see changes',
      type: NotificationType.Info,
    });

    assetInteractionStore.clearMultiselect();
    // scroll to top

    viewMode = ViewMode.VIEW_ASSETS;
  };

  const handleMergeSameFace = async (response: [PersonResponseDto, PersonResponseDto]) => {
    const [personToMerge, personToBeMergedIn] = response;
    viewMode = ViewMode.VIEW_ASSETS;
    isEditingName = false;
    try {
      await api.personApi.mergePerson({
        id: personToBeMergedIn.id,
        mergePersonDto: { ids: [personToMerge.id] },
      });
      notificationController.show({
        message: 'Merge faces succesfully',
        type: NotificationType.Info,
      });
      people = people.filter((person: PersonResponseDto) => person.id !== personToMerge.id);
      if (personToBeMergedIn.name != personName && data.person.id === personToBeMergedIn.id) {
        changeName();
        invalidateAll();
        return;
      }
      goto(`${AppRoute.PEOPLE}/${personToBeMergedIn.id}`, { replaceState: true });
    } catch (error) {
      handleError(error, 'Unable to save name');
    }
  };

  const handleSuggestPeople = (person: PersonResponseDto) => {
    isEditingName = false;
    potentialMergePeople = [];
    personMerge1 = data.person;
    personMerge2 = person;
    viewMode = ViewMode.SUGGEST_MERGE;
  };

  const changeName = async () => {
    viewMode = ViewMode.VIEW_ASSETS;
    data.person.name = personName;
    try {
      isEditingName = false;

      const { data: updatedPerson } = await api.personApi.updatePerson({
        id: data.person.id,
        personUpdateDto: { name: personName },
      });

      people = people.map((person: PersonResponseDto) => {
        if (person.id === updatedPerson.id) {
          return updatedPerson;
        }
        return person;
      });

      notificationController.show({
        message: 'Change name succesfully',
        type: NotificationType.Info,
      });
    } catch (error) {
      handleError(error, 'Unable to save name');
    }
  };

  const handleCancelEditName = () => {
    if (viewMode === ViewMode.SUGGEST_MERGE) {
      return;
    }

    isEditingName = false;
  };

  const handleNameChange = async (name: string) => {
    potentialMergePeople = [];
    personName = name;

    if (data.person.name === personName) {
      return;
    }

    const existingPerson = people.find(
      (person: PersonResponseDto) =>
        person.name.toLowerCase() === personName.toLowerCase() && person.id !== data.person.id && person.name,
    );
    if (existingPerson) {
      personMerge2 = existingPerson;
      personMerge1 = data.person;
      potentialMergePeople = people
        .filter(
          (person: PersonResponseDto) =>
            personMerge2.name.toLowerCase() === person.name.toLowerCase() &&
            person.id !== personMerge2.id &&
            person.id !== personMerge1.id &&
            !person.isHidden,
        )
        .slice(0, 3);
      viewMode = ViewMode.SUGGEST_MERGE;
      return;
    }
    changeName();
  };

  const handleSetBirthDate = async (birthDate: string) => {
    try {
      viewMode = ViewMode.VIEW_ASSETS;
      data.person.birthDate = birthDate;

      const { data: updatedPerson } = await api.personApi.updatePerson({
        id: data.person.id,
        personUpdateDto: { birthDate: birthDate.length > 0 ? birthDate : null },
      });

      people = people.map((person: PersonResponseDto) => {
        if (person.id === updatedPerson.id) {
          return updatedPerson;
        }
        return person;
      });

      notificationController.show({ message: 'Date of birth saved successfully', type: NotificationType.Info });
    } catch (error) {
      handleError(error, 'Unable to save date of birth');
    }
  };

  const handleGoBack = () => {
    viewMode = ViewMode.VIEW_ASSETS;
    if ($page.url.searchParams.has('action')) {
      $page.url.searchParams.delete('action');
      goto($page.url);
    }
  };
</script>

{#if viewMode === ViewMode.SUGGEST_MERGE}
  <MergeSuggestionModal
    {personMerge1}
    {personMerge2}
    {potentialMergePeople}
    on:close={() => (viewMode = ViewMode.VIEW_ASSETS)}
    on:reject={() => changeName()}
    on:confirm={(event) => handleMergeSameFace(event.detail)}
  />
{/if}

{#if viewMode === ViewMode.BIRTH_DATE}
  <SetBirthDateModal
    birthDate={data.person.birthDate ?? ''}
    on:close={() => (viewMode = ViewMode.VIEW_ASSETS)}
    on:updated={(event) => handleSetBirthDate(event.detail)}
  />
{/if}

{#if viewMode === ViewMode.MERGE_FACES}
  <MergeFaceSelector person={data.person} on:go-back={handleGoBack} />
{/if}

<header>
  {#if $isMultiSelectState}
    <AssetSelectControlBar assets={$selectedAssets} clearSelect={() => assetInteractionStore.clearMultiselect()}>
      <CreateSharedLink />
      <SelectAllAssets {assetStore} {assetInteractionStore} />
      <AssetSelectContextMenu icon={Plus} title="Add">
        <AddToAlbum />
        <AddToAlbum shared />
      </AssetSelectContextMenu>
      <DeleteAssets onAssetDelete={(assetId) => $assetStore.removeAsset(assetId)} />
      <AssetSelectContextMenu icon={DotsVertical} title="Add">
        <DownloadAction menuItem filename="{data.person.name || 'immich'}.zip" />
        <FavoriteAction menuItem removeFavorite={isAllFavorite} />
        <ArchiveAction menuItem unarchive={isAllArchive} onArchive={(ids) => $assetStore.removeAssets(ids)} />
      </AssetSelectContextMenu>
    </AssetSelectControlBar>
  {:else}
    {#if viewMode === ViewMode.VIEW_ASSETS || viewMode === ViewMode.SUGGEST_MERGE || viewMode === ViewMode.BIRTH_DATE}
      <ControlAppBar showBackButton backIcon={ArrowLeft} on:close-button-click={() => goto(previousRoute)}>
        <svelte:fragment slot="trailing">
          <AssetSelectContextMenu icon={DotsVertical} title="Menu">
            <MenuOption text="Change feature photo" on:click={() => (viewMode = ViewMode.SELECT_FACE)} />
            <MenuOption text="Set date of birth" on:click={() => (viewMode = ViewMode.BIRTH_DATE)} />
            <MenuOption text="Merge face" on:click={() => (viewMode = ViewMode.MERGE_FACES)} />
            <MenuOption text="Hide face" on:click={() => hideFace()} />
          </AssetSelectContextMenu>
        </svelte:fragment>
      </ControlAppBar>
    {/if}

    {#if viewMode === ViewMode.SELECT_FACE}
      <ControlAppBar on:close-button-click={() => (viewMode = ViewMode.VIEW_ASSETS)}>
        <svelte:fragment slot="leading">Select feature photo</svelte:fragment>
      </ControlAppBar>
    {/if}
  {/if}
</header>

<main class="relative h-screen overflow-hidden bg-immich-bg pt-[var(--navbar-height)] dark:bg-immich-dark-bg">
  {#key previousPersonId}
    <AssetGrid
      {assetStore}
      {assetInteractionStore}
      isSelectionMode={viewMode === ViewMode.SELECT_FACE}
      singleSelect={viewMode === ViewMode.SELECT_FACE}
      on:select={({ detail: asset }) => handleSelectFeaturePhoto(asset)}
      on:escape={handleEscape}
    >
      {#if viewMode === ViewMode.VIEW_ASSETS || viewMode === ViewMode.SUGGEST_MERGE || viewMode === ViewMode.BIRTH_DATE}
        <!-- Face information block -->
        <div
          role="button"
          class="relative w-fit p-4 sm:px-6"
          use:clickOutside
          on:outclick={handleCancelEditName}
          on:escape={handleCancelEditName}
        >
          <section class="flex w-96 place-items-center border-black">
            {#if isEditingName}
              <EditNameInput
                person={data.person}
                suggestedPeople={suggestedPeople.length > 0}
                bind:name
                on:change={(event) => handleNameChange(event.detail)}
              />
            {:else}
              <button on:click={() => (viewMode = ViewMode.VIEW_ASSETS)}>
                <ImageThumbnail
                  circle
                  shadow
                  url={api.getPeopleThumbnailUrl(data.person.id)}
                  altText={data.person.name}
                  widthStyle="3.375rem"
                  heightStyle="3.375rem"
                />
              </button>

              <button
                title="Edit name"
                class="px-4 text-immich-primary dark:text-immich-dark-primary"
                on:click={() => (isEditingName = true)}
              >
                {#if data.person.name}
                  <p class="py-2 font-medium">{data.person.name}</p>
                {:else}
                  <p class="w-fit font-medium">Add a name</p>
                  <p class="text-sm text-gray-500 dark:text-immich-gray">Find them fast by name with search</p>
                {/if}
              </button>
            {/if}
          </section>
          {#if isEditingName}
            <div class="absolute z-[999] w-96">
              {#each suggestedPeople as person, index (person.id)}
                <div
                  class="flex {index === suggestedPeople.length - 1
                    ? 'rounded-b-lg'
                    : 'border-b dark:border-immich-dark-gray'} place-items-center bg-gray-100 p-2 dark:bg-gray-700"
                >
                  <button class="flex w-full place-items-center" on:click={() => handleSuggestPeople(person)}>
                    <ImageThumbnail
                      circle
                      shadow
                      url={api.getPeopleThumbnailUrl(person.id)}
                      altText={person.name}
                      widthStyle="2rem"
                      heightStyle="2rem"
                    />
                    <p class="ml-4 text-gray-700 dark:text-gray-100">{person.name}</p>
                  </button>
                </div>
              {/each}
            </div>
          {/if}
        </div>
      {/if}
    </AssetGrid>
  {/key}
</main>
