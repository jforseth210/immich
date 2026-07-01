<script lang="ts">
  interface Props {
    imageUrl: string;
    mouseX: number;
    mouseY: number;
    contentWidth: number;
    contentHeight: number;
    offsetX: number;
    offsetY: number;
    visible: boolean;
    zoom?: number;
    lensSize?: number;
  }

  let {
    imageUrl,
    mouseX,
    mouseY,
    contentWidth,
    contentHeight,
    offsetX,
    offsetY,
    visible,
    zoom = 3,
    lensSize = 175,
  }: Props = $props();

  let clampedX = $derived(Math.max(0, Math.min(contentWidth, mouseX)));
  let clampedY = $derived(Math.max(0, Math.min(contentHeight, mouseY)));
</script>

{#if visible && contentWidth > 0}
  <div
    class="pointer-events-none absolute z-20"
    style="
      width: {lensSize}px;
      height: {lensSize}px;
      border-radius: 50%;
      border: 2px solid rgba(255,255,255,0.85);
      box-shadow: 0 2px 10px rgba(0,0,0,0.25);
      overflow: hidden;
      left: {clampedX + offsetX - lensSize / 2}px;
      top: {clampedY + offsetY - lensSize / 2}px;
    "
  >
    <div
      class="size-full"
      style="
        background-image: url({imageUrl});
        background-size: {contentWidth * zoom}px {contentHeight * zoom}px;
        background-position: {-clampedX * zoom + lensSize / 2}px {-clampedY * zoom + lensSize / 2}px;
        background-repeat: no-repeat;
      "
    />
  </div>
{/if}
