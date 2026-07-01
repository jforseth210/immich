<script lang="ts">
  import ImageThumbnail from '$lib/components/assets/thumbnail/ImageThumbnail.svelte';
  import { getPeopleThumbnailUrl } from '$lib/utils';
  import type { FaceCandidateDto } from '@immich/sdk';
  import SimilarityBadge from './SimilarityBadge.svelte';

  interface Props {
    candidate: FaceCandidateDto;
  }

  let { candidate }: Props = $props();
</script>

<div class="flex items-center gap-3 rounded-xl border border-gray-100 dark:border-gray-800 bg-gray-50 dark:bg-gray-800 px-3 py-2">
  <ImageThumbnail
    url={getPeopleThumbnailUrl(candidate.person)}
    altText={candidate.person.name}
    widthStyle="48px"
    heightStyle="48px"
    circle
  />
  <div class="flex-1 min-w-0 flex items-center gap-2">
    <p class="text-sm font-medium text-gray-800 dark:text-gray-100 truncate">
      {candidate.person.name}
    </p>
    {#if candidate.similarity != null}
      <SimilarityBadge similarity={candidate.similarity} />
    {/if}
  </div>
</div>
