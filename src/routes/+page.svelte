<script lang="ts">
  import { 
    Button, 
    Card, 
    Slider, 
    Icon, 
    ListItem, 
    Dialog, 
    TextField, 
    FAB 
  } from "m3-svelte";
  import iconChevronLeft from "@ktibow/iconset-material-symbols/chevron-left";
  import iconChevronRight from "@ktibow/iconset-material-symbols/chevron-right";
  import iconAdd from "@ktibow/iconset-material-symbols/add";
  import iconDelete from "@ktibow/iconset-material-symbols/delete";
  import { fade, fly } from 'svelte/transition';
  import { onMount } from 'svelte';

  // --- Metronome Logic ---
  let audioCtx: AudioContext | null = null;
  let bpm = $state(120);
  let isPlaying = $state(false);
  let nextNoteTime = 0.0;
  const scheduleAheadTime = 0.1;
  const lookahead = 25.0;
  let timerId: number | undefined;

  function updateBPM(newBpm: number) {
    bpm = Math.min(Math.max(newBpm, 40), 240);
  }

  function nextNote() {
    const secondsPerBeat = 60.0 / bpm;
    nextNoteTime += secondsPerBeat;
  }

  function scheduleNote(time: number) {
    if (!audioCtx) return;
    const osc = audioCtx.createOscillator();
    const envelope = audioCtx.createGain();
    osc.frequency.value = 880; 
    osc.connect(envelope);
    envelope.connect(audioCtx.destination);
    envelope.gain.setValueAtTime(0.5, time);
    envelope.gain.exponentialRampToValueAtTime(0.001, time + 0.05);
    osc.start(time);
    osc.stop(time + 0.05);
  }

  function scheduler() {
    if (!audioCtx) return;
    while (nextNoteTime < audioCtx.currentTime + scheduleAheadTime) {
      scheduleNote(nextNoteTime);
      nextNote();
    }
    timerId = window.setTimeout(scheduler, lookahead);
  }

  function togglePlay() {
    if (!audioCtx) {
      audioCtx = new (window.AudioContext || (window as any).webkitAudioContext)();
    }
    if (isPlaying) {
      clearTimeout(timerId);
      isPlaying = false;
      return;
    }
    isPlaying = true;
    nextNoteTime = audioCtx.currentTime + 0.05;
    scheduler();
  }

  let buttonLabel = $derived(isPlaying ? 'Stop' : 'Start');
  
  // --- ItemList Logic ---
  interface Item {
    id: string;
    name: string;
    tempo: number;
    book: string;
    page_number: number;
  }

  let items = $state<Item[]>([]);
  let isLoaded = $state(false);
  
  onMount(() => {
    const storedItems = localStorage.getItem('metronome-items');
    if (storedItems) {
      try {
        items = JSON.parse(storedItems);
      } catch (e) {
        console.error("Failed to parse stored items", e);
      }
    } 
    isLoaded = true;
  });

  $effect(() => {
    if (isLoaded) {
      localStorage.setItem('metronome-items', JSON.stringify(items));
    }
  });

  let isDialogOpen = $state(false);
  let newItemName = $state("");
  let newItemTempo = $state(120);
  let newItemBook = $state("");
  let newItemPageNumber = $state<number | null>(null);

  function addItem() {
    const trimmedName = newItemName.trim();
    if (trimmedName) {
      items = [...items, { 
        id: crypto.randomUUID(), 
        name: trimmedName, 
        tempo: newItemTempo, 
        book: newItemBook || "Unknown", 
        page_number: newItemPageNumber ?? 0
      }];
      resetDialog();
      isDialogOpen = false;
    }
  }

  function resetDialog() {
    newItemName = "";
    newItemTempo = bpm;
    newItemBook = "";
    newItemPageNumber = null;
  }

  function removeItem(id: string) {
    items = items.filter((item) => item.id !== id);
  }

  function openDialog() {
    resetDialog();
    isDialogOpen = true;
  }

  function loadItem(item: Item) {
    bpm = item.tempo;
  }

  let isListEmpty = $derived(items.length === 0);
  let isAddDisabled = $derived(!newItemName.trim());
</script>

{#snippet header()}
  <header in:fade={{ delay: 200 }}>
    <h1>
      <span class="metronome">Metronome</span>
    </h1>
  </header>
{/snippet}

<main>
  {@render header()}
  
  <div class="app-content">
    <!-- Metronome UI -->
    <Card variant="filled">
      <div class="metronome-content">
        <div class="bpm-display">
          <span class="bpm-value">{bpm}</span>
          <span class="bpm-label">BPM</span>
        </div>
        
        <div class="controls">
          <Button 
            variant="text" 
            iconType="full"
            onclick={() => updateBPM(bpm - 1)}
            aria-label="Decrease BPM"
          >
            <Icon icon={iconChevronLeft} />
          </Button>

          <div class="slider-wrapper">
            <Slider 
              min={40} 
              max={240} 
              step={1}
              bind:value={bpm}
            />
          </div>

          <Button 
            variant="text" 
            iconType="full"
            onclick={() => updateBPM(bpm + 1)}
            aria-label="Increase BPM"
          >
            <Icon icon={iconChevronRight} />
          </Button>
        </div>

        <Button 
          variant="elevated"
          onclick={togglePlay}
          size="m"
        >
          {buttonLabel}
        </Button>
      </div>
    </Card>

    <!-- Song List -->
    <Card variant="outlined">
      <div class="list-container">
        {#if isListEmpty}
          <div class="empty-state" in:fade>
            <p>No items yet. Add one!</p>
          </div>
        {:else}
          <div class="items">
            {#each items as item (item.id)}
              <div in:fly={{ y: 20, duration: 300 }} out:fade={{ duration: 200 }}>
                <ListItem 
                  headline={item.name} 
                  overline={`${item.book} • Page ${item.page_number}`}
                  supportingText={`${item.tempo} BPM`}
                  onclick={() => loadItem(item)}
                >
                  {#snippet trailing()}
                    <Button variant="text" iconType="full" onclick={(e) => { e.stopPropagation(); removeItem(item.id); }}>
                      <Icon icon={iconDelete} />
                    </Button>
                  {/snippet}
                </ListItem>
              </div>
            {/each}
          </div>
        {/if}
      </div>
    </Card>

    <div class="fab-wrapper">
      <FAB icon={iconAdd} onclick={openDialog} color="primary-container" text="Add Item" />
    </div>
  </div>
</main>

<Dialog headline="Add New Item" bind:open={isDialogOpen}>
  <div class="dialog-content">
    <TextField label="Name" bind:value={newItemName} />
    <TextField label="Tempo (BPM)" type="number" bind:value={newItemTempo} />
    <TextField label="Book" bind:value={newItemBook} />
    <TextField label="Page Number" type="number" bind:value={newItemPageNumber} enter={addItem} />
  </div>
  {#snippet buttons()}
    <Button variant="text" onclick={() => (isDialogOpen = false)}>Cancel</Button>
    <Button variant="filled" onclick={addItem} disabled={isAddDisabled}>Add</Button>
  {/snippet}
</Dialog>

<style>
  main {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 100dvh;
    padding: 2rem 1rem;
    box-sizing: border-box;
  }

  h1 {
    @apply --m3-display-medium;
    margin-bottom: 3rem;
    text-align: center;
    letter-spacing: -0.05em;
  }

  @media (min-width: 768px) {
    h1 {
      @apply --m3-display-large;
    }
  }

  .metronome {
    color: var(--m3c-on-surface);
    font-weight: 900;
  }

  .app-content {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 2rem;
    width: 100%;
  }

  /* Metronome Styles */
  :global(.m3-card) {
    width: 20rem;
    padding: 2rem;
  }

  .metronome-content {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 2rem;
  }

  .bpm-display {
    display: flex;
    align-items: baseline;
    gap: 0.5rem;
  }

  .bpm-value {
    @apply --m3-display-large;
    font-weight: 900;
    font-variant-numeric: tabular-nums;
  }

  .bpm-label {
    @apply --m3-label-medium;
    color: var(--m3c-on-surface-variant);
    font-weight: 500;
    letter-spacing: 0.1em;
    text-transform: uppercase;
  }

  .controls {
    display: flex;
    align-items: center;
    gap: 1rem;
    width: 100%;
  }

  .slider-wrapper {
    flex: 1;
  }

  /* ItemList Styles */
  .list-container {
    padding: 0.5rem;
    min-height: 100px;
    width: 20rem;
  }

  .empty-state {
    padding: 2rem;
    text-align: center;
    color: var(--m3c-on-surface-variant);
  }

  .items {
    display: flex;
    flex-direction: column;
  }

  .fab-wrapper {
    margin-top: 1.5rem;
    display: flex;
    justify-content: center;
  }

  .dialog-content {
    padding-top: 1rem;
    display: flex;
    flex-direction: column;
    gap: 1rem;
    min-width: 280px;
  }
</style>
