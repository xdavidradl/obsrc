<script>
  export let programScene
  let items = []

  import { onMount } from 'svelte'
  import { obs, sendCommand } from './obs.js'
  import SourceButton from './SourceButton.svelte'

  let isSourceListVisible = true;

  onMount(async function () {
    await refreshItems()

    const savedState = localStorage.getItem('isSourceListVisible');
    if (savedState !== null) {
      isSourceListVisible = JSON.parse(savedState); // String in Boolean umwandeln
    }
  })

  $: if (programScene) {
    refreshItems()
  }

  async function refreshItems () {
    //console.log('sceneName', programScene)
    const data = await sendCommand('GetSceneItemList', { sceneName: programScene })
    let itemshlp = []
    itemshlp = data.sceneItems || itemshlp
    itemshlp.reverse()
    let i = 0;
    //console.log('GetSceneItemList', data)
    if (itemshlp.length > 0) {
      for (i = 0; i < itemshlp.length; i++) {
        if (itemshlp[i].isGroup) {
          itemshlp[i].sceneName = programScene
          const groupData = await sendCommand('GetGroupSceneItemList', {
            sceneUuid: itemshlp[i].sourceUuid
          })
          groupData.sceneItems.reverse()
          //console.log('groupData', groupData)
          //die groupData ist ein array mit items. Diese zu dem items[i] an diser stelle dazwischen einfügen
          //in jedes item der groupData den name der guppe einfügen, der es zugehört
          for (let j = 0; j < groupData.sceneItems.length; j++) {
            groupData.sceneItems[j].isGroupChild = true
            groupData.sceneItems[j].sceneName = itemshlp[i].sourceName
            //console.log('groupData sceneName', groupData.sceneItems[j].sceneName)
          }
          itemshlp.splice(i + 1, 0, ...groupData.sceneItems)
        }
        else if (!itemshlp[i].isGroupChild)
        {
          itemshlp[i].sceneName = programScene
        }
      }
    }
    items = itemshlp
    //console.log('items', items)
  }

  obs.on('SceneItemEnableStateChanged', async (data) => {
    //console.log('SceneItemEnableStateChanged', data)
    //iteriere durch alle items und finde das item wo die sceneItemId mit der id übereinstimmt und der sceneName mit dem sceneName übereinstimmt
    for (let i = 0; i < items.length; i++) {
      if (items[i].sceneItemId === data.sceneItemId && items[i].sceneName === data.sceneName) {
        //console.log('SceneItemEnableStateChanged i', i, items[i])
        items[i].sceneItemEnabled = data.sceneItemEnabled
      }
    }
  })

  obs.on('SceneItemListReindexed', async (data) => {
    await refreshItems()
  })

  obs.on('SceneItemCreated', async (data) => {
    await refreshItems()
  })

  obs.on('SceneItemRemoved', async (data) => {
    await refreshItems()
  })

  function backgroundClicker (item) {
    return async function () {
      //console.log('item', item)
      if (item.sceneItemEnabled) {
        await sendCommand('SetSceneItemEnabled', {
          sceneName: item.sceneName,
          sceneItemId: item.sceneItemId,
          sceneItemEnabled: false
        })
      }
      else {
        await sendCommand('SetSceneItemEnabled', {
          sceneName: item.sceneName,
          sceneItemId: item.sceneItemId,
          sceneItemEnabled: true
        })
      }
    }
  }

  function toggleListVisibility() {
    isSourceListVisible = !isSourceListVisible; // Sichtbarkeit umschalten
    localStorage.setItem('isSourceListVisible', JSON.stringify(isSourceListVisible)); // Zustand speichern
  }
</script>

<div class="source-switcher">
  <h2>
    <strong>
      <button class="toggle-button" on:click={toggleListVisibility}>
        {isSourceListVisible ? '▼ Sources' : '▶ Sources'}
      </button>
    </strong>
  </h2>
  {#if isSourceListVisible}
    <ol>
      {#each items as item}
      <li>
        <SourceButton name={item.sourceName}
          on:click={backgroundClicker(item)}
          isActive={item.sceneItemEnabled}
          isGroup={item.isGroup}
          isGroupChild={item.isGroupChild}
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
