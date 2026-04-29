<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import CurrentBoxes from './lib/CurrentBoxes.svelte';
  import NewBoxes from './lib/NewBoxes.svelte';
  import SpeechRecognition from './lib/SpeechRecognition.svelte';

  let ws: WebSocket | null = null;
  let status = 'DISCONNECTED';
  let wsUrl = 'ws://localhost:8001';
  
  let heartbeatData = {
    unknown_box_ids: [],
    recognized_faces: {},
    crew_info: {},
    shelf_info: {},
    box_info: {},
    known_boxes: []
  };

  let activeTab: 'current' | 'new' = 'current';
  let latestVoiceCommand: string | null = null;

  function connect() {
    try {
      ws = new WebSocket(wsUrl);
    } catch (e: any) {
      console.error(e);
      return;
    }
    ws.onopen = () => {
      status = 'CONNECTED';
      console.debug('[WS] Connected to', wsUrl);
    };
    ws.onmessage = (event) => {
      console.debug('[WS] Raw message received:', event.data);
      try {
        const data = JSON.parse(event.data);
        console.debug('[WS] Parsed message:', data);
        if (data.request_id && pendingRequests.has(data.request_id)) {
          console.debug('[WS] Resolving pending request id:', data.request_id, 'with:', data);
          pendingRequests.get(data.request_id)!.resolve(data);
          pendingRequests.delete(data.request_id);
        } else if (data.unknown_box_ids !== undefined) {
          console.debug('[WS] Heartbeat update:', data);
          heartbeatData = { ...heartbeatData, ...data };
        } else {
          console.warn('[WS] Unhandled message (no matching request_id, no unknown_box_ids):', data);
        }
      } catch (e) {
        console.error('[WS] Failed to parse message. Raw data was:', event.data, 'Error:', e);
      }
    };
    ws.onclose = (event) => {
      console.debug('[WS] Disconnected. Code:', event.code, 'Reason:', event.reason || '(none)', 'Clean:', event.wasClean);
      status = 'DISCONNECTED';
      ws = null;
    };
    ws.onerror = (event) => {
      console.error('[WS] Socket error event:', event);
    };
  }

  function toggleConnection() {
    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.close();
    } else {
      connect();
    }
  }

  function sendEvent(eventName: string, payload: any) {
    if (!ws || ws.readyState !== WebSocket.OPEN) {
      console.error('[WS] sendEvent failed — not connected. Event:', eventName, 'Payload:', payload);
      alert('Not connected to server!');
      return;
    }
    const fullMessage = { event: eventName, ...payload };
    console.debug('[WS] Sending event:', fullMessage);
    ws.send(JSON.stringify(fullMessage));
  }

  type PendingRequest = { resolve: (val: any) => void; reject: (err: any) => void };
  const pendingRequests = new Map<string, PendingRequest>();

  function sendRequest(eventName: string, payload: any = {}): Promise<any> {
    return new Promise((resolve, reject) => {
      if (!ws || ws.readyState !== WebSocket.OPEN) {
        console.error('[WS] sendRequest failed — not connected. Event:', eventName, 'Payload:', payload);
        reject(new Error('Not connected to server'));
        return;
      }
      const request_id = crypto.randomUUID();
      const fullMessage = { event: eventName, request_id, ...payload };
      console.debug('[WS] Sending request:', fullMessage);
      pendingRequests.set(request_id, { resolve, reject });
      ws.send(JSON.stringify(fullMessage));
    });
  }

  onMount(() => {
    connect();
  });

  onDestroy(() => {
    if (ws) ws.close();
  });

  $: unknownCount = heartbeatData.unknown_box_ids?.length || 0;

  function handleVoiceCommand(event: CustomEvent<{ command: string }>) {
    const cmd = event.detail.command;
    latestVoiceCommand = cmd;
    if (cmd.includes('current') || cmd.includes('old boxes') || cmd.includes('known boxes')) {
      activeTab = 'current';
    } else if (cmd.includes('new') || cmd.includes('unknown')) {
      activeTab = 'new';
    } else if (cmd.includes('connect')) {
      if (!ws || ws.readyState !== WebSocket.OPEN) connect();
    } else if (cmd.includes('disconnect')) {
      if (ws) ws.close();
    }
  }
</script>

<main class="kiosk-app">
  <header class="kiosk-header">
    <img src="/aims-logo.svg" alt="AIMS" class="aims-logo" />
    <SpeechRecognition on:command={handleVoiceCommand} />
    <div class="tabs">
      <button 
        class:active={activeTab === 'current'} 
        on:click={() => activeTab = 'current'}>
        Current Boxes
      </button>
      <button 
        class:active={activeTab === 'new'} 
        on:click={() => activeTab = 'new'}>
        New/Unknown Boxes
        {#if unknownCount > 0}
          <span class="badge">{unknownCount}</span>
        {/if}
      </button>
    </div>
    <div class="status-indicator">
      <div class="dot" class:connected={status === 'CONNECTED'}></div>
      {status}
    </div>
  </header>

  <div class="content-area">
    {#if activeTab === 'current'}
      <CurrentBoxes
        boxInfo={heartbeatData.box_info}
        knownBoxes={heartbeatData.known_boxes}
        {sendEvent} />
    {:else}
      <NewBoxes
        unknownBoxIds={heartbeatData.unknown_box_ids}
        recognizedFaces={heartbeatData.recognized_faces}
        shelfInfo={heartbeatData.shelf_info}
        knownBoxes={heartbeatData.known_boxes}
        {sendEvent}
        {sendRequest}
        voiceCommand={latestVoiceCommand} />
    {/if}
  </div>
</main>

<style>
  :global(body) { margin: 0; font-family: sans-serif; background: #e0e0e0; }
  .kiosk-app { display: flex; flex-direction: column; height: 100vh; }
  
  .kiosk-header { 
    display: flex; justify-content: space-between; align-items: center; 
    background: #333; color: white; padding: 15px 30px; 
  }
  .aims-logo { height: 36px; width: auto; object-fit: contain; }

  .tabs button {
    background: none; border: none; color: #ccc; font-size: 1.2em;
    padding: 10px 20px; cursor: pointer; transition: color 0.2s;
    position: relative;
  }
  .tabs button.active { color: white; font-weight: bold; border-bottom: 3px solid #007AFF; }
  .badge {
    background: #e91e63; color: white; border-radius: 50%;
    padding: 2px 8px; font-size: 0.8em; margin-left: 8px;
    vertical-align: top;
  }

  .status-indicator { display: flex; align-items: center; gap: 8px; font-size: 0.9em; }
  .dot { width: 10px; height: 10px; border-radius: 50%; background: #f44336; }
  .dot.connected { background: #007AFF; }

  .content-area { flex: 1; overflow-y: auto; padding: 20px; }
  
  :global(button) { font-family: inherit; }
</style>
