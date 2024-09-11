<script>
  export let buttonStyle = 'text'
  export let programScene
  export let programSources
  let items = []
  const itemsIndex = {}

  import { onMount } from 'svelte'
  import { obs, sendCommand } from './obs.js'
  import SourceButton from './SourceButton.svelte'

  onMount(async function () {
    await refreshItems()
  })

  $: if (programScene) {
    refreshItems()
  }

  async function refreshItems () {
    //console.log('sceneName', programScene)
    const data = await sendCommand('GetSceneItemList', { sceneName: programScene })
    items = []
    programSources = ''
    items = data.sceneItems || items
    items.reverse()
    //console.log('GetSceneItemList', data)
    if (items.length > 0) {
      let i = 0;
      const item = items[i]
      itemsIndex[item.sceneItemId] = i
      programSources += item.sourceName
      i++
      for (; i < items.length; i++) {
        const item = items[i]
        itemsIndex[item.sceneItemId] = i
        programSources += ', ' + item.sourceName
      }
    } 
    else {
      programSources = 'n/a'
    }
    //console.log('programSources', programSources)
  }

  obs.on('SceneItemEnableStateChanged', async (data) => {
    if (data.sceneName === programScene) {
      const i = itemsIndex[data.sceneItemId]
      items[i].sceneItemEnabled = data.sceneItemEnabled
    }
  })

  obs.on('SceneItemListReindexed', async (data) => {
    if (data.sceneName === programScene) {
      await refreshItems()
    }
  })

  obs.on('SceneItemCreated', async (data) => {
    if (data.sceneName === programScene) {
      await refreshItems()
    }
  })

  obs.on('SceneItemRemoved', async (data) => {
    if (data.sceneName === programScene) {
      await refreshItems()
    }
  })

  function backgroundClicker (item) {
    return async function () {
      let data = null
      //console.log('item', item)
      //This is a fix if a camera was disconnected. Resetting the settings seems to reload the stream/device
      data = await sendCommand('GetInputSettings', {
        inputUuid: item.sourceUuid
      })
      //console.log('1 GetInputSettings', data)
      if (item.sceneItemEnabled) {
        await sendCommand('SetSceneItemEnabled', {
          sceneName: programScene,
          sceneItemId: item.sceneItemId,
          sceneItemEnabled: false
        })
        if (data && data.inputSettings && 'active' in data.inputSettings) {
          data.inputSettings.active = false
        }
      }
      else {
        await sendCommand('SetSceneItemEnabled', {
          sceneName: programScene,
          sceneItemId: item.sceneItemId,
          sceneItemEnabled: true
        })
        if (data && data.inputSettings && 'active' in data.inputSettings) {
          data.inputSettings.active = true
        }
      }
      //console.log('2 GetInputSettings', data)
      await sendCommand('SetInputSettings', {
        inputUuid: item.sourceUuid,
        inputSettings: data.inputSettings
      })
    }
  }
</script>

<ol>
  {#each items as item}
  <li>
    <SourceButton name={item.sourceName}
      on:click={backgroundClicker(item)}
      isProgram={item.sceneItemEnabled}
      buttonStyle={buttonStyle}
    />
  </li>
  {/each}
</ol>

<style>
  /*ol {
    list-style: None;
    display: flex;
    flex-wrap: wrap;
    align-content: space-between;
    gap: .5rem;
    margin-bottom: 2rem;
  }
  li {
    max-width: 192px;
    white-space: nowrap;
  }*/
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
</style>
