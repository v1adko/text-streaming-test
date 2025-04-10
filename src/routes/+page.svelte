<script lang="ts">
  import { onMount } from "svelte";

  let content = $state("");
  let startTime = $state<number>();
  let endTime = $state<number>();
  let duration = $state<number>();
  let chunksReceived = $state(0);

  async function testStream() {
    content = "";
    chunksReceived = 0;
    startTime = Date.now();

    try {
      const response = await fetch("/api/chat-test", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({ test: true }),
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const reader = response.body?.getReader();
      if (!reader) {
        throw new Error("No reader available");
      }

      while (true) {
        const { done, value } = await reader.read();

        if (done) {
          endTime = Date.now();
          duration = endTime - startTime;
          break;
        }

        // Convert the chunk to text and append it
        const chunk = new TextDecoder().decode(value);
        content += chunk;
        chunksReceived++;

        // Log each chunk for debugging
        console.log(`Received chunk ${chunksReceived}:`, chunk.length, "bytes");
      }
    } catch (error: unknown) {
      console.error("Error:", error);
      content = `Error: ${error instanceof Error ? error.message : String(error)}`;
    }
  }
</script>

<div class="p-4">
  <h1 class="text-2xl font-bold mb-4">Streaming Test</h1>

  <button
    on:click={testStream}
    class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded mb-4"
  >
    Test Stream
  </button>

  {#if duration}
    <div class="mb-4">
      <p>Duration: {duration}ms</p>
      <p>Chunks received: {chunksReceived}</p>
      <p>Total content length: {content.length} characters</p>
    </div>
  {/if}

  <div class="border p-4 max-h-96 overflow-auto">
    {content}
  </div>
</div>
