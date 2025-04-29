<script>
  import { onMount } from 'svelte'
  import { obs, sendCommand } from './obs.js'
  import SourceButton from './SourceButton.svelte'

  export let programScene = {}
  export let scenes = []

  let scenesFiltered = []
  let isSceneListVisible = true;

  $: scenesFiltered = scenes.filter((scene) => scene.sceneName.indexOf('(hidden)') === -1).reverse()

  onMount(async function () {
    let data = await sendCommand('GetSceneList')
    //console.log('GetSceneList', data)
    programScene = data.currentProgramSceneName || ''
    scenes = data.scenes

    const savedState = localStorage.getItem('isSceneListVisible');
    if (savedState !== null) {
      isSceneListVisible = JSON.parse(savedState); // String in Boolean umwandeln
    }
  })

  obs.on('SceneListChanged', async (data) => {
    //console.log('SceneListChanged', data.scenes.length)
    scenes = data.scenes
  })

  obs.on('SceneCreated', async (data) => {
    //console.log('SceneCreated', data)
  })

  obs.on('SceneRemoved', async (data) => {
    //console.log('SceneRemoved', data)
    for (let i = 0; i < scenes.length; i++) {
      if (scenes[i].sceneName === data.sceneName) {
        delete scenes[i]
      }
    }
  })

  obs.on('SceneNameChanged', async (data) => {
    //console.log('SceneNameChanged', data)
    for (let i = 0; i < scenes.length; i++) {
      if (scenes[i].sceneName === data.oldSceneName) {
        scenes[i].sceneName = data.sceneName
      }
    }
    // Rename in sceneIcons
    sceneIcons[data.sceneName] = sceneIcons[data.oldSceneName]
  })

  obs.on('CurrentProgramSceneChanged', (data) => {
    //console.log('CurrentProgramSceneChanged', data)
    programScene = data.sceneName || ''
  })

  function sceneClicker (scene) {
    return async function () {
      await sendCommand('SetCurrentProgramScene', { sceneName: scene.sceneName })
    }
  }

  function toggleListVisibility() {
    isSceneListVisible = !isSceneListVisible; // Sichtbarkeit umschalten
    localStorage.setItem('isSceneListVisible', JSON.stringify(isSceneListVisible)); // Zustand speichern
  }
</script>

<div class="scene-switcher">
  <h2>
    <strong>
      <button class="toggle-button" on:click={toggleListVisibility}>
        {isSceneListVisible ? '▼ Scenes' : '▶ Scenes'}
      </button>
    </strong>
  </h2>
  {#if isSceneListVisible}
    <ol>
      {#each scenesFiltered as scene}
        <li>
          <SourceButton
            name={scene.sceneName}
            on:click={sceneClicker(scene)}
            isActive={programScene === scene.sceneName}
          />
        </li>
      {/each}
    </ol>
  {/if}
</div>

<style>
  ol {
    width: 100%;
    list-style: None;
    display: block;
    margin-bottom: 2rem;
  }
  li {
    display: block;
    margin-top: 0.5rem;
    margin-bottom: 0.5rem;
  }
</style>
