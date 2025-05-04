<script lang="ts">
  import { onMount, onDestroy } from "svelte";

  // --- State Variables ---
  let ws: WebSocket | null = null;
  let wsUrl: string = "ws://127.0.0.1:8000/ws"; // Default API URL

  // Connection & Interaction State
  let connectionStatus: "Disconnected" | "Connecting" | "Connected" | "Error" =
    "Disconnected";
  let interactionMode: "text" | "voice" = "text"; // Start in text mode
  let voiceState: string = "Idle"; // Tracks backend state during voice interaction
  let isProcessing: boolean = false; // General flag for ongoing interaction (text or voice)

  // UI Content State
  let textInput: string = "";
  let currentTranscript: string = "";
  let assistantResponseText: string = ""; // For displaying text/LLM responses
  let messages: string[] = []; // Keep raw message log for debugging

  // Audio Playback State
  let audioContext: AudioContext | null = null;
  let audioQueue: { buffer: ArrayBuffer; sampleRate: number }[] = [];
  let isPlayingAudio: boolean = false;
  const DEFAULT_SAMPLE_RATE = 16000; // Default if not provided

  // --- Audio Playback ---
  function initializeAudio() {
    if (!audioContext) {
      try {
        audioContext = new (window.AudioContext ||
          (window as any).webkitAudioContext)();
        console.log("AudioContext initialized.");
      } catch (e) {
        console.error("Web Audio API is not supported in this browser.", e);
        // Maybe disable voice output features?
      }
    }
  }

  async function playNextAudioChunk() {
    if (!audioContext || isPlayingAudio || audioQueue.length === 0) {
      return;
    }

    isPlayingAudio = true;
    const audioJob = audioQueue.shift();

    if (audioJob) {
      const { buffer, sampleRate } = audioJob;
      try {
        // --- Manual AudioBuffer Creation and Filling for Raw PCM (s16le) ---
        const numberOfChannels = 1; // Assuming mono TTS
        const bytesPerSample = 2; // For s16le
        const numberOfSamples = buffer.byteLength / bytesPerSample;

        if (numberOfSamples === 0) {
          console.warn("Received empty audio buffer, skipping.");
          isPlayingAudio = false;
          playNextAudioChunk(); // Try next chunk
          return;
        }

        const audioBuffer = audioContext.createBuffer(
          numberOfChannels,
          numberOfSamples,
          sampleRate
        );

        // Get the channel data buffer (Float32Array)
        const channelData = audioBuffer.getChannelData(0);

        // Create a Int16 view on the received ArrayBuffer
        const int16Data = new Int16Array(buffer);

        // Copy and scale the Int16 data to Float32 required by Web Audio
        for (let i = 0; i < numberOfSamples; i++) {
          channelData[i] = int16Data[i] / 32768.0; // Scale s16 range to [-1.0, 1.0]
        }
        // -------------------------------------------------------------------

        const source = audioContext.createBufferSource();
        source.buffer = audioBuffer;
        source.connect(audioContext.destination);
        source.onended = () => {
          console.log("Audio chunk playback finished.");
          isPlayingAudio = false;
          // Check immediately if there's more audio to play
          playNextAudioChunk();
        };
        source.start();
        console.log(
          `Playing audio chunk: ${numberOfSamples} samples at ${sampleRate} Hz.`
        );
      } catch (error) {
        console.error("Error creating/playing manual AudioBuffer:", error);
        isPlayingAudio = false;
        // Try playing the next chunk even if this one failed
        playNextAudioChunk();
      }
    } else {
      isPlayingAudio = false; // Should not happen if queue.length > 0 checked
    }
  }

  function handleAudioChunk(base64Data: string, receivedRate?: number) {
    if (!audioContext) {
      console.warn("AudioContext not available, cannot play audio.");
      return;
    }
    const sampleRate = receivedRate || DEFAULT_SAMPLE_RATE;
    try {
      // Decode Base64
      const binaryString = atob(base64Data);
      const len = binaryString.length;
      const bytes = new Uint8Array(len);
      for (let i = 0; i < len; i++) {
        bytes[i] = binaryString.charCodeAt(i);
      }

      if (bytes.buffer.byteLength === 0) {
        console.warn("Decoded audio chunk is empty, skipping queue.");
        return;
      }

      // Add the ArrayBuffer and its sample rate to the queue
      audioQueue.push({ buffer: bytes.buffer, sampleRate: sampleRate });
      console.log(
        `Audio chunk received and queued (${bytes.buffer.byteLength} bytes, ${sampleRate} Hz). ${audioQueue.length} in queue.`
      );
      // Start playing if not already playing
      if (!isPlayingAudio) {
        playNextAudioChunk();
      }
    } catch (error) {
      console.error("Error decoding Base64 audio data:", error);
    }
  }

  // --- WebSocket Logic ---
  function connect() {
    if (ws && ws.readyState === WebSocket.OPEN) {
      console.log("WebSocket already open.");
      return;
    }

    console.log(`Connecting to ${wsUrl}...`);
    connectionStatus = "Connecting";
    messages = [];
    assistantResponseText = "";
    currentTranscript = "";
    textInput = "";
    audioQueue = [];
    isPlayingAudio = false;
    isProcessing = false;
    voiceState = "Idle";
    interactionMode = "text"; // Reset to text mode on connect

    try {
      ws = new WebSocket(wsUrl);
      initializeAudio(); // Initialize AudioContext on connect attempt

      ws.onopen = () => {
        console.log("WebSocket connected");
        connectionStatus = "Connected";
        voiceState = "Idle"; // Ensure idle state on connect
      };

      ws.onmessage = (event) => {
        console.log("Message from server:", event.data);
        messages = [
          ...messages,
          `[${new Date().toLocaleTimeString()}] ${event.data}`,
        ]; // Log raw message with timestamp

        try {
          const messageData = JSON.parse(event.data);

          switch (messageData.type) {
            case "status":
              voiceState = messageData.state || "Unknown";
              status = `Status: ${voiceState}`; // Keep the old status for basic display compat for now
              console.log("Status Updated:", voiceState);
              if (interactionMode === "voice") {
                isProcessing = ![
                  "Idle",
                  "Error",
                  "Interrupted",
                  "Cancelled",
                  "Disabled",
                ].includes(voiceState);
                if (!isProcessing) {
                  // Voice interaction finished, switch back to text mode
                  interactionMode = "text";
                  currentTranscript = ""; // Clear transcript after voice interaction
                  console.log(
                    "Voice interaction ended. Switching to text mode."
                  );
                  // Optionally clear assistant text response too, or keep it? Keep it for now.
                }
              } else if (interactionMode === "text") {
                isProcessing = !["Idle", "Error"].includes(voiceState);
                if (!isProcessing) {
                  console.log("Text interaction response complete.");
                }
              }
              break;

            case "transcript":
              currentTranscript = messageData.text || "";
              console.log("Transcript Updated:", currentTranscript);
              break;

            case "llm_chunk": // Assuming this type for text responses
              assistantResponseText += messageData.text || "";
              console.log("LLM Chunk Received");
              isProcessing = true; // Mark as processing while receiving chunks
              break;

            case "audio_chunk":
              if (messageData.data) {
                handleAudioChunk(messageData.data, messageData.sample_rate);
              } else {
                console.warn("Received audio_chunk without data field.");
              }
              break;

            case "error":
              voiceState = "Error";
              status = `Error: ${messageData.message}`; // Keep old status variable updated
              console.error("Server Error:", messageData.message);
              isProcessing = false;
              // Decide if we switch back to text mode on error
              interactionMode = "text";
              break;

            default:
              console.warn("Received unknown message type:", messageData.type);
          }
        } catch (error) {
          console.error(
            "Failed to parse message or handle data:",
            event.data,
            error
          );
          messages = [...messages, `[RAW/ERROR] ${event.data}`];
          assistantResponseText += `
[Error processing message: ${event.data}]`;
          isProcessing = false; // Stop processing on parse error
          interactionMode = "text"; // Revert to text mode on error
        }
      };

      ws.onerror = (error) => {
        console.error("WebSocket error:", error);
        connectionStatus = "Error";
        isProcessing = false;
        interactionMode = "text";
        ws = null;
      };

      ws.onclose = (event) => {
        console.log("WebSocket closed:", event.code, event.reason);
        connectionStatus = "Disconnected";
        isProcessing = false;
        interactionMode = "text";
        currentTranscript = "";
        assistantResponseText = "";
        voiceState = "Idle";
        audioQueue = [];
        isPlayingAudio = false;
        ws = null;
      };
    } catch (error) {
      console.error("Failed to create WebSocket:", error);
      connectionStatus = "Error";
      ws = null;
    }
  }

  function disconnect() {
    if (ws) {
      console.log("Closing WebSocket connection...");
      ws.close();
      // State updates handled by onclose handler
    } else {
      console.log("WebSocket is not connected.");
    }
  }

  function sendTextMessage() {
    if (
      ws &&
      ws.readyState === WebSocket.OPEN &&
      textInput.trim() &&
      interactionMode === "text" &&
      !isProcessing
    ) {
      console.log("Sending text message:", textInput);
      assistantResponseText = ""; // Clear previous response
      isProcessing = true;
      voiceState = "Processing"; // Use voiceState temporarily for unified status
      ws.send(JSON.stringify({ action: "send_text", text: textInput.trim() }));
      textInput = ""; // Clear input field after sending
    } else {
      console.warn("Cannot send text message. Conditions not met:", {
        connected: ws?.readyState === WebSocket.OPEN,
        hasText: !!textInput.trim(),
        mode: interactionMode,
        processing: isProcessing,
      });
    }
  }

  function startVoiceInteraction() {
    if (
      ws &&
      ws.readyState === WebSocket.OPEN &&
      interactionMode === "text" &&
      !isProcessing
    ) {
      console.log("Sending start voice interaction command...");
      interactionMode = "voice";
      isProcessing = true; // Set processing true immediately
      voiceState = "Initializing"; // Or Starting?
      assistantResponseText = ""; // Clear previous text response
      currentTranscript = ""; // Clear previous transcript
      audioQueue = []; // Clear any residual audio
      isPlayingAudio = false;
      ws.send(JSON.stringify({ action: "start", mode: "voice" }));
    } else {
      console.warn("Cannot start voice interaction. Conditions not met:", {
        connected: ws?.readyState === WebSocket.OPEN,
        mode: interactionMode,
        processing: isProcessing,
      });
    }
  }

  function stopInteraction() {
    // This can stop both voice and potentially long text generations
    if (ws && ws.readyState === WebSocket.OPEN && isProcessing) {
      console.log("Sending stop interaction command...");
      ws.send(JSON.stringify({ action: "stop" }));
      // State changes (like setting isProcessing=false, interactionMode='text')
      // will be triggered by the final status message from the server via onmessage.
      // We might want to optimistically set isProcessing = false here, but let's wait for confirmation.
      voiceState = "Stopping..."; // Optimistic update
    } else {
      console.warn("WebSocket not connected or no interaction active to stop.");
    }
  }

  onMount(() => {
    // Don't connect automatically, let user click button
    // initializeAudio(); // Initialize audio context when component mounts? Or on connect? Let's do on connect.
  });

  onDestroy(() => {
    disconnect();
    if (audioContext && audioContext.state !== "closed") {
      audioContext.close();
    }
  });
</script>

<div class="alpaca-interface">
  <h2>Alpaca Voice Assistant Interface</h2>

  <!-- Connection Controls -->
  <div class="controls connection-controls">
    <label>
      API URL:
      <input
        type="text"
        bind:value={wsUrl}
        disabled={connectionStatus === "Connected" ||
          connectionStatus === "Connecting"}
      />
    </label>
    {#if connectionStatus !== "Connected"}
      <button on:click={connect} disabled={connectionStatus === "Connecting"}
        >Connect</button
      >
    {:else}
      <button on:click={disconnect}>Disconnect</button>
    {/if}
    <span>Status: <strong>{connectionStatus}</strong></span>
  </div>

  {#if connectionStatus === "Connected"}
    <!-- Interaction Controls -->
    <div class="controls interaction-controls">
      {#if interactionMode === "text"}
        <button on:click={startVoiceInteraction} disabled={isProcessing}
          >Start Voice Interaction</button
        >
      {:else}
        <!-- In voice mode -->
        <span>Voice Status: <strong>{voiceState}</strong></span>
      {/if}
      <!-- Stop button is available in both modes if processing -->
      <button on:click={stopInteraction} disabled={!isProcessing}
        >Stop Interaction</button
      >
    </div>

    <!-- Text Interaction Area (only enabled in text mode) -->
    {#if interactionMode === "text"}
      <div class="text-interaction">
        <textarea
          bind:value={textInput}
          placeholder="Enter your message here..."
          rows="3"
          disabled={isProcessing || connectionStatus !== "Connected"}
          on:keydown={(e) => {
            if (e.key === "Enter" && !e.shiftKey) {
              e.preventDefault();
              sendTextMessage();
            }
          }}
        ></textarea>
        <button
          on:click={sendTextMessage}
          disabled={isProcessing || !textInput.trim()}>Send Text</button
        >
      </div>
    {/if}

    <!-- Transcript Display (relevant during voice mode) -->
    {#if interactionMode === "voice" || currentTranscript}
      <div class="transcript-display">
        <h3>Transcript:</h3>
        <p>{currentTranscript || "..."}</p>
      </div>
    {/if}

    <!-- Assistant Response Display -->
    <div class="response-display">
      <h3>Assistant Response:</h3>
      <pre>{assistantResponseText ||
          (isProcessing ? "..." : "(No response yet)")}</pre>
    </div>
  {/if}

  <!-- Raw Message Log (for debugging) -->
  <details>
    <summary>Raw Message Log</summary>
    <pre class="message-log">{messages.join("\n") || "No messages yet."}</pre>
  </details>
</div>

<style>
  .alpaca-interface {
    font-family: sans-serif;
    padding: 1rem;
    border: 1px solid #ccc;
    border-radius: 5px;
    max-width: 800px; /* Increased width */
    margin: 1rem auto;
    display: flex;
    flex-direction: column;
    gap: 1rem; /* Added gap between sections */
  }
  .controls {
    display: flex;
    gap: 0.5rem;
    align-items: center;
    flex-wrap: wrap;
    padding-bottom: 0.5rem; /* Add some spacing below controls */
    border-bottom: 1px solid #eee; /* Separator */
  }
  .connection-controls label {
    flex-grow: 1; /* Allow URL input to grow */
    display: flex;
    align-items: center;
  }
  .connection-controls input {
    flex-grow: 1;
    margin-left: 0.5rem;
  }

  label {
    display: flex;
    align-items: center;
    gap: 0.3rem;
  }
  input[type="text"],
  textarea {
    padding: 0.5rem; /* Increased padding */
    border: 1px solid #ccc;
    border-radius: 4px;
    font-size: 1em;
  }
  textarea {
    width: 100%;
    resize: vertical;
    min-height: 60px;
  }
  button {
    padding: 0.6em 1.2em; /* Adjusted padding */
    cursor: pointer;
    border: 1px solid #aaa;
    border-radius: 4px;
    background-color: #f0f0f0;
  }
  button:hover:not(:disabled) {
    background-color: #e0e0e0;
    border-color: #888;
  }
  button:disabled {
    cursor: not-allowed;
    opacity: 0.6;
  }
  .message-log,
  .response-display pre {
    background-color: #f4f4f4;
    border: 1px solid #eee;
    padding: 0.5rem;
    max-height: 300px;
    overflow-y: auto;
    white-space: pre-wrap; /* Wrap long lines */
    word-wrap: break-word; /* Break long words */
    font-size: 0.9em; /* Slightly smaller font for logs/code */
  }
  .response-display pre {
    min-height: 50px; /* Ensure it has some height even when empty */
  }

  .text-interaction {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    margin-top: 0.5rem;
  }
  .text-interaction button {
    align-self: flex-end; /* Position button to the right */
  }

  .transcript-display,
  .response-display {
    margin-top: 0.5rem;
  }
  h3 {
    margin-bottom: 0.3rem;
    font-size: 1.1em;
  }
  details {
    margin-top: 1rem;
    border-top: 1px solid #eee;
    padding-top: 0.5rem;
  }
  summary {
    cursor: pointer;
    font-weight: bold;
  }
</style>
