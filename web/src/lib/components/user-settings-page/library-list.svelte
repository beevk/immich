<script lang="ts">
  import { api, UpdateLibraryDto, LibraryResponseDto, LibraryType, LibraryStatsResponseDto } from '@api';
  import { onMount } from 'svelte';
  import Button from '../elements/buttons/button.svelte';
  import { notificationController, NotificationType } from '../shared-components/notification/notification';
  import ConfirmDialogue from '../shared-components/confirm-dialogue.svelte';
  import { handleError } from '$lib/utils/handle-error';
  import { fade } from 'svelte/transition';
  import DotsVertical from 'svelte-material-icons/DotsVertical.svelte';
  import Database from 'svelte-material-icons/Database.svelte';
  import Upload from 'svelte-material-icons/Upload.svelte';
  import Pulse from 'svelte-loading-spinners/Pulse.svelte';
  import { slide } from 'svelte/transition';
  import LibraryImportPathsForm from '../forms/library-import-paths-form.svelte';
  import LibraryScanSettingsForm from '../forms/library-scan-settings-form.svelte';
  import LibraryRenameForm from '../forms/library-rename-form.svelte';
  import { getBytesWithUnit } from '$lib/utils/byte-units';
  import Portal from '../shared-components/portal/portal.svelte';
  import ContextMenu from '../shared-components/context-menu/context-menu.svelte';
  import MenuOption from '../shared-components/context-menu/menu-option.svelte';

  let libraries: LibraryResponseDto[] = [];

  let stats: LibraryStatsResponseDto[] = [];
  let photos: number[] = [];
  let videos: number[] = [];
  let totalCount: number[] = [];
  let diskUsage: number[] = [];
  let diskUsageUnit: string[] = [];

  let confirmDeleteLibrary: LibraryResponseDto | null = null;
  let deleteLibrary: LibraryResponseDto | null = null;

  let editImportPaths: number | null;
  let editScanSettings: number | null;
  let renameLibrary: number | null;

  let updateLibraryIndex: number | null;

  let deleteAssetCount = 0;

  let dropdownOpen: boolean[] = [];
  let showContextMenu = false;
  let contextMenuPosition = { x: 0, y: 0 };
  let libraryType: LibraryType;

  onMount(() => {
    readLibraryList();
  });

  const closeAll = () => {
    editImportPaths = null;
    editScanSettings = null;
    renameLibrary = null;
    updateLibraryIndex = null;
    showContextMenu = false;

    for (let i = 0; i < dropdownOpen.length; i++) {
      dropdownOpen[i] = false;
    }
  };

  const showMenu = ({ x, y }: MouseEvent, type: LibraryType) => {
    contextMenuPosition = { x, y };
    showContextMenu = !showContextMenu;
    libraryType = type;
  };

  const onMenuExit = () => {
    showContextMenu = false;
  };
  const refreshStats = async (listIndex: number) => {
    const { data } = await api.libraryApi.getLibraryStatistics({ id: libraries[listIndex].id });
    stats[listIndex] = data;
    photos[listIndex] = stats[listIndex].photos;
    videos[listIndex] = stats[listIndex].videos;
    totalCount[listIndex] = stats[listIndex].total;
    [diskUsage[listIndex], diskUsageUnit[listIndex]] = getBytesWithUnit(stats[listIndex].usage, 0);
  };

  async function readLibraryList() {
    const { data } = await api.libraryApi.getAllForUser();
    libraries = data;

    dropdownOpen.length = libraries.length;

    for (let i = 0; i < libraries.length; i++) {
      await refreshStats(i);
      dropdownOpen[i] = false;
    }
  }

  const handleCreate = async (libraryType: LibraryType) => {
    try {
      const { data } = await api.libraryApi.createLibrary({
        createLibraryDto: { type: libraryType },
      });

      const createdLibrary = data;

      notificationController.show({
        message: `Created library: ${createdLibrary.name}`,
        type: NotificationType.Info,
      });
    } catch (error) {
      handleError(error, 'Unable to create library');
    } finally {
      await readLibraryList();
    }
  };

  const handleUpdate = async (event: CustomEvent<UpdateLibraryDto>) => {
    if (updateLibraryIndex === null) {
      return;
    }

    try {
      const dto = event.detail;
      const libraryId = libraries[updateLibraryIndex].id;

      await api.libraryApi.updateLibrary({ id: libraryId, updateLibraryDto: dto });
    } catch (error) {
      handleError(error, 'Unable to update library');
    } finally {
      closeAll();
      await readLibraryList();
    }
  };

  const handleDelete = async () => {
    if (confirmDeleteLibrary) {
      deleteLibrary = confirmDeleteLibrary;
    }

    if (!deleteLibrary) {
      return;
    }

    try {
      await api.libraryApi.deleteLibrary({ id: deleteLibrary.id });
      notificationController.show({
        message: `Library deleted`,
        type: NotificationType.Info,
      });
    } catch (error) {
      handleError(error, 'Unable to remove library');
    } finally {
      confirmDeleteLibrary = null;
      deleteLibrary = null;
      await readLibraryList();
    }
  };

  const handleScanAll = async () => {
    try {
      for (const library of libraries) {
        if (library.type === LibraryType.External) {
          await api.libraryApi.scanLibrary({ id: library.id, scanLibraryDto: {} });
        }
      }
      notificationController.show({
        message: `Refreshing all libraries`,
        type: NotificationType.Info,
      });
    } catch (error) {
      handleError(error, 'Unable to scan libraries');
    }
  };

  const handleScan = async (libraryId: string) => {
    try {
      await api.libraryApi.scanLibrary({ id: libraryId, scanLibraryDto: {} });
      notificationController.show({
        message: `Scanning library for new files`,
        type: NotificationType.Info,
      });
    } catch (error) {
      handleError(error, 'Unable to scan library');
    }
  };

  const handleScanChanges = async (libraryId: string) => {
    try {
      await api.libraryApi.scanLibrary({ id: libraryId, scanLibraryDto: { refreshModifiedFiles: true } });
      notificationController.show({
        message: `Scanning library for changed files`,
        type: NotificationType.Info,
      });
    } catch (error) {
      handleError(error, 'Unable to scan library');
    }
  };

  const handleForceScan = async (libraryId: string) => {
    try {
      await api.libraryApi.scanLibrary({ id: libraryId, scanLibraryDto: { refreshAllFiles: true } });
      notificationController.show({
        message: `Forcing refresh of all library files`,
        type: NotificationType.Info,
      });
    } catch (error) {
      handleError(error, 'Unable to scan library');
    }
  };

  const handleRemoveOffline = async (libraryId: string) => {
    try {
      await api.libraryApi.removeOfflineFiles({ id: libraryId });
      notificationController.show({
        message: `Removing Offline Files`,
        type: NotificationType.Info,
      });
    } catch (error) {
      handleError(error, 'Unable to remove offline files');
    }
  };

  const onRenameClicked = (index: number) => {
    closeAll();
    renameLibrary = index;
    updateLibraryIndex = index;
  };

  const onEditImportPathClicked = (index: number) => {
    closeAll();
    editImportPaths = index;
    updateLibraryIndex = index;
  };

  const onScanNewLibraryClicked = (libraryId: string) => {
    closeAll();
    handleScan(libraryId);
  };

  const onScanSettingClicked = (index: number) => {
    closeAll();
    editScanSettings = index;
    updateLibraryIndex = index;
  };

  const onScanAllLibraryFilesClicked = (libraryId: string) => {
    closeAll();
    handleScanChanges(libraryId);
  };

  const onForceScanAllLibraryFilesClicked = (libraryId: string) => {
    closeAll();
    handleForceScan(libraryId);
  };

  const onRemoveOfflineFilesClicked = (libraryId: string) => {
    closeAll();
    handleRemoveOffline(libraryId);
  };

  const onDeleteLibraryClicked = (index: number, library: LibraryResponseDto) => {
    closeAll();

    if (confirm(`Are you sure you want to delete ${library.name} library?`) == true) {
      refreshStats(index);
      if (totalCount[index] > 0) {
        deleteAssetCount = totalCount[index];
        confirmDeleteLibrary = library;
      } else {
        deleteLibrary = library;
        handleDelete();
      }
    }
  };
</script>

{#if confirmDeleteLibrary}
  <ConfirmDialogue
    title="Warning!"
    prompt="Are you sure you want to delete this library? This will DELETE all {deleteAssetCount} contained assets and cannot be undone."
    on:confirm={handleDelete}
    on:cancel={() => (confirmDeleteLibrary = null)}
  />
{/if}

<section class="my-4">
  <div class="flex flex-col gap-2" in:fade={{ duration: 500 }}>
    {#if libraries.length > 0}
      <table class="w-full text-left">
        <thead
          class="mb-4 flex h-12 w-full rounded-md border bg-gray-50 text-immich-primary dark:border-immich-dark-gray dark:bg-immich-dark-gray dark:text-immich-dark-primary"
        >
          <tr class="flex w-full place-items-center">
            <th class="w-1/6 text-center text-sm font-medium">Type</th>
            <th class="w-1/3 text-center text-sm font-medium">Name</th>
            <th class="w-1/5 text-center text-sm font-medium">Assets</th>
            <th class="w-1/6 text-center text-sm font-medium">Size</th>
            <th class="w-1/6 text-center text-sm font-medium" />
          </tr>
        </thead>
        <tbody class="block w-full overflow-y-auto rounded-md border dark:border-immich-dark-gray">
          {#each libraries as library, index}
            {#key library.id}
              <tr
                class={`flex h-[80px] w-full place-items-center text-center dark:text-immich-dark-fg ${
                  index % 2 == 0
                    ? 'bg-immich-gray dark:bg-immich-dark-gray/75'
                    : 'bg-immich-bg dark:bg-immich-dark-gray/50'
                }`}
              >
                <td class="w-1/6 px-10 text-sm">
                  {#if library.type === LibraryType.External}
                    <Database size="40" title="External library (created on {library.createdAt})" />
                  {:else if library.type === LibraryType.Upload}
                    <Upload size="40" title="Upload library (created on {library.createdAt})" />
                  {/if}</td
                >

                <td class="w-1/3 text-ellipsis px-4 text-sm">{library.name}</td>
                {#if totalCount[index] == undefined}
                  <td colspan="2" class="flex w-1/3 items-center justify-center text-ellipsis px-4 text-sm">
                    <Pulse color="gray" size="40" unit="px" />
                  </td>
                {:else}
                  <td class="w-1/6 text-ellipsis px-4 text-sm">
                    {totalCount[index]}
                  </td>
                  <td class="w-1/6 text-ellipsis px-4 text-sm">{diskUsage[index]} {diskUsageUnit[index]} </td>
                {/if}

                <td class="w-1/6 text-ellipsis px-4 text-sm">
                  <button
                    class="rounded-full bg-immich-primary p-3 text-gray-100 transition-all duration-150 hover:bg-immich-primary/75 dark:bg-immich-dark-primary dark:text-gray-700"
                    on:click|stopPropagation|preventDefault={(e) => showMenu(e, library.type)}
                  >
                    <DotsVertical size="16" />
                  </button>

                  {#if showContextMenu}
                    <Portal target="body">
                      <ContextMenu {...contextMenuPosition} on:outclick={() => onMenuExit()}>
                        <MenuOption on:click={() => onRenameClicked(index)} text="Rename" />

                        {#if libraryType === LibraryType.External}
                          <MenuOption on:click={() => onEditImportPathClicked(index)} text="Edit Import Paths" />
                          <MenuOption on:click={() => onScanSettingClicked(index)} text="Scan Settings" />
                          <hr />
                          <MenuOption
                            on:click={() => onScanNewLibraryClicked(library.id)}
                            text="Scan New Library Files"
                          />
                          <MenuOption
                            on:click={() => onScanAllLibraryFilesClicked(library.id)}
                            text="Re-scan All Library Files"
                            subtitle={'Only refreshes modified files'}
                          />
                          <MenuOption
                            on:click={() => onForceScanAllLibraryFilesClicked(library.id)}
                            text="Force Re-scan All Library Files"
                            subtitle={'Refreshes every file'}
                          />
                          <hr />
                          <MenuOption
                            on:click={() => onRemoveOfflineFilesClicked(library.id)}
                            text="Remove Offline Files"
                          />
                          <MenuOption on:click={() => onDeleteLibraryClicked(index, library)}>
                            <p class="text-red-600">Delete library</p>
                          </MenuOption>
                        {/if}
                      </ContextMenu>
                    </Portal>
                  {/if}
                </td>
              </tr>
              {#if renameLibrary === index}
                <div transition:slide={{ duration: 250 }}>
                  <LibraryRenameForm {library} on:submit={handleUpdate} on:cancel={() => (renameLibrary = null)} />
                </div>
              {/if}
              {#if editImportPaths === index}
                <div transition:slide={{ duration: 250 }}>
                  <LibraryImportPathsForm
                    {library}
                    on:submit={handleUpdate}
                    on:cancel={() => (editImportPaths = null)}
                  />
                </div>
              {/if}
              {#if editScanSettings === index}
                <div transition:slide={{ duration: 250 }} class="mb-4 ml-4 mr-4">
                  <LibraryScanSettingsForm
                    {library}
                    on:submit={handleUpdate}
                    on:cancel={() => (editScanSettings = null)}
                  />
                </div>
              {/if}
            {/key}
          {/each}
        </tbody>
      </table>
    {/if}
    <div class="my-2 flex justify-end gap-2">
      <Button size="sm" on:click={() => handleScanAll()}>Scan All Libraries</Button>
      <Button size="sm" on:click={() => handleCreate(LibraryType.External)}>Create External Library</Button>
    </div>
  </div>
</section>
