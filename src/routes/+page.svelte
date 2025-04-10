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
  let abortTimeout = $state(0); // Default timeout in ms
  let contentContainer: HTMLDivElement;
  let reader: ReadableStreamDefaultReader<Uint8Array> | null = null;
  let isStreaming = $state(false);
  let abortController: AbortController | null = null;
  let abortReason = $state<string | null>(null);

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
    if (abortController) {
      abortController.abort();
      abortController = null;
    }
    if (reader) {
      try {
        await reader.cancel();
      } catch (error) {
        // Ignore AbortError as it's expected when the stream is already aborted
        if (!(error instanceof Error && error.name === "AbortError")) {
          console.error("Error canceling reader:", error);
        }
      }
      reader = null;
    }
    abortReason = "Stream aborted by user";
    isStreaming = false;
    clearInterval(timerInterval);
    if (startTime) {
      endTime = Date.now();
      duration = endTime - startTime;
    }
  }

  async function startStream() {
    content = "";
    chunksReceived = 0;
    startTime = Date.now();
    elapsedSeconds = 0;
    isStreaming = true;
    abortReason = null;

    // Create new AbortController for this request
    abortController = new AbortController();

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
          abortTimeout: Math.max(0, Math.floor(abortTimeout)), // Ensure minimum 0ms
        }),
        signal: abortController.signal,
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

          // If we have an abortTimeout and didn't receive all chunks, assume server aborted
          if (abortTimeout > 0 && chunksReceived < expectedChunks) {
            abortReason = `Stream aborted due to a server timeout after ${abortTimeout}ms`;
          }
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
      if (error instanceof Error && error.name === "AbortError") {
        console.log("Stream aborted by user");
      } else {
        console.error("Error:", error);
        // If we have an abortTimeout set and it wasn't a user abort, assume it was the server
        abortReason = `Error: ${error instanceof Error ? error.message : String(error)}`;
      }
      clearInterval(timerInterval);
      isStreaming = false;
    } finally {
      abortController = null;
    }
  }
</script>

<div class="p-4">
  <h1 class="text-2xl font-bold mb-4">Text Streaming Test</h1>

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

  <div class="mb-4">
    <label
      for="abortTimeout"
      class="block text-sm font-medium text-gray-700 mb-2"
    >
      Abort Timeout (ms)
    </label>
    <input
      type="number"
      id="abortTimeout"
      bind:value={abortTimeout}
      min="0"
      step="100"
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

  {#if abortReason}
    <p class="text-red-500">{abortReason}</p>
  {/if}
</div>
