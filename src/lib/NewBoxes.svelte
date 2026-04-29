<script lang="ts">
  export let unknownBoxIds: any[] = [];
  export let recognizedFaces: Record<string, any> = {};
  export let shelfInfo: Record<string, any> = {};
  export let knownBoxes: any[] = [];
  export let sendEvent: (eventName: string, payload: any) => void;
  export let sendRequest: (eventName: string, payload?: any) => Promise<any>;

  let activeWizardBox: string | null = null;
  let wizardStep: 'identity' | 'pills' | 'placement' = 'identity';
  let selectedPerson: any = null;
  let assignedShelf: string | null = null;
  let capturedPills: any[] | null = null;
  let capturing = false;
  let captureError: string | null = null;

  function startRegistration(boxId: string) {
    activeWizardBox = boxId;
    wizardStep = 'identity';
    selectedPerson = null;
    assignedShelf = null;
    capturedPills = null;
    captureError = null;
  }

  function selectPerson(person: any) {
    selectedPerson = person;
    wizardStep = 'pills';
  }

  async function capturePills() {
    capturing = true;
    captureError = null;
    try {
      const response = await sendRequest('get_on_screen_pills');
      capturedPills = response.pills ?? [];
    } catch (e: any) {
      captureError = e.message ?? 'Failed to capture pills';
    } finally {
      capturing = false;
    }
  }

  function retakePills() {
    capturedPills = null;
    captureError = null;
  }

  function confirmPills() {
    const shelfKeys = Object.keys(shelfInfo);
    assignedShelf = shelfKeys.length > 0
      ? shelfKeys[Math.floor(Math.random() * shelfKeys.length)]
      : "MockShelf1";
    wizardStep = 'placement';
  }

  function finishRegistration() {
    const shelf = shelfInfo[assignedShelf!];
    const takenPositions = Object.keys(shelf.boxes).map(Number);
    const position = takenPositions.length > 0 ? Math.max(...takenPositions) + 1 : 0;

    sendEvent('register_box', {
      box_id: activeWizardBox,
      placed_by: selectedPerson.name,
      shelf_code: shelf.code,
      position,
      pills: capturedPills ?? []
    });
    activeWizardBox = null;
  }

  function isKnown(id: string) {
    return knownBoxes.some(kb => kb.code.toString() === id || kb.name === id);
  }
</script>

<div class="new-boxes">
  {#if activeWizardBox}
    <div class="wizard">
      <h2>Registering Box: {!isKnown(activeWizardBox) ? `Unknown ID: ${activeWizardBox} (Please add to DB)` : activeWizardBox}</h2>
      
      {#if wizardStep === 'identity'}
        <h3>Step 1: Who are you?</h3>
        {#if Object.keys(recognizedFaces).length === 0}
          <p>No faces recognized. Please step into the camera view.</p>
          <button on:click={() => selectPerson({id: 'mock_user', name: 'Mock User'})}>Use Mock User</button>
        {:else}
          <div class="face-grid">
            {#each Object.values(recognizedFaces) as person}
              <button class="face-btn" on:click={() => selectPerson(person)}>
                {person.name} (ID: {person.id})
              </button>
            {/each}
          </div>
        {/if}
        <button class="cancel-btn" on:click={() => activeWizardBox = null}>Cancel</button>

      {:else if wizardStep === 'pills'}
        <h3>Step 2: Pill Recognition</h3>
        <div class="pill-container">
          <br/>
          Displaying items...
        </div>
        {#if capturedPills === null}
          {#if captureError}
            <p class="error">{captureError}</p>
          {/if}
          <div class="actions">
            <button class="primary" on:click={capturePills} disabled={capturing}>
              {capturing ? 'Capturing...' : 'Capture'}
            </button>
          </div>
        {:else}
          <div class="capture-result">
            {#if capturedPills.length === 0}
              <p>No pills detected.</p>
            {:else}
              {#each capturedPills as pill}
                <div class="pill-row">
                  <strong>{pill.name}</strong> &times;{pill.quantity}
                  <span class="confidence">({Math.round(pill.confidence * 100)}%)</span>
                </div>
              {/each}
            {/if}
          </div>
          <div class="actions">
            <button on:click={retakePills}>Retake</button>
            <button class="primary" on:click={confirmPills}>Continue</button>
          </div>
        {/if}

      {:else if wizardStep === 'placement'}
        <h3>Step 3: Placement</h3>
        <p>Please place the box on: <strong>Shelf {shelfInfo[assignedShelf!]?.name || assignedShelf}</strong></p>
        <div class="actions">
          <button class="primary" on:click={finishRegistration}>I placed it</button>
        </div>
      {/if}
    </div>
  {:else}
    <h2>Unregistered / New Boxes</h2>
    {#if unknownBoxIds.length === 0}
      <p>No new boxes detected.</p>
    {:else}
      <div class="grid">
        {#each [...new Set(unknownBoxIds)] as boxId}
          <div class="card">
            <h3>Box #{boxId}</h3>
            {#if !isKnown(boxId.toString())}
              <p class="warn">Note: This ID is not in the known boxes database.</p>
            {/if}
            <button class="primary" on:click={() => startRegistration(boxId.toString())}>Register Box</button>
          </div>
        {/each}
      </div>
    {/if}
  {/if}
</div>

<style>
  .new-boxes { padding: 20px; }
  .grid, .face-grid { display: grid; gap: 15px; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); }
  .card { background: #fff3e0; padding: 15px; border-radius: 8px; color: black; }
  .card .warn { color: #d32f2f; font-size: 0.9em; }
  .wizard { background: #f9f9f9; padding: 20px; border-radius: 8px; color: black; }
  .pill-container { background: #ccc; height: 200px; display: flex; align-items: center; justify-content: center; margin: 20px 0; border-radius: 8px; flex-direction: column; }
  button { padding: 10px 15px; cursor: pointer; border: none; border-radius: 4px; margin-right: 10px; }
  button.primary { background: #007AFF; color: white; }
  button.face-btn { background: #2196f3; color: white; padding: 15px; font-size: 1.1em; }
  button.cancel-btn { background: #f44336; color: white; margin-top: 20px; }
  .actions { margin-top: 20px; }
  .capture-result { margin-top: 15px; background: #e8f5e9; padding: 12px 16px; border-radius: 6px; font-size: 1em; }
  .pill-row { padding: 4px 0; }
  .confidence { color: #777; font-size: 0.85em; margin-left: 6px; }
  .error { color: #d32f2f; margin-top: 10px; }
</style>
