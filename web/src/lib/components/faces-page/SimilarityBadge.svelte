<script lang="ts">
  interface Props {
    similarity: number;
  }

  let { similarity }: Props = $props();

  let pct = $derived(Math.round(similarity * 100));
  let color = $derived(
    pct >= 80
      ? 'bg-green-100 dark:bg-green-900 text-green-700 dark:text-green-300'
      : pct >= 60
        ? 'bg-yellow-100 dark:bg-yellow-900 text-yellow-700 dark:text-yellow-300'
        : 'bg-gray-100 dark:bg-gray-700 text-gray-500 dark:text-gray-400',
  );
</script>

<span class="shrink-0 rounded px-2 py-0.5 text-xs font-medium {color}" title="Cosine similarity">
  {pct}%
</span>
