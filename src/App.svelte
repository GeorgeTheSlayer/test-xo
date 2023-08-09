<script lang="ts">
  import {
    createDevice,
    TransportEvent,
    TempoEvent,
    TimeNow,
    type ExternalDataInfo,
    type Device,
    type Parameter,
  } from "@rnbo/js";
  import "./app.css";
  import type { IPatcher, IParameterDescription } from "@rnbo/js";

  let WAContext = window.AudioContext || window.webkitAudioContext;
  let context = new WAContext();
  let isSetup = false;

  // Unlock audio on iOS
  if (context.state === "suspended" && "ontouchstart" in window) {
    // Unlock it
    context.resume();
    let unlock = () => {
      context.resume().then(function () {
        document.body.removeEventListener("touchstart", unlock);
        document.body.removeEventListener("touchend", unlock);
      });
    };

    document.body.addEventListener("touchstart", unlock, false);
    document.body.addEventListener("touchend", unlock, false);
  }

  let device: Device;
  let outputNode: GainNode;
  let params: Parameter[];
  let tempo: number = 60;
  let gain: number = 50;
  let counter = 0;
  let isRunning = false;
  let keyVal = false;
  let xClass = "text-black";
  let oClass = "text-black";

  const setupAudio = async () => {
    let rawPatcher = await fetch("/max/breakerR.export.json");
    let devPatch: IPatcher = await rawPatcher.json();
    device = await createDevice({ context: context, patcher: devPatch });

    let dependencies = await fetch("/max/dependencies.json");
    dependencies = await dependencies.json();
    const results = await device.loadDataBufferDependencies(dependencies);
    results.forEach((result) => {
      if (result.type === "success") {
        console.log(`Successfully loaded buffer with id ${result.id}`);
      } else {
        console.log(
          `Failed to load buffer with id ${result.id}, ${result.error}`
        );
      }
    });

    params = device.parameters;

    // Create gain node and connect it to audio output
    outputNode = context.createGain();
    outputNode.connect(context.destination);

    // This connects the device to audio output, but you may still need to call context.resume()
    // from a user-initiated function.
    device.node.connect(outputNode);

    device.messageEvent.subscribe((ev) => {
      if (ev.tag === "counter") {
        counter = ev.payload;
      }
    });
  };

  let promise = setupAudio();
  promise.then(async () => {
    console.log("Audio setup complete");
    isSetup = true;
  });

  const startAudio = () => {
    if (context.state === "suspended") {
      context.resume();
      let tempoEvent = new TempoEvent(TimeNow, tempo);
      device.scheduleEvent(tempoEvent);
      let transportEvent = new TransportEvent(TimeNow, 1);
      device.scheduleEvent(transportEvent);
      isRunning = true;
    } else {
      context.suspend();
      let transportEvent = new TransportEvent(TimeNow, 0);
      device.scheduleEvent(transportEvent);
      isRunning = false;
    }
  };

  function changeParam(index: number) {
    if (isSetup) {
      if (params[index].value == 2) {
        params[index].value = 0;
      } else {
        params[index].value = params[index].value + 1;
      }
    }
  }

  function setTempo(hTempo: number) {
    let tempoEvent = new TempoEvent(TimeNow, hTempo);
    device.scheduleEvent(tempoEvent);
  }

  function pamToDisp(input: number) {
    if (input === 1) return "X";
    else if (input === 2) return "O";
    else return " ";
  }

  function counterToDisp(input: number, count: number = counter) {
    if (input == params[count].value) {
      return true;
    } else {
      return false;
    }
  }

  function resetParams() {
    params.forEach((param) => {
      param.value = 0;
    });
    keyVal = !keyVal;
  }

  function randomizeParams() {
    params.forEach((param) => {
      param.value = Math.floor(Math.random() * 3);
    });
    keyVal = !keyVal;
  }

  $: {
    if (isSetup) setTempo(tempo);
  }
  $: {
    if (isSetup) outputNode.gain.value = (gain / 100) * 2;
  }

  $: {
    if (isSetup) {
      if (counterToDisp(1, counter)) {
        xClass = "text-red-500";
      } else {
        xClass = "text-black";
      }
      if (counterToDisp(2, counter)) {
        oClass = "text-blue-500";
      } else {
        oClass = "text-black";
      }
    }
  }
</script>

<main class="font-sans text-center select-none">
  <body>
    <div class="sm:landscape:hidden md:landscape:block">
      {#await promise}
        <p>Setting Up Audio</p>
      {:then}
        <h1 class="font-bold text-7xl mt-8">
          <span class={xClass}>X</span>
          <span class={oClass}>0</span>
        </h1>
        <p class="text-xl opacity-50">Press Play and Tap the Pads</p>
        {#key keyVal}
          <div class="square grid grid-cols-4 max-w-lg p-2 mx-auto gap-4 mt-4">
            {#each params as pam, index}
              <button
                class="square {counter == index
                  ? 'bg-black text-white'
                  : 'bg-slate-200 text-black'} font-bold text-4xl lg:text-7xl text-center p-4 hover:opacity-70"
                on:click={() => changeParam(index)}
              >
                {pamToDisp(pam.value)}
              </button>
            {/each}
          </div>
        {/key}
        <div class="flex justify-evenly max-w-xl mx-auto w-full">
          <!-- randomize -->
          <button
            on:click={randomizeParams}
            class="text-white ease-in-out duration-200 hover:opacity-50 bg-black p-4 mt-2 max-w-lg"
            ><svg
              xmlns="http://www.w3.org/2000/svg"
              class="icon icon-tabler icon-tabler-dice-5-filled"
              width="24"
              height="24"
              viewBox="0 0 24 24"
              stroke-width="2"
              stroke="currentColor"
              fill="none"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <path stroke="none" d="M0 0h24v24H0z" fill="none" />
              <path
                d="M18.333 2c1.96 0 3.56 1.537 3.662 3.472l.005 .195v12.666c0 1.96 -1.537 3.56 -3.472 3.662l-.195 .005h-12.666a3.667 3.667 0 0 1 -3.662 -3.472l-.005 -.195v-12.666c0 -1.96 1.537 -3.56 3.472 -3.662l.195 -.005h12.666zm-2.833 12a1.5 1.5 0 1 0 0 3a1.5 1.5 0 0 0 0 -3zm-7 0a1.5 1.5 0 1 0 0 3a1.5 1.5 0 0 0 0 -3zm3.5 -3.5a1.5 1.5 0 1 0 0 3a1.5 1.5 0 0 0 0 -3zm-3.5 -3.5a1.5 1.5 0 1 0 0 3a1.5 1.5 0 0 0 0 -3zm7 0a1.5 1.5 0 1 0 0 3a1.5 1.5 0 0 0 0 -3z"
                stroke-width="0"
                fill="currentColor"
              />
            </svg></button
          >
          <!-- Start -->
          {#if !isRunning}
            <button
              class="text-white ease-in-out duration-200 hover:opacity-50 bg-black p-4 mt-2 max-w-lg"
              on:click={startAudio}
            >
              <svg
                xmlns="http://www.w3.org/2000/svg"
                class="icon icon-tabler icon-tabler-player-play-filled"
                width="24"
                height="24"
                viewBox="0 0 24 24"
                stroke-width="2"
                stroke="currentColor"
                fill="none"
                stroke-linecap="round"
                stroke-linejoin="round"
              >
                <path stroke="none" d="M0 0h24v24H0z" fill="none" />
                <path
                  d="M6 4v16a1 1 0 0 0 1.524 .852l13 -8a1 1 0 0 0 0 -1.704l-13 -8a1 1 0 0 0 -1.524 .852z"
                  stroke-width="0"
                  fill="currentColor"
                />
              </svg>
            </button>
          {:else}
            <button
              class="text-white ease-in-out duration-200 hover:opacity-50 bg-black p-4 mt-2 max-w-lg"
              on:click={startAudio}
            >
              <svg
                xmlns="http://www.w3.org/2000/svg"
                class="icon icon-tabler icon-tabler-player-pause-filled"
                width="24"
                height="24"
                viewBox="0 0 24 24"
                stroke-width="2"
                stroke="currentColor"
                fill="none"
                stroke-linecap="round"
                stroke-linejoin="round"
              >
                <path stroke="none" d="M0 0h24v24H0z" fill="none" />
                <path
                  d="M9 4h-2a2 2 0 0 0 -2 2v12a2 2 0 0 0 2 2h2a2 2 0 0 0 2 -2v-12a2 2 0 0 0 -2 -2z"
                  stroke-width="0"
                  fill="currentColor"
                />
                <path
                  d="M17 4h-2a2 2 0 0 0 -2 2v12a2 2 0 0 0 2 2h2a2 2 0 0 0 2 -2v-12a2 2 0 0 0 -2 -2z"
                  stroke-width="0"
                  fill="currentColor"
                />
              </svg>
            </button>
          {/if}
          <!-- reset -->
          <button
            on:click={resetParams}
            class="text-white ease-in-out duration-200 hover:opacity-50 bg-black p-4 mt-2 max-w-lg"
            ><svg
              xmlns="http://www.w3.org/2000/svg"
              class="icon icon-tabler icon-tabler-clear-all"
              width="24"
              height="24"
              viewBox="0 0 24 24"
              stroke-width="2"
              stroke="currentColor"
              fill="none"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <path stroke="none" d="M0 0h24v24H0z" fill="none" />
              <path d="M8 6h12" />
              <path d="M6 12h12" />
              <path d="M4 18h12" />
            </svg></button
          >
        </div>
        <div
          class="flex mx-auto mt-4 w-full max-w-lg p-4 gap-x-8 justify-evenly"
        >
          <div class="flex flex-col w-full">
            <input
              type="range"
              name="tempo"
              class="slider"
              max="160"
              min="40"
              bind:value={tempo}
            />
            <label class="text-lg" for="tempo">Tempo: {tempo}</label>
          </div>
          <div class="flex flex-col w-full">
            <input
              class="slider"
              type="range"
              name="gainVal"
              max="100"
              min="0"
              bind:value={gain}
            />
            <label class="text-lg" for="gainVal">Gain: {gain}</label>
          </div>
        </div>
      {/await}
    </div>
    <h1
      class="font-bold text-7xl mt-8 h-full w-full p-4 sm:block portrait:hidden md:hidden"
    >
      Please Rotate Device
    </h1>
  </body>
</main>

<style lang="postcss">
  /* The slider itself */
  .slider {
    -webkit-appearance: none; /* Override default CSS styles */
    appearance: none;
    width: 100%; /* Full-width */
    height: 25px; /* Specified height */
    background: #838383; /* Grey background */
    outline: none; /* Remove outline */
    opacity: 0.7; /* Set transparency (for mouse-over effects on hover) */
    -webkit-transition: 0.2s; /* 0.2 seconds transition on hover */
    transition: opacity 0.2s;
  }

  /* Mouse-over effects */
  .slider:hover {
    opacity: 1; /* Fully shown on mouse-over */
  }

  /* The slider handle (use -webkit- (Chrome, Opera, Safari, Edge) and -moz- (Firefox) to override default look) */
  .slider::-webkit-slider-thumb {
    -webkit-appearance: none; /* Override default look */
    appearance: none;
    width: 25px; /* Set a specific slider handle width */
    height: 25px; /* Slider handle height */
    background: black; /* Green background */
    cursor: pointer; /* Cursor on hover */
  }

  .slider::-moz-range-thumb {
    width: 25px; /* Set a specific slider handle width */
    height: 25px; /* Slider handle height */
    background: black; /* Green background */
    cursor: pointer; /* Cursor on hover */
  }
</style>
