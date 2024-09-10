<script>
  import { onMount, onDestroy } from 'svelte'
  import { obs, sendCommand } from './obs.js'

  import { Client } from 'tmi.js'
  import { tick } from 'svelte'
  import { mdiMessageSettings } from '@mdi/js';

  export let imageFormat = 'jpg'
  export let hidden = true

  let isStudioMode = false
  let programScene = ''
  let previewScene = ''

  let program = {}
  let preview = {}
  let screenshotInterval
  let transitions = []
  // let currentTransition = ''

  onMount(async () => {
    let data
    if (!programScene) {
      data = await sendCommand('GetCurrentProgramScene')
      programScene = data.currentProgramSceneName || ''
    }
    data = await sendCommand('GetStudioModeEnabled')
    if (data && data.studioModeEnabled) {
      isStudioMode = true
      data = await sendCommand('GetCurrentPreviewScene')
      previewScene = data.currentPreviewSceneName || ''
    }

    data = await sendCommand('GetSceneTransitionList')
    console.log('GetSceneTransitionList', data)
    transitions = data.transitions || []
    // currentTransition = data.currentSceneTransitionName || ''
    screenshotInterval = setInterval(getScreenshot, 1000)
  })

  onDestroy(() => {
    clearInterval(screenshotInterval)
  })

  // eslint-disable-next-line
  $: getScreenshot(), programScene, previewScene

  obs.on('StudioModeStateChanged', async (data) => {
    console.log('StudioModeStateChanged', data.studioModeEnabled)
    isStudioMode = data.studioModeEnabled
    if (isStudioMode) {
      previewScene = programScene
    }
  })

  obs.on('CurrentPreviewSceneChanged', async (data) => {
    console.log('CurrentPreviewSceneChanged', data.sceneName)
    previewScene = data.sceneName
  })

  obs.on('CurrentProgramSceneChanged', async (data) => {
    console.log('CurrentProgramSceneChanged', data.sceneName)
    programScene = data.sceneName
  })

  obs.on('SceneNameChanged', async (data) => {
    if (data.oldSceneName === programScene) programScene = data.sceneName
    if (data.oldSceneName === previewScene) previewScene = data.sceneName
  })

  // TODO: does not exist???
  obs.on('TransitionListChanged', async (data) => {
    console.log('TransitionListChanged', data)
    transitions = data.transitions || []
  })

  async function getScreenshot () {
    if (!programScene) return
    let data = await sendCommand('GetSourceScreenshot', {
      sourceName: programScene,
      imageFormat,
      imageWidth: 960,
      imageHeight: 540
    })
    if (data && data.imageData && program) {
      program.src = data.imageData
      program.className = ''
    }

    if (isStudioMode) {
      if (previewScene !== programScene) {
        data = await sendCommand('GetSourceScreenshot', {
          sourceName: previewScene,
          imageFormat,
          imageWidth: 960,
          imageHeight: 540
        })
      }
      if (data && data.imageData && preview) {
        preview.src = data.imageData
      }
    }
  }


  //Chat
  let messages = []
  let chatDiv
  let isAtBottom = true
  let showJumpButton = false

  const client = new Client({
      channels: ['davidradl']
  })

  client.connect();   

  client.on('message', (channel, tags, message, self) => {
      //if (tags['color'] == )
      let newMessage = { username: tags['display-name'], color: tags['color'], message}
      console.log('<', tags['color'], '> ', tags['display-name'], ': ', message)
      messages = [...messages, newMessage]
      if (messages.length > 100) {
        messages.shift()
      }
      console.log(messages.length)
  })

  async function scrollToBottom() {
      await tick()
      chatDiv.scrollTop = chatDiv.scrollHeight
  }

  function handleScroll() {
      const scrollPosition = chatDiv.scrollTop + chatDiv.clientHeight
      const isBottom = chatDiv.scrollHeight - scrollPosition < 5

      isAtBottom = isBottom;
      showJumpButton = !isBottom
  }

  $: if (messages.length > 0 && isAtBottom) {//isScrolledToBottom()) {
      scrollToBottom();
  }
</script>

<div class="container"
  class:hidden={hidden}>
  
  <img class="img-container" bind:this={program} alt="Program"/>

  {#if messages.length > 0}
    <div 
      bind:this={chatDiv}
      class="chat-container"
      on:scroll={handleScroll}>

      {#each messages as message}
          <p><strong style="color: {message.color}">{message.username}:</strong> {message.message}</p>
      {/each}

      {#if showJumpButton}
          <button class="jump-to-bottom" on:click={scrollToBottom}>
              Jump to bottom
          </button>
      {/if}
    </div>
  {/if}
</div>

<style>
  .hidden {
    display: none
  }

  .container {
    position: relative;
  }

  .img-container {
    display: block;
    width: 100%;
  }

  /*Chat*/
  .chat-container {
      position: absolute;
      bottom: 6px;
      left: 0;
      max-width: 50%;
      max-height: 200px;
      overflow-y: auto;
      background: rgba(80, 80, 80, 0.4);
      color: rgb(255, 255, 255);
      padding: 10px;
      border-radius: 5px;
      font-family: Arial, sans-serif;
    }

    .chat-container p {
        margin: 5px 0;
        line-height: 1.2;
    }

    .chat-container::-webkit-scrollbar {
        display: none;
    }

    .chat-container {
        scrollbar-width: none;
    }

    .jump-to-bottom {
        position: sticky;
        bottom: 0;
        left: 0;
        width: 100%;
        padding: 10px;
        background-color: #3e8ed0;
        color: rgb(255, 255, 255);
        border: none;
        border-radius: 5px;
        cursor: pointer;
        opacity: 0.6;
        z-index: 1000;
        text-align: center;
    }

    .jump-to-bottom:hover {
        opacity: 1;
    }
</style>
