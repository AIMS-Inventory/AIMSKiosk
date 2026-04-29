<script lang="ts">
  export let boxInfo: Record<string, { id: string, contents: { placed_by: string, pills: string[] } }> = {};
  export let knownBoxes: { name: string, code: number }[] = [];
  export let sendEvent: (eventName: string, payload: any) => void = () => {};

  let search = '';

  function getBoxName(id: string) {
    const known = knownBoxes.find(kb => kb.code.toString() === id || kb.name === id);
    return known ? known.name : `Box #${id}`;
  }

  $: query = search.trim().toLowerCase();

  $: filteredEntries = Object.entries(boxInfo).filter(([id, box]) => {
    if (!query) return true;
    return box.contents.pills.some(p => p.toLowerCase().includes(query));
  });

  function removeBox(boxId: string) {
    sendEvent('remove_box', { box_id: boxId });
  }
</script>

<div class="current-boxes">
  <div class="toolbar">
    <h2>Current Boxes</h2>
    <input
      class="search"
      type="text"
      placeholder="Search by pill name..."
      bind:value={search}
    />
  </div>

  {#if Object.keys(boxInfo).length === 0}
    <p class="empty">No boxes currently tracked.</p>
  {:else if filteredEntries.length === 0}
    <p class="empty">No boxes contain "{search}".</p>
  {:else}
    <ul class="box-list">
      {#each filteredEntries as [id, box]}
        <li class="box-row">
          <div class="box-main">
            <span class="box-name">{getBoxName(id)}</span>
            <span class="box-meta">ID: {id} &mdash; Placed by: {box.contents.placed_by || 'Unknown'}</span>
          </div>
          <div class="pill-tags">
            {#if box.contents.pills.length === 0}
              <span class="pill-tag empty-tag">No pills</span>
            {:else}
              {#each box.contents.pills as pill}
                <span class="pill-tag" class:highlight={query && pill.toLowerCase().includes(query)}>
                  {pill}
                </span>
              {/each}
            {/if}
          </div>
          <button class="delete-btn" on:click={() => removeBox(id)} title="Remove box">✕</button>
        </li>
      {/each}
    </ul>
  {/if}
</div>

<style>
  .current-boxes { padding: 20px; }

  .toolbar {
    display: flex;
    align-items: center;
    gap: 20px;
    margin-bottom: 16px;
    flex-wrap: wrap;
  }
  .toolbar h2 { margin: 0; }

  .search {
    flex: 1;
    min-width: 200px;
    max-width: 400px;
    padding: 10px 14px;
    font-size: 1em;
    border: 2px solid #ccc;
    border-radius: 6px;
    outline: none;
    font-family: inherit;
  }
  .search:focus { border-color: #007AFF; }

  .empty { color: #777; font-style: italic; }

  .box-list {
    list-style: none;
    margin: 0;
    padding: 0;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  .box-row {
    background: #fff;
    border: 1px solid #ddd;
    border-radius: 8px;
    padding: 14px 18px;
    display: flex;
    align-items: center;
    gap: 20px;
    flex-wrap: wrap;
  }

  .box-main {
    display: flex;
    flex-direction: column;
    min-width: 180px;
  }
  .box-name { font-size: 1.1em; font-weight: bold; color: #222; }
  .box-meta { font-size: 0.85em; color: #777; margin-top: 2px; }

  .pill-tags {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    flex: 1;
  }

  .pill-tag {
    background: #e3f2fd;
    color: #1565c0;
    border-radius: 12px;
    padding: 4px 12px;
    font-size: 0.9em;
    white-space: nowrap;
  }
  .pill-tag.highlight {
    background: #fff176;
    color: #333;
    font-weight: bold;
  }
  .pill-tag.empty-tag {
    background: #f5f5f5;
    color: #aaa;
  }

  .delete-btn {
    background: #f44336;
    color: white;
    border: none;
    border-radius: 4px;
    padding: 6px 10px;
    font-size: 1.2em;
    cursor: pointer;
    white-space: nowrap;
  }
  .delete-btn:hover {
    background: #d32f2f;
  }
</style>
