<script>
  import { onMount, onDestroy } from 'svelte';
  import * as Tone from 'tone';
  import * as dat from 'dat.gui';

  let canvas;
  let ctx;
  let videoElement;
  let isPlaying = false;
  let imageData;
  let cursorPosition = 0;
  let animationId;
  let stream;
  let showModal = false;
  let gui;
  let midiInputs = [];
  let selectedMidiInput = null;
  let inputType = 'image'; // Change default to 'image'
  let mediaLoaded = false; // New flag to check if an image has been loaded

  // Parameters
  let controls = {
    volume: -10,
    octave: 4,
    baseNote: 'C',
    cursorSpeed: 1,
    oscillatorType: 'sine',
    voiceCount: 4,
    midiInput: 'None'
  };

  // Tone.js setup
  let synths = [];

  const noteFrequencies = {
    'C': 261.63, 'C#': 277.18, 'D': 293.66, 'D#': 311.13, 'E': 329.63,
    'F': 349.23, 'F#': 369.99, 'G': 392.00, 'G#': 415.30, 'A': 440.00,
    'A#': 466.16, 'B': 493.88
  };

  onMount(async () => {
    ctx = canvas.getContext('2d');
    
    // Initialize synths
    for (let i = 0; i < controls.voiceCount; i++) {
      const synth = new Tone.Synth().toDestination();
      synths.push(synth);
    }

    // Setup MIDI
    if (navigator.requestMIDIAccess) {
      try {
        const midiAccess = await navigator.requestMIDIAccess();
        midiInputs = Array.from(midiAccess.inputs.values());
        controls.midiInput = 'None';
        setupMIDIListeners(midiAccess);
      } catch (err) {
        console.error("Error accessing MIDI devices:", err);
      }
    }

    // Setup dat.gui
    gui = new dat.GUI();
    gui.add(controls, 'volume', -60, 0).onChange(updateVolume);
    gui.add(controls, 'octave', 1, 8).step(1);
    gui.add(controls, 'baseNote', Object.keys(noteFrequencies));
    gui.add(controls, 'cursorSpeed', 0.1, 10);
    gui.add(controls, 'oscillatorType', ['sine', 'square', 'sawtooth', 'triangle']).onChange(updateOscillatorType);
    gui.add(controls, 'voiceCount', 1, 8).step(1).onChange(updateVoiceCount);
    gui.add(controls, 'midiInput', ['None', ...midiInputs.map(input => input.name)]).onChange(updateMIDIInput);

    // Setup visibility change listener
    document.addEventListener("visibilitychange", handleVisibilityChange);

    // Initialize screen capture as default
    await updateInput();
  });

  onDestroy(() => {
    if (stream) {
      stream.getTracks().forEach(track => track.stop());
    }
    synths.forEach(synth => synth.dispose());
    gui.destroy();
    document.removeEventListener("visibilitychange", handleVisibilityChange);
  });

  function setupMIDIListeners(midiAccess) {
    midiAccess.inputs.forEach((input) => {
      input.onmidimessage = getMIDIMessage;
    });
  }

  function updateMIDIInput(value) {
    selectedMidiInput = midiInputs.find(input => input.name === value);
  }

  function handleVisibilityChange() {
    if (document.hidden) {
      pauseAudio();
    }
  }

  function updateVolume(value) {
    synths.forEach(synth => synth.volume.value = value);
  }

  function updateOscillatorType(value) {
    synths.forEach(synth => synth.oscillator.type = value);
  }

 /* function updateVoiceCount(value) {
    if (isPlaying) {
      pauseAudio();
      isPlaying = false;
    }
    while (synths.length > value) {
      const synth = synths.pop();
      synth.dispose();
    }
    while (synths.length < value) {
      const synth = new Tone.Synth().toDestination();
      synths.push(synth);
    }
  } */


  function updateVoiceCount(value) {
    // Do not pause audio when changing voice count
    while (synths.length > value) {
      const synth = synths.pop();
      synth.dispose();
    }
    while (synths.length < value) {
      const synth = new Tone.Synth().toDestination();
      synth.volume.value = controls.volume;
      synth.oscillator.type = controls.oscillatorType;
      if (isPlaying) {
        synth.triggerAttack(440);
      }
      synths.push(synth);
    }
  }

  function getMIDIMessage(message) {
    let command = message.data[0];
    let note = message.data[1];
    let velocity = (message.data.length > 2) ? message.data[2] : 0;

    switch (command) {
      case 144: // noteOn
        if (velocity > 0) {
          controls.cursorSpeed = note / 64; // Map note to speed
        }
        break;
      case 176: // cc
        switch (note) {
          case 7: // volume
            controls.volume = Tone.gainToDb(velocity / 127);
            synths.forEach(synth => synth.volume.value = controls.volume);
            break;
          case 1: // modulation wheel
            controls.octave = Math.floor(velocity / 16) + 1; // Map to octaves 1-8
            break;
          case 74: // Filter Cutoff
            controls.oscillatorType = ['sine', 'square', 'sawtooth', 'triangle'][Math.floor(velocity / 32)];
            synths.forEach(synth => synth.oscillator.type = controls.oscillatorType);
            break;
        }
        break;
    }
  }

  function togglePlay() {
    if ((inputType === 'image' || inputType === 'video') && !mediaLoaded) {
      showModal = true; // Show modal to select media if not loaded
    } else if (isPlaying) {
      pauseAudio();
      isPlaying = false;
    } else {
      startAudio();
      isPlaying = true;
    }
  }

  async function updateInput() {
    pauseAudio();
    if (stream) {
      stream.getTracks().forEach(track => track.stop());
    }
    if (inputType === 'webcam') {
      try {
        stream = await navigator.mediaDevices.getUserMedia({ video: true });
        videoElement.srcObject = stream;
        await videoElement.play();
      } catch (err) {
        console.error("Error accessing the webcam", err);
      }
    } else if (inputType === 'screen') {
      try {
        stream = await navigator.mediaDevices.getDisplayMedia({ video: true });
        videoElement.srcObject = stream;
        await videoElement.play();
      } catch (err) {
        console.error("Error accessing the screen", err);
      }
    } else if (inputType === 'video' || inputType === 'image') {
      videoElement.srcObject = null;
    }
    cursorPosition = 0;
    ctx.clearRect(0, 0, canvas.width, canvas.height);
  }

  function handleFileInput(event) {
    const file = event.target.files[0];
    if (inputType === 'image') {
      const reader = new FileReader();
      reader.onload = function(e) {
        const img = new Image();
        img.onload = function() {
          canvas.width = img.width;
          canvas.height = img.height;
          ctx.drawImage(img, 0, 0);
          imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
          cursorPosition = 0;
          mediaLoaded = true;
          showModal = false;
          startAudio();
          isPlaying = true;
        }
        img.src = e.target.result;
      }
      reader.readAsDataURL(file);
    } else if (inputType === 'video') {
      videoElement.src = URL.createObjectURL(file);
      videoElement.onloadedmetadata = () => {
        canvas.width = videoElement.videoWidth;
        canvas.height = videoElement.videoHeight;
        cursorPosition = 0;
        mediaLoaded = true;
        showModal = false;
        startAudio();
        isPlaying = true;
      };
    }
  }

  function handleInputChange(event) {
    inputType = event.target.value;
    mediaLoaded = false;
    if (inputType === 'image' || inputType === 'video') {
      showModal = true;
    } else {
      updateInput();
    }
  }

  function startAudio() {
    Tone.start();
    const startTime = Tone.now() + 0.1;
    synths.forEach((synth, index) => {
      synth.oscillator.type = controls.oscillatorType;
      synth.volume.value = controls.volume;
      synth.triggerAttack(440, startTime + index * 0.01);
    });
    animationId = requestAnimationFrame(updateAudio);
  }

  function pauseAudio() {
    synths.forEach(synth => synth.triggerRelease());
    cancelAnimationFrame(animationId);
  }

  function updateAudio() {
  let sourceWidth, sourceHeight;
  if (inputType === 'image') {
    if (!imageData) return;
    sourceWidth = canvas.width;
    sourceHeight = canvas.height;
    ctx.putImageData(imageData, 0, 0);
  } else if (inputType === 'video' || inputType === 'webcam' || inputType === 'screen') {
    if (!videoElement || !videoElement.videoWidth) return;
    sourceWidth = videoElement.videoWidth;
    sourceHeight = videoElement.videoHeight;
    canvas.width = sourceWidth;
    canvas.height = sourceHeight;
    ctx.drawImage(videoElement, 0, 0, sourceWidth, sourceHeight);
  } else {
    return; // Exit if no valid input type
  }

  imageData = ctx.getImageData(0, 0, sourceWidth, sourceHeight);

  let samplePoints = [];
  const now = Tone.now();
  for (let i = 0; i < controls.voiceCount; i++) {
    // Randomize sample point location along the vertical line at cursorPosition
    let x = Math.floor(cursorPosition);
    let y = Math.floor(sourceHeight / controls.voiceCount * (i + 0.5));
    let index = (y * sourceWidth + x) * 4;
    let r = imageData.data[index] / 255;
    let g = imageData.data[index + 1] / 255;
    let b = imageData.data[index + 2] / 255;
    let a = imageData.data[index + 3] / 255;

    // Optimize color to frequency conversion
    let colorValue = (r + g + b) / 3; // Use average of RGB for better color representation
    let frequency = noteFrequencies[controls.baseNote] * Math.pow(2, controls.octave - 4 + colorValue);
    
    if (synths[i]) {
      synths[i].frequency.rampTo(frequency, 0.1, now);
    }

    samplePoints.push({ x, y });
  }

  cursorPosition += controls.cursorSpeed;
  if (cursorPosition >= sourceWidth) {
    cursorPosition = 0;
  }

  // Draw cursor (vertical line)
  ctx.fillStyle = 'rgba(255, 0, 0, 0.5)'; // Semi-transparent red
  ctx.fillRect(cursorPosition, 0, 2, sourceHeight);

  // Draw sample points
  ctx.fillStyle = 'yellow';
  samplePoints.forEach(point => {
    ctx.beginPath();
    ctx.arc(point.x, point.y, 4, 0, 2 * Math.PI);
    ctx.fill();
  });

  animationId = requestAnimationFrame(updateAudio);
}
</script>

<main>
  <canvas bind:this={canvas}></canvas>
 
  <div>
    <select bind:value={inputType} on:change={handleInputChange}>
      <option value="image">Image</option>
      <option value="video">Video</option>
      <option value="webcam">Webcam</option>
      <option value="screen">Screen</option>
    </select>
    <button on:click={togglePlay}>{isPlaying ? '⏸️ Pause' : '▶️ Play'}</button>
  </div>
   <video 
    bind:this={videoElement}
    style="display: block;,width:1%"
    controls={inputType === 'video'}
    loop=true
    muted
    autoplay
  ></video>
</main>

{#if showModal}
  <div class="modal">
    <div class="modal-content">
      <h2>Select {inputType === 'image' ? 'an image' : 'a video'} file</h2>
      <input type="file" on:change={handleFileInput} accept={inputType === 'image' ? 'image/*' : 'video/*'} />
      <button on:click={() => showModal = false}>Cancel</button>
    </div>
  </div>
{/if}

<style>
  main {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 20px;
  }
  canvas {
    border: 1px solid #000;
    margin-bottom: 20px;
  }
  .modal {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.5);
    display: flex;
    justify-content: center;
    align-items: center;
  }
  .modal-content {
    background-color: white;
    padding: 20px;
    border-radius: 5px;
  }
</style>