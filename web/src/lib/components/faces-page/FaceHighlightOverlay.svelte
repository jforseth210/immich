<script lang="ts">
  import type { Rect } from '$lib/utils/container-utils';

  interface Props {
    faceBox: Rect | null;
  }

  let { faceBox }: Props = $props();

  let maskId = `face-dim-mask-${Math.random().toString(36).slice(2, 9)}`;
</script>

{#if faceBox}
  <div class="pointer-events-none absolute inset-0 z-10">
    <svg class="size-full">
      <defs>
        <mask id={maskId}>
          <rect width="100%" height="100%" fill="white" />
          <rect
            x={faceBox.left}
            y={faceBox.top}
            width={faceBox.width}
            height={faceBox.height}
            fill="black"
            rx="8"
          />
        </mask>
      </defs>
      <rect width="100%" height="100%" fill="rgba(0,0,0,0.2)" mask="url(#{maskId})" />
    </svg>
  </div>
  <div
    class="absolute border-3 border-white rounded-lg pointer-events-none z-20"
    style="left: {faceBox.left}px; top: {faceBox.top}px; width: {faceBox.width}px; height: {faceBox.height}px;"
  />
{/if}
