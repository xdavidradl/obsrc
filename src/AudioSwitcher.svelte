<script>
  import { onMount } from 'svelte'
  import { obs, sendCommand } from './obs.js'
  import {
    mdiVolumeMute,
    mdiVolumeHigh
  } from '@mdi/js'
  import Icon from 'mdi-svelte'
  //import SourceButton from './SourceButton.svelte'

  let inputs = []
  let isAudioListVisible = true;
  //export let buttonStyle = 'text' // text, screenshot, icon

  onMount(async function () {
    await fetchInputs()

    const sliders = document.querySelectorAll('input[type="range"]')
    sliders.forEach(slider => updateSliderBackground(slider))

    const savedState = localStorage.getItem('isAudioListVisible');
    if (savedState !== null) {
      isAudioListVisible = JSON.parse(savedState); // String in Boolean umwandeln
    }
  })

  async function fetchInputs() {
    let data = await sendCommand('GetInputList')
    data.inputs.reverse()
    inputs = []
    for (let input of data.inputs) {
      let muteData = await sendCommand('GetInputMute', {
        inputUuid: input.inputUuid
      })
      let volumeData = await sendCommand('GetInputVolume', {
        inputUuid: input.inputUuid
      })
      //console.log(Object.keys(muteData).length, Object.keys(volumeData).length)
      if (Object.keys(muteData).length > 0 && Object.keys(volumeData).length > 0) {
        input.inputMuted = muteData.inputMuted
        input.inputVolumeDb = volumeData.inputVolumeDb
        input.inputVolumeMul = volumeData.inputVolumeMul
        //console.log(input)
        inputs = [...inputs, input]
      }
    }
    //console.log(inputs)
  }

  obs.on('InputCreated', async (data) => {
    await fetchInputs()
  })

  obs.on('InputRemoved', async (data) => {
    await fetchInputs()
  })

  obs.on('InputNameChanged', async (data) => {
    for (let i = 0; i < inputs.length; i++) {
      if (inputs[i].inputUuid === data.inputUuid) {
        inputs[i].inputName = data.inputName
        //console.log(inputs[i].inputName, 'changed')
      }
    }
  })

  obs.on('InputMuteStateChanged', async (data) => {
    for (let i = 0; i < inputs.length; i++) {
      if (inputs[i].inputUuid === data.inputUuid) {
        inputs[i].inputMuted = data.inputMuted
        //console.log(inputs[i].inputName, inputs[i].inputMuted)
      }
    }
  })

  obs.on('InputVolumeChanged', async (data) => {
    for (let i = 0; i < inputs.length; i++) {
      if (inputs[i].inputUuid === data.inputUuid) {
        inputs[i].inputVolumeDb = data.inputVolumeDb
        inputs[i].inputVolumeMul = data.inputVolumeMul
        //console.log(inputs[i].inputName, inputs[i].inputVolumeDb, inputs[i].inputVolumeMul)
      }
    }
    //const sliders = document.querySelectorAll('input[type="range"]')
    //sliders.forEach(slider => updateSliderBackground(slider))
    const slider = document.querySelector(`input[data-uuid="${data.inputUuid}"]`);
    if (slider) {
      console.log('Slider found for input:', data.inputName)
      slider.value = Math.sqrt(data.inputVolumeMul); // Logarithmische Umrechnung
      updateSliderBackground(slider); // Hintergrund aktualisieren
    }
  })

  async function toggleMute(input) {
    let muted = await sendCommand('ToggleInputMute', {
      inputUuid: input.inputUuid
    })
    //console.log(input.inputName, 'toggleMute muted:', input.inputMuted)
  }

  async function setVolume(input, event) {
    let linearValue = parseFloat(event.target.value)
    let volume = Math.pow(linearValue, 2); // Quadratische Skala (logarithmisch)
    //console.log(volume)
    await sendCommand('SetInputVolume', {
      inputUuid: input.inputUuid,
      inputVolumeMul: volume
    })

    input.inputVolumeMul = volume; // Aktualisiere den Wert im Input-Objekt
    updateSliderBackground(event.target)
  }

  function updateSliderBackground(slider) {
    let value = (slider.value - slider.min) / (slider.max - slider.min) * 100;
    slider.value = slider.value; // Setze den Wert des Sliders auf den Prozentsatz
    slider.style.background = `linear-gradient(to right, #3e8ed0 ${value}%, #aaaaaa ${value}%)`
  }

  function toggleListVisibility() {
    isAudioListVisible = !isAudioListVisible; // Sichtbarkeit umschalten
    localStorage.setItem('isAudioListVisible', JSON.stringify(isAudioListVisible)); // Zustand speichern

    setTimeout(() => {
      const sliders = document.querySelectorAll('input[type="range"]');
      sliders.forEach(slider => updateSliderBackground(slider));
    }, 0); 
  }
</script>

<div class="audio-switcher">
  <h2>
    <strong>
      <button class="toggle-button" on:click={toggleListVisibility}>
        {isAudioListVisible ? '▼ Audio' : '▶ Audio'}
      </button>
    </strong>
  </h2>
  {#if isAudioListVisible}
    {#each inputs as input}
      <div class="audio-control">
        <div class="input-name">{input.inputName}
          <div class="audio-switcher-row">
            <button class="mute-button" on:click={() => toggleMute(input)}>
              <span class="icon">
                {#if input.inputMuted}
                  <Icon path={mdiVolumeMute} color='red'/>
                {:else}
                  <Icon path={mdiVolumeHigh} />
                {/if}
              </span>
            </button>
            <input
              class="volume-slider"
              type="range"
              min="0"
              max="1"
              step="0.01"
              data-uuid={input.inputUuid}
              value={Math.sqrt(input.inputVolumeMul)}
              on:input={(event) => setVolume(input, event)}
            />
          </div>
        </div>
      </div>
    {/each}
  {/if}
</div>

<!--value={input.inputVolumeMul}-->
<style>
  .audio-switcher {
    display: grid;
    gap: 1rem;
  }

  .audio-control {
    width: 100%;
    display: flex;
    flex-direction: column;
    align-items: flex-start;
    gap: 0.5rem;
  }

  .input-name {
    font-weight: bold;
    width: 100%;
  }

  .audio-switcher-row {
    display: flex;
    align-items: center;
    gap: 1rem;
  }

  .mute-button {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 2rem;
    padding: 0.5rem;
    border: none;
    cursor: pointer;
    background-color: #666666;
    color: #fff;
    border-radius: 5px;
  }

  .volume-slider {
    width: 100%;
    height: 0.75rem;
    appearance: none;
    background: linear-gradient(to right, #3e8ed0 50%, #aaaaaa 50%);
    border-radius: 5px;
  }

  .volume-slider::-webkit-slider-thumb {
    width: 3rem;
    height: 1.5em;
    background: #777;
    border-radius: 5px;
    cursor: pointer;
    appearance: none;
  }

  .volume-slider::-moz-range-thumb {
    width: 3rem;
    height: 1.5rem;
    background: #777;
    border-radius: 5px;
    cursor: pointer;
  }

</style>
  