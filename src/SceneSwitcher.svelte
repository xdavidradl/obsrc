<script>
  import { onMount } from 'svelte'
  import { obs, sendCommand } from './obs.js'
  import SourceButton from './SourceButton.svelte'

  export let programScene = {}
  export let scenes = []
  export let buttonStyle = 'text' // text, screenshot, icon

  let scenesFiltered = []

  $: scenesFiltered = scenes.filter((scene) => scene.sceneName.indexOf('(hidden)') === -1).reverse()

  onMount(async function () {
    let data = await sendCommand('GetSceneList')
    console.log('GetSceneList', data)
    programScene = data.currentProgramSceneName || ''
    scenes = data.scenes
  })

  obs.on('SceneListChanged', async (data) => {
    console.log('SceneListChanged', data.scenes.length)
    scenes = data.scenes
  })

  obs.on('SceneCreated', async (data) => {
    console.log('SceneCreated', data)
  })

  obs.on('SceneRemoved', async (data) => {
    console.log('SceneRemoved', data)
    for (let i = 0; i < scenes.length; i++) {
      if (scenes[i].sceneName === data.sceneName) {
        delete scenes[i]
      }
    }
  })

  obs.on('SceneNameChanged', async (data) => {
    console.log('SceneNameChanged', data)
    for (let i = 0; i < scenes.length; i++) {
      if (scenes[i].sceneName === data.oldSceneName) {
        scenes[i].sceneName = data.sceneName
      }
    }
    // Rename in sceneIcons
    sceneIcons[data.sceneName] = sceneIcons[data.oldSceneName]
  })

  obs.on('CurrentProgramSceneChanged', (data) => {
    console.log('CurrentProgramSceneChanged', data)
    programScene = data.sceneName || ''
  })

  function sceneClicker (scene) {
    return async function () {
      await sendCommand('SetCurrentProgramScene', { sceneName: scene.sceneName })
    }
  }
</script>

<ol
  class:with-icon={buttonStyle === 'icon'}
  >
  {#each scenesFiltered as scene}
  <li>
    <SourceButton name={scene.sceneName}
      on:click={sceneClicker(scene)}
      isProgram={programScene === scene.sceneName}
      buttonStyle={buttonStyle}
    />
  </li>
  {/each}
</ol>

<style>
  ol {
    list-style: None;
    display: flex;
    flex-wrap: wrap;
    justify-content: space-between;
    gap: .5rem;
    margin-bottom: 2rem;
  }
  li {
    display: inline-block;
    min-width: 10rem;
    flex-grow: 1;
  }
  ol.with-icon {
    justify-content: center;
  }
  ol.with-icon li {
    min-width: 0;
    flex-grow: 0;
    flex-shrink: 1;
  }
</style>
