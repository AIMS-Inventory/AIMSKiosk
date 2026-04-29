<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { createEventDispatcher } from 'svelte';

  const dispatch = createEventDispatcher();

  let transcript = '';
  let isListening = false;
  let recognition: typeof window.SpeechRecognition | null = null;
  let errorMsg = '';

  onMount(() => {
    // Check for browser support
    const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
    
    if (SpeechRecognition) {
      recognition = new SpeechRecognition();
      recognition.continuous = true;
      recognition.interimResults = true;
      recognition.lang = 'en-US';

      recognition.onstart = () => {
        isListening = true;
        errorMsg = '';
        transcript = '';
        dispatch('start');
      };

      recognition.onresult = (event) => {
        let interimTranscript = '';
        let finalTranscript = '';

        for (let i = event.resultIndex; i < event.results.length; ++i) {
          if (event.results[i].isFinal) {
            finalTranscript += event.results[i][0].transcript;
          } else {
            interimTranscript += event.results[i][0].transcript;
          }
        }

        transcript = finalTranscript + interimTranscript;
        
        // Dispatch result if it's final
        if (finalTranscript) {
          dispatch('command', { command: finalTranscript.trim().toLowerCase() });
        }
      };

      recognition.onerror = (event) => {
        console.error('Speech recognition error', event.error);
        if (event.error !== 'no-speech') {
            errorMsg = event.error;
        }
        isListening = false;
        dispatch('error', { error: event.error });
      };

      recognition.onend = () => {
        isListening = false;
        dispatch('end');
      };
    } else {
      errorMsg = 'Speech Recognition API is not supported in this browser.';
    }
  });

  onDestroy(() => {
    if (recognition && isListening) {
      recognition.stop();
    }
  });

  function toggleListening() {
    if (!recognition) return;
    
    if (isListening) {
      recognition.stop();
    } else {
      transcript = '';
      recognition.start();
    }
  }
</script>

<div class="speech-container">
  <button 
    class="mic-btn {isListening ? 'listening' : ''}" 
    on:click={toggleListening}
    title={isListening ? 'Stop listening' : 'Start speaking'}
    disabled={!recognition}
  >
    {#if isListening}
      <span class="pulse"></span>
    {/if}
    <svg class="icon" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
      <rect x="9" y="2" width="6" height="11" rx="3" fill="currentColor"/>
      <path d="M5 10a7 7 0 0 0 14 0" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
      <line x1="12" y1="19" x2="12" y2="22" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
      <line x1="9" y1="22" x2="15" y2="22" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
    </svg>
  </button>
  
  {#if transcript}
    <div class="transcript-bubble">
      "{transcript}"
    </div>
  {:else if isListening}
    <div class="transcript-bubble placeholder">
      Listening...
    </div>
  {:else if errorMsg}
    <div class="transcript-bubble error">
      {errorMsg}
    </div>
  {/if}
</div>

<style>
  .speech-container {
    display: flex;
    align-items: center;
    gap: 1rem;
    position: relative;
  }

  .mic-btn {
    position: relative;
    background: #f1f5f9;
    border: 2px solid #e2e8f0;
    border-radius: 50%;
    width: 48px;
    height: 48px;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    transition: all 0.2s ease;
    z-index: 2;
  }

  .mic-btn:hover:not(:disabled) {
    background: #e2e8f0;
    transform: scale(1.05);
  }

  .mic-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
    filter: grayscale(1);
  }

  .mic-btn.listening {
    border-color: #ef4444;
    color: white;
  }

  .pulse {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(239, 68, 68, 0.4);
    border-radius: 50%;
    z-index: -1;
    animation: pulser 1.5s infinite;
  }

  .icon {
    position: relative;
    z-index: 1;
    width: 22px;
    height: 22px;
    color: #334155;
  }

  .mic-btn.listening .icon {
    color: #ef4444;
  }

  .transcript-bubble {
    background: white;
    padding: 0.75rem 1.25rem;
    border-radius: 20px;
    box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
    font-size: 0.95rem;
    color: #1e293b;
    max-width: 300px;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    border-left: 4px solid #3b82f6;
    animation: slideIn 0.3s ease-out forwards;
  }

  .transcript-bubble.placeholder {
    color: #64748b;
    font-style: italic;
    border-left-color: #cbd5e1;
  }

  .transcript-bubble.error {
    color: #ef4444;
    border-left-color: #ef4444;
  }

  @keyframes pulser {
    0% { transform: scale(1); opacity: 0.8; }
    100% { transform: scale(1.5); opacity: 0; }
  }

  @keyframes slideIn {
    from { opacity: 0; transform: translateX(-10px); }
    to { opacity: 1; transform: translateX(0); }
  }
</style>