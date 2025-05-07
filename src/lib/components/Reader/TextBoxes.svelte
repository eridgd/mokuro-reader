<script lang="ts">
  import { clamp, promptConfirmation } from '$lib/util';
  import type { Page } from '$lib/types';
  import { settings } from '$lib/settings';
  import { imageToWebp, showCropper, updateLastCard } from '$lib/anki-connect';

  export let page: Page;
  export let src: File;

  $: textBoxes = page.blocks
    .map((block) => {
      const { img_height, img_width } = page;
      const { box, font_size, lines, vertical } = block;

      let [_xmin, _ymin, _xmax, _ymax] = box;

      const xmin = clamp(_xmin, 0, img_width);
      const ymin = clamp(_ymin, 0, img_height);
      const xmax = clamp(_xmax, 0, img_width);
      const ymax = clamp(_ymax, 0, img_height);

      const width = xmax - xmin;
      const height = ymax - ymin;
      const area = width * height;

      const textBox = {
        left: `${xmin}px`,
        top: `${ymin}px`,
        width: `${width}px`,
        height: `${height}px`,
        fontSize: $settings.fontSize === 'auto' ? `${font_size}px` : `${$settings.fontSize}pt`,
        writingMode: vertical ? 'vertical-rl' : 'horizontal-tb',
        lines,
        area
      };

      return textBox;
    })
    .sort(({ area: a }, { area: b }) => {
      return b - a;
    });

  $: fontWeight = $settings.boldFont ? 'bold' : '400';
  $: display = $settings.displayOCR ? 'block' : 'none';
  $: border = $settings.textBoxBorders ? '1px solid red' : 'none';
  $: contenteditable = $settings.textEditable;

  $: triggerMethod = $settings.ankiConnectSettings.triggerMethod || 'both';

  let isTouching = false;
  let touchStartTime = 0;
  
  // Custom tap detection
  let lastTapTime = 0;
  let lastTapElement: HTMLElement | null = null;

  function handleTouchStart(event: TouchEvent) {
    isTouching = true;
    touchStartTime = Date.now();
  }

  function handleTouchEnd(event: TouchEvent) {
    if (!isTouching) return;
    
    const touchDuration = Date.now() - touchStartTime;
    isTouching = false;

    // Long press - more than 250ms
    if (touchDuration > 250) {
      const target = event.target as HTMLElement;
      const textBox = target.closest('.textBox') as HTMLElement;
      const selection = window.getSelection();
      if (selection && textBox) {
        selection.selectAllChildren(textBox);
      }
      return;
    }
    
    // Handle tap detection for custom double-tap
    const now = Date.now();
    const target = event.target as HTMLElement;
    const textBox = target.closest('.textBox') as HTMLElement;
    
    if (!textBox) return;
    
    // Check if this is a double tap (two taps within 300ms on same element)
    if (lastTapElement === textBox && (now - lastTapTime) < 300) {
      // This is our custom double-tap, do the selection
      const selection = window.getSelection();
      if (selection) {
        selection.selectAllChildren(textBox);
      }
      
      // Handle Anki if enabled
      const lines = textBox.innerText.split('\n').filter(line => line.trim());
      if (triggerMethod === 'both' || triggerMethod === 'doubleTap') {
        setTimeout(() => onUpdateCard(lines), 300);
      }
      
      // Reset tap tracking
      lastTapTime = 0;
      lastTapElement = null;
    } else {
      // First tap, record it
      lastTapTime = now;
      lastTapElement = textBox;
    }
  }

  async function onUpdateCard(lines: string[]) {
    if ($settings.ankiConnectSettings.enabled) {
      const sentence = lines.join(' ');
      if ($settings.ankiConnectSettings.cropImage) {
        showCropper(URL.createObjectURL(src), sentence);
      } else {
        promptConfirmation('Add image to last created anki card?', async () => {
          const imageData = await imageToWebp(src);
          updateLastCard(imageData, sentence);
        });
      }
    }
  }

  function onContextMenu(event: Event, lines: string[]) {
    if (triggerMethod === 'both' || triggerMethod === 'rightClick') {
      event.preventDefault();
      onUpdateCard(lines);
    }
  }
</script>

{#each textBoxes as { fontSize, height, left, lines, top, width, writingMode }, index (`textBox-${index}`)}
  <div
    class="textBox"
    style:width
    style:height
    style:left
    style:top
    style:font-size={fontSize}
    style:font-weight={fontWeight}
    style:display
    style:border
    style:writing-mode={writingMode}
    role="none"
    on:contextmenu={(e) => onContextMenu(e, lines)}
    on:touchstart={handleTouchStart}
    on:touchend={handleTouchEnd}
    {contenteditable}
  >
    {#each lines as line}
      <p>{line}</p>
    {/each}
  </div>
{/each}

<style>
  .textBox {
    color: black;
    padding: 0;
    position: absolute;
    line-height: 1.1em;
    font-size: 16pt;
    white-space: nowrap;
    border: 1px solid rgba(0, 0, 0, 0);
    z-index: 11;
    -webkit-user-select: text;
    user-select: text;
    touch-action: manipulation;
  }

  .textBox:focus,
  .textBox:hover {
    background: rgb(255, 255, 255);
    border: 1px solid rgba(0, 0, 0, 0);
  }

  .textBox p {
    display: none;
    white-space: nowrap;
    letter-spacing: 0.1em;
    line-height: 1.1em;
    margin: 0;
    background-color: rgb(255, 255, 255);
    font-weight: var(--bold);
    z-index: 11;
    -webkit-user-select: text;
    user-select: text;
  }

  .textBox:focus p,
  .textBox:hover p {
    display: block;
  }

  /* Show text boxes on touch devices when being interacted with */
  @media (hover: none) {
    .textBox:active {
      background: rgb(255, 255, 255);
    }
    
    .textBox:active p {
      display: block;
    }
  }

  .text-content {
    width: 100%;
    height: 100%;
    -webkit-user-select: text;
    user-select: text;
  }
</style>
