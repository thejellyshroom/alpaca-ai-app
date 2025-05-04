<script lang="ts">
  import { invoke } from "@tauri-apps/api/core";
  import AlpacaInterface from "../lib/components/AlpacaInterface.svelte";

  let name = $state("");
  let greetMsg = $state("");

  async function greet(event: Event) {
    event.preventDefault();
    // Learn more about Tauri commands at https://tauri.app/develop/calling-rust/
    greetMsg = await invoke("greet", { name });
  }
</script>

<main class="container">
  <h1>Welcome to Tauri + Svelte</h1>

  <div class="row">
    <a href="https://tauri.app" target="_blank" rel="noreferrer">
      <img src="/tauri.svg" class="logo tauri" alt="Tauri logo" />
    </a>
    <a href="https://svelte.dev" target="_blank" rel="noreferrer">
      <img src="/svelte.svg" class="logo svelte" alt="Svelte logo" />
    </a>
  </div>

  <p>Click on the Tauri and Svelte logos to learn more.</p>

  <form
    class="row"
    onsubmit={(e) => {
      greet(e);
    }}
  >
    <input id="greet-input" bind:value={name} placeholder="Enter a name..." />
    <button type="submit">Greet</button>
  </form>

  <p>{greetMsg}</p>
</main>

<!-- Alpaca Interface below the main content -->
<AlpacaInterface />

<style>
  .logo {
    height: 6em;
    padding: 1.5em;
    will-change: filter;
    transition: 0.75s;
  }

  .logo.tauri:hover {
    filter: drop-shadow(0 0 2em #24c8db);
  }

  .logo.svelte:hover {
    filter: drop-shadow(0 0 2em #ff3e00);
  }

  .row {
    display: flex;
    justify-content: center;
  }

  a:hover {
    filter: drop-shadow(0 0 2em #646cffaa);
  }

  input,
  button {
    border-radius: 8px;
    border: 1px solid transparent;
    padding: 0.6em 1.2em;
    font-size: 1em;
    font-weight: 500;
    font-family: inherit;
    color: #0f0f0f;
    background-color: #ffffff;
    transition: border-color 0.25s;
    box-shadow: 0 2px 2px rgba(0, 0, 0, 0.2);
  }

  button {
    cursor: pointer;
  }

  button:hover {
    border-color: #396cd8;
  }

  button:active {
    border-color: #396cd8;
    background-color: #e8e8e8;
  }

  input,
  button {
    outline: none;
  }

  .container {
    padding: 2rem;
    text-align: center;
  }

  p {
    color: rgba(255, 255, 255, 0.5);
  }
</style>
