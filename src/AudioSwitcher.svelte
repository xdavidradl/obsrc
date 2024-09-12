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
  //export let buttonStyle = 'text' // text, screenshot, icon

  onMount(async function () {
    await fetchInputs()

    const sliders = document.querySelectorAll('input[type="range"]')
    sliders.forEach(slider => updateSliderBackground(slider))
  })

  async function fetchInputs() {
    console.log('start 1')
    let data = await sendCommand('GetInputList')
    console.log('GetSceneList', data)

    inputs = []
    for (let input of data.inputs) {
      let muteData = await sendCommand('GetInputMute', {
        inputUuid: input.inputUuid
      })
      let volumeData = await sendCommand('GetInputVolume', {
        inputUuid: input.inputUuid
      })
      console.log(Object.keys(muteData).length, Object.keys(volumeData).length)
      if (Object.keys(muteData).length > 0 && Object.keys(volumeData).length > 0) {
        input.inputMuted = muteData.inputMuted
        input.inputVolumeDb = volumeData.inputVolumeDb
        input.inputVolumeMul = volumeData.inputVolumeMul
        console.log(input)
        inputs = [...inputs, input]
      }
    }
    console.log(inputs)
    //data = await sendCommand('GetInputKindList')
    //console.log('GetInputKindList', data)
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
        console.log(inputs[i].inputName, 'changed')
      }
    }
  })

  obs.on('InputMuteStateChanged', async (data) => {
    for (let i = 0; i < inputs.length; i++) {
      if (inputs[i].inputUuid === data.inputUuid) {
        inputs[i].inputMuted = data.inputMuted
        console.log(inputs[i].inputName, inputs[i].inputMuted)
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
    const sliders = document.querySelectorAll('input[type="range"]')
    sliders.forEach(slider => updateSliderBackground(slider))
  })

  async function toggleMute(input) {
    let muted = await sendCommand('ToggleInputMute', {
      inputUuid: input.inputUuid
    })
    //console.log(input.inputName, 'toggleMute muted:', input.inputMuted)
  }

  async function setVolume(input, event) {
    let volume = parseFloat(event.target.value)
    //console.log(volume)
    await sendCommand('SetInputVolume', {
      inputUuid: input.inputUuid,
      inputVolumeMul: volume
    })
    updateSliderBackground(event.target)
  }

  function updateSliderBackground(slider) {
    let value = (slider.value - slider.min) / (slider.max - slider.min) * 100;
    slider.style.background = `linear-gradient(to right, #3e8ed0 ${value}%, #aaaaaa ${value}%)`
  }
</script>

<div class="controls">
  {#each inputs as input}
    <div class="audio-control">
      <div class="input-name">{input.inputName}
        <div class="controls-row">
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
            value={input.inputVolumeMul}
            on:input={(event) => setVolume(input, event)}
          />
        </div>
      </div>
    </div>
  {/each}
</div>

<style>
  .controls {
    display: grid;
    gap: 1rem;
    width: 100%;
  }

  .audio-control {
    width: 100%;
    display: flex;
    flex-direction: column;
    align-items: flex-start;
    gap: 1rem;
  }

  .input-name {
    font-weight: bold;
    margin-bottom: 0.5rem;
    width: 100%;
  }

  .controls-row {
    display: flex;
    align-items: center;
    gap: 1rem;
  }

  .mute-button {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 3rem;
    width: 4rem;
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
  