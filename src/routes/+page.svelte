<script lang="ts">
  let content = $state("");
  let startTime = $state<number | undefined>();
  let endTime = $state<number | undefined>();
  let duration = $state<number>();
  let chunksReceived = $state(0);
  let elapsedSeconds = $state(0);
  let timerInterval: number;
  let responseLength = $state(1000); // Default length
  let chunkSize = $state(100); // Default chunk size
  let contentContainer: HTMLDivElement;
  let reader: ReadableStreamDefaultReader<Uint8Array> | null = null;
  let isStreaming = $state(false);

  // Calculate expected number of chunks
  const expectedChunks = $derived(Math.ceil(responseLength / chunkSize));
  const progressPercentage = $derived((chunksReceived / expectedChunks) * 100);

  function updateElapsedTime() {
    if (startTime) {
      elapsedSeconds = Math.floor((Date.now() - startTime) / 1000);
    }
  }

  function scrollToBottom() {
    if (contentContainer) {
      contentContainer.scrollTop = contentContainer.scrollHeight;
    }
  }

  async function stopStream() {
    if (reader) {
      await reader.cancel();
      reader = null;
      isStreaming = false;
      clearInterval(timerInterval);
      if (startTime) {
        endTime = Date.now();
        duration = endTime - startTime;
      }
    }
  }

  async function startStream() {
    content = "";
    chunksReceived = 0;
    startTime = Date.now();
    elapsedSeconds = 0;
    isStreaming = true;

    // Start the timer
    timerInterval = setInterval(updateElapsedTime, 1000);

    try {
      const response = await fetch("/api/chat-test", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          test: true,
          length: Math.max(1, Math.floor(responseLength)), // Ensure positive number
          chunkSize: Math.max(1, Math.floor(chunkSize)), // Ensure positive number
        }),
      });

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const newReader = response.body?.getReader();
      if (!newReader) {
        throw new Error("No reader available");
      }
      reader = newReader;

      while (true) {
        const { done, value } = await reader.read();

        if (done) {
          endTime = Date.now();
          duration = endTime - startTime;
          clearInterval(timerInterval);
          isStreaming = false;
          break;
        }

        // Convert the chunk to text and append it
        const chunk = new TextDecoder().decode(value);
        content += chunk;
        chunksReceived++;
        scrollToBottom();

        // Log each chunk for debugging
        console.log(`Received chunk ${chunksReceived}:`, chunk.length, "bytes");
      }
    } catch (error: unknown) {
      console.error("Error:", error);
      content = `Error: ${error instanceof Error ? error.message : String(error)}`;
      clearInterval(timerInterval);
      isStreaming = false;
    }
  }
</script>

<div class="p-4">
  <h1 class="text-2xl font-bold mb-4">Streaming Test</h1>

  <div class="mb-4">
    <label
      for="responseLength"
      class="block text-sm font-medium text-gray-700 mb-2"
    >
      Response Length (characters)
    </label>
    <input
      type="number"
      id="responseLength"
      bind:value={responseLength}
      min="1"
      class="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500"
    />
  </div>

  <div class="mb-4">
    <label for="chunkSize" class="block text-sm font-medium text-gray-700 mb-2">
      Chunk Size (characters)
    </label>
    <input
      type="number"
      id="chunkSize"
      bind:value={chunkSize}
      min="1"
      class="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500"
    />
  </div>

  <div class="flex gap-2 mb-4">
    <button
      onclick={startStream}
      disabled={isStreaming}
      class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded disabled:opacity-50 disabled:cursor-not-allowed"
    >
      Start Stream
    </button>

    <button
      onclick={stopStream}
      disabled={!isStreaming}
      class="bg-red-500 hover:bg-red-700 text-white font-bold py-2 px-4 rounded disabled:opacity-50 disabled:cursor-not-allowed"
    >
      Stop Stream
    </button>
  </div>

  {#if startTime}
    <div class="mb-4">
      <div class="mb-2">
        <div class="flex justify-between text-sm text-gray-600 mb-1">
          <span>Progress: {chunksReceived} / {expectedChunks} chunks</span>
          <span>{progressPercentage.toFixed(1)}%</span>
        </div>
        <div class="w-full bg-gray-200 rounded-full h-2.5">
          <div
            class="bg-blue-600 h-2.5 rounded-full transition-all duration-300"
            style="width: {progressPercentage}%"
          ></div>
        </div>
      </div>

      <p>Elapsed time: {elapsedSeconds} seconds</p>
      {#if duration}
        <p>Total duration: {duration}ms</p>
        <p>Chunks received: {chunksReceived}</p>
        <p>Total content length: {content.length} characters</p>
      {/if}
    </div>
  {/if}

  <div bind:this={contentContainer} class="border p-4 max-h-96 overflow-auto">
    {content}
  </div>
</div>
