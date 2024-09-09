<script>
  export let buttonStyle = 'screenshot'
  export let programScene
  export let programSources
  export let hidden = true
  //export let name = 'Backgrounds (hidden)'
  //export let imageFormat = 'jpg'
  let items = []
  const itemsIndex = {}
  //let currentItemId = ''
  const screenshottedIds = new Set()

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
    console.log('sceneName', programScene)
    const data = await sendCommand('GetSceneItemList', { sceneName: programScene })
    items = []
    programSources = ''
    items = data.sceneItems || items
    items.reverse()
    console.log('GetSceneItemList', data)
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
        /*if (item.sceneItemEnabled) {
          currentItemId = item.sceneItemId
        }*/
      }
    } 
    else {
      programSources = 'n/a'
    }
      console.log('programSources', programSources)
    /*for (let i = 0; i < items.length; i++) {
      items[i].img = await getItemScreenshot(items[i])
    }*/
  }

  obs.on('SceneItemEnableStateChanged', async (data) => {
    if (data.sceneName === programScene) {
      const i = itemsIndex[data.sceneItemId]
      items[i].sceneItemEnabled = data.sceneItemEnabled
      /*if (items[i].sceneItemEnabled && !items[i].img) {
        items[i].img = await getItemScreenshot(items[i])
        if (screenshottedIds.has(items[i].sceneItemId)) {
          items[i].img = await getItemScreenshot(items[i])
          await sendCommand('SetSceneItemEnabled', {
            sceneName: name,
            sceneItemId: items[i].sceneItemId,
            sceneItemEnabled: false
          })
          screenshottedIds.delete(items[i].sceneItemId)
        }
      }*/
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

  /*
  async function getItemScreenshot (item) {
    if (item.img) return item.img
    let data = null
    let retry = item.sceneItemEnabled ? 3 : 1
    while (retry--) {
      // Random sleep to avoid burst of thumbnail rendering
      await new Promise((resolve) => setTimeout(resolve, Math.random() * 500 + 100))
      data = await sendCommand('GetSourceScreenshot', {
        sourceName: item.sourceName,
        imageFormat,
        width: 192,
        height: 108
      })
      if (data && data.imageData) {
        return data.imageData
      }
    }
  }
  */

  /*
  async function loadMissingScreenshots () {
    for (let i = 0; i < items.length; i++) {
      if (!items[i].img) {
        await sendCommand('SetSceneItemEnabled', {
          sceneName: name,
          sceneItemId: items[i].sceneItemId,
          sceneItemEnabled: true
        })
        screenshottedIds.add(items[i].sceneItemId)
      }
    }
  }*/
</script>

<ol>
  {#each items as item}
  <li>
    <SourceButton name={item.sourceName} subname={null}
      on:click={backgroundClicker(item)}
      isProgram={item.sceneItemEnabled}
      buttonStyle={buttonStyle}
      hidden={hidden}
    />
  </li>
  {/each}
</ol>
<!--<button class="button" on:click={loadMissingScreenshots}>Load missing thumbnails</button>-->

<style>
  ol {
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
  }
</style>
