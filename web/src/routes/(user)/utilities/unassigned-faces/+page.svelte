<script lang="ts">
  import UserPageLayout from '$lib/components/layouts/UserPageLayout.svelte';
  import ImageThumbnail from '$lib/components/assets/thumbnail/ImageThumbnail.svelte';
  import FaceCandidateRow from '$lib/components/faces-page/FaceCandidateRow.svelte';
  import FaceHighlightOverlay from '$lib/components/faces-page/FaceHighlightOverlay.svelte';
  import MagnifierLens from '$lib/components/faces-page/MagnifierLens.svelte';
  import PeopleSearch from '$lib/components/faces-page/PeopleSearch.svelte';
  import { getAssetMediaUrl, getPeopleThumbnailUrl } from '$lib/utils';
  import { handleError } from '$lib/utils/handle-error';
  import { getContentMetrics, mapNormalizedRectToContent } from '$lib/utils/container-utils';
  import { shortcuts } from '$lib/actions/shortcut';
  import { locale } from '$lib/stores/preferences.store';
  import ShortcutsModal from '$lib/modals/ShortcutsModal.svelte';
  import {
    searchRandom,
    getFaceCandidates,
    mergePerson,
    updatePeople,
    reassignFacesById,
    createPerson,
    getFaces,
    AssetMediaSize,
    type AssetResponseDto,
    type AssetFaceResponseDto,
    type FaceCandidateDto,
    type PersonResponseDto,
    type PersonWithFacesResponseDto,
  } from '@immich/sdk';
  import { Button, HStack, IconButton, LoadingSpinner, Text, modalManager, toastManager } from '@immich/ui';
  import { mdiCheck, mdiClose, mdiEyeOffOutline, mdiSkipNext, mdiPlus, mdiKeyboard } from '@mdi/js';
  import { onMount } from 'svelte';
  import { t } from 'svelte-i18n';
  import type { PageData } from './$types';

  interface Props {
    data: PageData;
  }

  let { data = $bindable() }: Props = $props();

  interface Shortcuts {
    general: ExplainedShortcut[];
    actions: ExplainedShortcut[];
  }
  interface ExplainedShortcut {
    key: string[];
    action: string;
    info?: string;
  }

  const faceReviewShortcuts: Shortcuts = {
    general: [],
    actions: [
      { key: ['y'], action: $t('yes') },
      { key: ['n'], action: $t('no') },
      { key: ['h'], action: $t('hide') },
      { key: ['s'], action: $t('skip') },
    ],
  };

  let loading = $state(true);
  let queue: AssetResponseDto[] = $state([]);
  let currentAsset: AssetResponseDto | null = $state(null);
  let currentUntaggedPerson: PersonWithFacesResponseDto | null = $state(null);
  let currentFace: AssetFaceResponseDto | null = $state(null);
  let candidates: FaceCandidateDto[] = $state([]);
  let candidateIndex = $state(0);
  let isSubmitting = $state(false);
  let empty = $state(false);
  let totalReviewed = $state(0);

  let imgEl: HTMLImageElement | null = $state(null);
  let imageReady = $state(false);
  let error = $state<string | null>(null);

  let hover = $state(false);
  let mouse = $state({ x: 0, y: 0 });

  let reviewedIds = $state(new Set<string>());

  // PeopleSearch bindings
  let searchName = $state('');
  let searchedPeople: PersonResponseDto[] = $state([]);
  let isSearching = $state(false);

  let currentCandidate = $derived(candidates[candidateIndex] ?? null);
  let hasNextCandidate = $derived(candidateIndex + 1 < candidates.length);
  let currentFaceId = $derived(currentFace?.id ?? null);
  let hasCurrentReview = $derived(!!currentAsset && !!currentUntaggedPerson);

  let currentAssetUrl = $derived(
    currentAsset
      ? getAssetMediaUrl({
          id: currentAsset.id,
          size: AssetMediaSize.Preview,
          cacheKey: currentAsset.thumbhash,
        })
      : '',
  );

  let contentMetrics = $derived.by(() => {
    if (!imgEl || !imageReady) return null;
    return getContentMetrics(imgEl);
  });

  let faceBox = $derived.by(() => {
    const face = currentFace;
    if (!face || !contentMetrics) return null;

    return mapNormalizedRectToContent(
      { x: face.boundingBoxX1 / face.imageWidth, y: face.boundingBoxY1 / face.imageHeight },
      { x: face.boundingBoxX2 / face.imageWidth, y: face.boundingBoxY2 / face.imageHeight },
      contentMetrics,
    );
  });

  function onImageLoad() {
    imageReady = true;
  }

  function handleMouseMove(e: MouseEvent) {
    if (!contentMetrics || !imgEl?.parentElement) return;
    const rect = imgEl.parentElement.getBoundingClientRect();
    mouse = {
      x: e.clientX - rect.left - contentMetrics.offsetX,
      y: e.clientY - rect.top - contentMetrics.offsetY,
    };
  }

  const withConfirmation = async (callback: () => Promise<void>, prompt?: string, confirmText?: string) => {
    if (prompt && confirmText) {
      const isConfirmed = await modalManager.showDialog({ prompt, confirmText });
      if (!isConfirmed) {
        return;
      }
    }

    try {
      return await callback();
    } catch (error) {
      handleError(error, $t('errors.unable_to_resolve_duplicate'));
    }
  };

  async function fetchRandomAssets(): Promise<AssetResponseDto[]> {
    try {
      return await searchRandom({ randomSearchDto: { withPeople: true, size: 50 } });
    } catch (error) {
      handleError(error, 'Failed to fetch random assets');
      return [];
    }
  }

  function findUntaggedPerson(asset: AssetResponseDto): PersonWithFacesResponseDto | null {
    return (asset.people ?? []).find((p) => !p.name && !p.isHidden) ?? null;
  }

  async function findNextCandidate() {
    loading = true;
    empty = false;
    imageReady = false;
    error = null;

    while (queue.length > 0) {
      const asset = queue.shift()!;
      const untagged = findUntaggedPerson(asset);
      if (!untagged) {
        continue;
      }

      currentAsset = asset;
      currentUntaggedPerson = untagged;
      currentFace = null;
      candidateIndex = 0;
      candidates = [];
      searchName = '';
      searchedPeople = [];

      try {
        const faces = await getFaces({ id: asset.id });
        currentFace = faces.find((f) => f.person?.id === untagged.id) ?? null;
      } catch (error) {
        handleError(error);
      }

      if (currentFace) {
        try {
          candidates = await getFaceCandidates({ id: currentFace.id, size: 20 });
        } catch (err) {
          handleError(err);
        }
      }

      if (!currentFace) {
        continue;
      }

      loading = false;
      return;
    }

    try {
      const fresh = await fetchRandomAssets();
      const unseen = fresh.filter((a) => !reviewedIds.has(a.id));
      const hasUntagged = unseen.filter(findUntaggedPerson);

      if (hasUntagged.length === 0) {
        empty = true;
        loading = false;
        return;
      }

      queue = unseen;
      await findNextCandidate();
    } catch (err) {
      error = 'Failed to load more assets';
      loading = false;
      handleError(err);
    }
  }

  async function advance() {
    if (currentAsset) {
      reviewedIds.add(currentAsset.id);
    }
    totalReviewed++;
    await findNextCandidate();
  }

  async function mergeWith(personId: string) {
    if (!currentUntaggedPerson) return;
    isSubmitting = true;
    try {
      await mergePerson({
        id: personId,
        mergePersonDto: { ids: [currentUntaggedPerson.id] },
      });
      toastManager.primary($t('face_merged'));
      await advance();
    } catch (error) {
      handleError(error);
    } finally {
      isSubmitting = false;
    }
  }

  async function hidePerson() {
    if (!currentUntaggedPerson) return;

    return withConfirmation(
      async () => {
        isSubmitting = true;
        try {
          await updatePeople({
            peopleUpdateDto: { people: [{ id: currentUntaggedPerson!.id, isHidden: true }] },
          });
          toastManager.primary($t('face_hidden'));
          await advance();
        } finally {
          isSubmitting = false;
        }
      },
      $t('hide_person_confirmation'),
      $t('hide'),
    );
  }

  async function reassignFace(personId: string) {
    if (!currentFaceId) return;
    isSubmitting = true;
    try {
      await reassignFacesById({ id: personId, faceDto: { id: currentFaceId } });
      toastManager.primary($t('face_reassigned'));
      await advance();
    } catch (error) {
      handleError(error);
    } finally {
      isSubmitting = false;
    }
  }

  async function createAndAssignPerson(name: string) {
    if (!currentFaceId || !name.trim()) return;
    isSubmitting = true;
    try {
      const person = await createPerson({ personCreateDto: { name: name.trim() } });
      await reassignFacesById({ id: person.id, faceDto: { id: currentFaceId } });
      toastManager.primary($t('person_created'));
      await advance();
    } catch (error) {
      handleError(error);
    } finally {
      isSubmitting = false;
    }
  }

  async function handleNo() {
    if (hasNextCandidate) {
      candidateIndex++;
    } else {
      await advance();
    }
  }

  onMount(() => {
    findNextCandidate();
  });
</script>

<svelte:document
  use:shortcuts={[
    { shortcut: { key: 'y' }, onShortcut: () => currentCandidate && mergeWith(currentCandidate.person.id) },
    { shortcut: { key: 'n' }, onShortcut: handleNo },
    { shortcut: { key: 'h' }, onShortcut: hidePerson },
    { shortcut: { key: 's' }, onShortcut: advance },
  ]}
/>

<UserPageLayout title={data.meta.title}>
  {#snippet buttons()}
    <HStack gap={0}>
      <IconButton
        shape="round"
        variant="ghost"
        color="secondary"
        icon={mdiKeyboard}
        title={$t('show_keyboard_shortcuts')}
        onclick={() => modalManager.show(ShortcutsModal, { shortcuts: faceReviewShortcuts })}
        aria-label={$t('show_keyboard_shortcuts')}
      />
    </HStack>
  {/snippet}

  <div class="mx-auto max-w-3xl px-4 py-6 space-y-4">
    <Text size="small" color="muted" class="mb-2">
      <p>{$t('unassigned_faces_description')}</p>
    </Text>

    {#if loading}
      <div class="flex items-center justify-center p-12">
        <LoadingSpinner />
      </div>
    {:else if error}
      <div class="flex flex-col items-center justify-center p-12 text-red-500 dark:text-red-400">
        <p class="text-lg">{error}</p>
        <Button class="mt-4" onclick={findNextCandidate}>{$t('try_again')}</Button>
      </div>
    {:else if empty}
      <div class="flex flex-col items-center justify-center p-12 text-gray-500 dark:text-gray-400">
        <p class="text-lg">{$t('no_unassigned_faces_found')}</p>
        <p class="mt-2 text-sm">{$t('unassigned_faces_empty_description')}</p>
        <Button class="mt-4" onclick={findNextCandidate}>{$t('try_again')}</Button>
      </div>
    {:else if hasCurrentReview}
      {#key currentUntaggedPerson?.id}
        <div
          class="rounded-xl border border-gray-200 dark:border-gray-700 bg-gray-100 dark:bg-gray-900 flex items-center justify-center"
        >
          <div
            class="relative w-full"
            style="height: min(70vh, 600px);"
            onmouseenter={() => (hover = true)}
            onmouseleave={() => (hover = false)}
            onmousemove={handleMouseMove}
          >
            <img
              bind:this={imgEl}
              onload={onImageLoad}
              src={currentAssetUrl}
              alt={currentAsset!.originalFileName ?? ''}
              class="size-full object-contain rounded-lg"
              draggable="false"
            />

            <FaceHighlightOverlay faceBox={faceBox} />

            {#if contentMetrics}
              <MagnifierLens
                imageUrl={currentAssetUrl}
                mouseX={mouse.x}
                mouseY={mouse.y}
                contentWidth={contentMetrics.contentWidth}
                contentHeight={contentMetrics.contentHeight}
                offsetX={contentMetrics.offsetX}
                offsetY={contentMetrics.offsetY}
                visible={hover}
              />
            {/if}
          </div>
        </div>

        <div class="rounded-xl border border-gray-200 dark:border-gray-700 bg-white dark:bg-gray-900 p-5 space-y-4">
          {#if currentCandidate}
            <Text size="small" class="font-medium">
              {$t('is_this_person', { values: { name: currentCandidate.person.name } })}
            </Text>
            {#if candidates.length > 1}
              <Text size="tiny" color="muted">
                {$t('candidate_x_of_y', { values: { index: candidateIndex + 1, total: candidates.length } })}
              </Text>
            {/if}

            <FaceCandidateRow candidate={currentCandidate} />
          {:else}
            <Text size="small" color="muted">{$t('no_named_candidates')}</Text>
          {/if}

          <div class="flex flex-wrap gap-2">
            {#if currentCandidate}
              <Button
                color="success"
                leadingIcon={mdiCheck}
                onclick={() => mergeWith(currentCandidate.person.id)}
                disabled={isSubmitting}
              >
                {$t('yes')}
              </Button>

              {#if hasNextCandidate}
                <Button color="secondary" leadingIcon={mdiClose} onclick={() => candidateIndex++} disabled={isSubmitting}>
                  {$t('try_next_candidate')}
                </Button>
              {:else}
                <Button color="danger" leadingIcon={mdiClose} onclick={advance} disabled={isSubmitting}>
                  {$t('no')}
                </Button>
              {/if}
            {/if}

            <Button color="danger" leadingIcon={mdiEyeOffOutline} onclick={hidePerson} disabled={isSubmitting}>
              {$t('hide')}
            </Button>

            <Button variant="outline" color="secondary" leadingIcon={mdiSkipNext} onclick={advance} disabled={isSubmitting}>
              {$t('skip')}
            </Button>
          </div>

          <div class="border-t border-gray-100 dark:border-gray-800 pt-4">
            <label class="text-sm text-gray-600 dark:text-gray-400">
              {$t('search_for_person')}
            </label>
            <div class="mt-2">
              <PeopleSearch
                type="input"
                bind:searchName
                bind:searchedPeopleLocal={searchedPeople}
                bind:showLoadingSpinner={isSearching}
                inputClass="w-full rounded-lg border border-gray-300 dark:border-gray-600 bg-white dark:bg-gray-800 px-3 py-2 text-sm text-gray-900 dark:text-gray-100 placeholder-gray-400 focus:outline-none focus:ring-2 focus:ring-blue-500"
                placeholder={$t('type_to_search')}
              />
            </div>
            {#if isSearching}
              <div class="mt-2">
                <LoadingSpinner />
              </div>
            {:else if searchName.trim()}
              {#if searchedPeople.length > 0}
                <div class="mt-2 grid grid-cols-3 sm:grid-cols-4 md:grid-cols-6 gap-2">
                  {#each searchedPeople as person (person.id)}
                    <button
                      onclick={() => reassignFace(person.id)}
                      disabled={isSubmitting}
                      class="flex flex-col items-center gap-1 rounded-lg border border-gray-100 dark:border-gray-800 p-2 hover:bg-gray-50 dark:hover:bg-gray-800 transition-colors disabled:opacity-50"
                    >
                      <ImageThumbnail
                        url={getPeopleThumbnailUrl(person)}
                        altText={person.name}
                        widthStyle="56px"
                        heightStyle="56px"
                        circle
                      />
                      <span class="text-xs text-gray-700 dark:text-gray-300 truncate w-full text-center"
                        >{person.name}</span
                      >
                    </button>
                  {/each}
                </div>
              {/if}

              <Button
                variant="outline"
                color="primary"
                leadingIcon={mdiPlus}
                onclick={() => createAndAssignPerson(searchName)}
                disabled={isSubmitting}
                class="mt-2 w-full"
              >
                {$t('create_new_person_named', { values: { name: searchName.trim() } })}
              </Button>
            {/if}
          </div>
        </div>

        <Text size="tiny" color="muted" class="text-center">
          {$t('unassigned_faces_reviewed', { values: { count: totalReviewed.toLocaleString($locale) } })}
        </Text>
      {/key}
    {/if}
  </div>
</UserPageLayout>