<script>
  /* eslint-env browser */
  const OBS_WEBSOCKET_LATEST_VERSION = '5.0.1' // https://api.github.com/repos/Palakis/obs-websocket/releases/latest

  // Imports
  import { onMount } from 'svelte'
  import {
    mdiFullscreen,
    mdiFullscreenExit,
    mdiAccessPoint,
    mdiAccessPointOff,
    mdiConnection,
  } from '@mdi/js'
  import Icon from 'mdi-svelte'
  import { compareVersions } from 'compare-versions'

  import './style.scss'
  import { obs, sendCommand } from './obs.js'
  import SceneSwitcher from './SceneSwitcher.svelte'
  import SourceSwitcher from './SourceSwitcher.svelte'
  import ProfileSelect from './ProfileSelect.svelte'
  import SceneCollectionSelect from './SceneCollectionSelect.svelte'
  import AudioSwitcher from './AudioSwitcher.svelte'
  import { OBSWebSocketError } from 'obs-websocket-js';

  onMount(async () => {
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.register('/service-worker.js')
    }

    // Request screen wakelock
    if ('wakeLock' in navigator) {
      try {
        await navigator.wakeLock.request('screen')
        // Re-request when coming back
        document.addEventListener('visibilitychange', async () => {
          if (document.visibilityState === 'visible') {
            await navigator.wakeLock.request('screen')
          }
        })
      } catch (e) {}
    }

    // Toggle the navigation hamburger menu on mobile
    const navbar = document.querySelector('.navbar-burger')
    navbar.addEventListener('click', () => {
      navbar.classList.toggle('is-active')
      document
        .getElementById(navbar.dataset.target)
        .classList.toggle('is-active')
    })

    // Listen for fullscreen changes
    document.addEventListener('fullscreenchange', () => {
      isFullScreen = document.fullscreenElement
    })

    document.addEventListener('webkitfullscreenchange', () => {
      isFullScreen = document.webkitFullscreenElement
    })

    document.addEventListener('msfullscreenchange', () => {
      isFullScreen = document.msFullscreenElement
    })

    if (document.location.hash !== '') {
      // Read address from hash
      address = document.location.hash.slice(1)

      // This allows you to add a password in the URL like this:
      // http://192.168.1.10/#ws://localhost:4455#password
      if (address.includes('#')) {
        [address, password] = address.split('#')
      }
      await connect()
    }

    // Export the sendCommand() function to the window object
    window.sendCommand = sendCommand
  })

  // State
  let connected
  let heartbeat = {}
  let heartbeatInterval
  let isFullScreen
  let address
  let password
  let scenes = []
  let programScene = {}
  let errorMessage = ''
  let imageFormat = 'jpg'

  function formatTime (secs) {
    secs = Math.round(secs / 1000)
    const hours = Math.floor(secs / 3600)
    secs -= hours * 3600
    const mins = Math.floor(secs / 60)
    secs -= mins * 60
    return hours > 0
      ? `${hours}:${mins < 10 ? '0' : ''}${mins}:${secs < 10 ? '0' : ''}${secs}`
      : `${mins < 10 ? '0' : ''}${mins}:${secs < 10 ? '0' : ''}${secs}`
  }

  function toggleFullScreen () {
    if (isFullScreen) {
      if (document.exitFullscreen) {
        document.exitFullscreen()
      } else if (document.webkitExitFullscreen) {
        document.webkitExitFullscreen()
      } else if (document.msExitFullscreen) {
        document.msExitFullscreen()
      }
    } else {
      if (document.documentElement.requestFullscreen) {
        document.documentElement.requestFullscreen()
      } else if (document.documentElement.webkitRequestFullscreen) {
        document.documentElement.webkitRequestFullscreen()
      } else if (document.documentElement.msRequestFullscreen) {
        document.documentElement.msRequestFullscreen()
      }
    }
  }

  async function startStream () {
    await sendCommand('StartStream')
  }

  async function stopStream () {
    await sendCommand('StopStream')
  }

  async function connect () {
    if (address)
    {
      if (address.indexOf('://') === -1) {
        const secure = location.protocol === 'https:' || address.endsWith(':443')
        address = secure ? 'wss://' : 'ws://' + address
      }
      //console.log('Connecting to:', address, '- using password:', password)
      await disconnect()
      try {
        const { obsWebSocketVersion, negotiatedRpcVersion } = await obs.connect(
          address,
          password
        )
        //console.log(`Connected to obs-websocket version ${obsWebSocketVersion} (using RPC ${negotiatedRpcVersion})`)
      } catch (e) {
        console.log(e)
        errorMessage = e.message
      }
    }
    else {
      errorMessage = 'Please enter a valid address.'
    }
  }

  async function disconnect () {
    await obs.disconnect()
    clearInterval(heartbeatInterval)
    connected = false
    errorMessage = 'Disconnected'
  }

  // OBS events
  obs.on('ConnectionClosed', () => {
    connected = false
    window.history.pushState(
      '',
      document.title,
      window.location.pathname + window.location.search
    ) // Remove the hash
    //console.log('Connection closed')
  })

  obs.on('Identified', async () => {
    //console.log('Connected')
    connected = true
    document.location.hash = address // For easy bookmarking
    const data = await sendCommand('GetVersion')
    const version = data.obsWebSocketVersion || ''
    //console.log('OBS-websocket version:', version)
    if (compareVersions(version, OBS_WEBSOCKET_LATEST_VERSION) < 0) {
      alert(
        'You are running an outdated OBS-websocket (version ' +
          version +
          '), please upgrade to the latest version for full compatibility.'
      )
    }
    if (
      data.supportedImageFormats.includes('webp') &&
      document
        .createElement('canvas')
        .toDataURL('image/webp')
        .indexOf('data:image/webp') === 0
    ) {
      imageFormat = 'webp'
    }
    heartbeatInterval = setInterval(async () => {
      const stats = await sendCommand('GetStats')
      const streaming = await sendCommand('GetStreamStatus')
      heartbeat = { stats, streaming }
      // console.log(heartbeat);
    }, 1000) // Heartbeat
  })

  obs.on('ConnectionError', async () => {
    errorMessage = 'Please enter your password:'
    document.getElementById('password').focus()
    if (!password) {
      connected = false
    } else {
      await connect()
    }
  })
</script>

<svelte:head>
  <title>OBSrc.com</title>
</svelte:head>

<nav class="navbar is-primary" aria-label="main navigation">
  <div class="navbar-brand">
    <a class="navbar-item is-size-4 has-text-weight-bold" href="/">
      OBSrc.com</a
    >

    <!-- svelte-ignore a11y-missing-attribute -->
    <button
      class="navbar-burger burger"
      aria-label="menu"
      aria-expanded="false"
      data-target="navmenu"
    >
      <span aria-hidden="true" />
      <span aria-hidden="true" />
      <span aria-hidden="true" />
    </button>
  </div>

  <div id="navmenu" class="navbar-menu">
    <div class="navbar-end">
      <div class="navbar-item">
        <div class="buttons">
          <!-- svelte-ignore a11y-missing-attribute -->
          {#if connected}
            <button class="button is-info is-light" disabled>
              {#if heartbeat && heartbeat.stats}
                {Math.round(heartbeat.stats.activeFps)} fps, {Math.round(
                  heartbeat.stats.cpuUsage
                )}% CPU, {heartbeat.stats.renderSkippedFrames} skipped frames
              {:else}Connected{/if}
            </button>
            {#if heartbeat && heartbeat.streaming && heartbeat.streaming.outputActive}
              <button
                class="button is-danger"
                on:click={stopStream}
                title="Stop Stream"
              >
                <span class="icon"><Icon path={mdiAccessPointOff} /></span>
                <span>{formatTime(heartbeat.streaming.outputDuration)}</span>
              </button>
            {:else}
              <button
                class="button is-danger is-light"
                on:click={startStream}
                title="Start Stream"
              >
                <span class="icon"><Icon path={mdiAccessPoint} /></span>
              </button>
            {/if}
            <ProfileSelect />
            <SceneCollectionSelect />
            <button
              class="button is-danger is-light"
              on:click={disconnect}
              title="Disconnect"
            >
              <span class="icon"><Icon path={mdiConnection} /></span>
            </button>
          {:else}
            <button class="button is-danger" disabled
              >{errorMessage || 'Disconnected'}</button
            >
          {/if}
          <!-- svelte-ignore a11y-missing-attribute -->
          <button
            class:is-light={!isFullScreen}
            class="button is-link"
            on:click={toggleFullScreen}
            title="Toggle Fullscreen"
          >
            <span class="icon">
              <Icon path={isFullScreen ? mdiFullscreenExit : mdiFullscreen} />
            </span>
          </button>
        </div>
      </div>
    </div>
  </div>
</nav>

<section class="section">
    {#if connected}
      <div class="container">
        <SceneSwitcher
          bind:scenes
          bind:programScene
        />
        <SourceSwitcher
          bind:programScene
        />
        <AudioSwitcher/>
      </div>
    {:else}
      <div class="no-wrap-conainer">
        <h1 class="subtitle">
          Welcome to
          <strong>OBSrc.com</strong>
        </h1>

        <p>Enter your OBS remote control url and password below and click "Connect".</p>

        <form on:submit|preventDefault={connect}>
          <div class="field is-grouped">
            <p class="control is-expanded">
              <input
                id="username"
                bind:value={address}
                class="input"
                type="text"
                autocomplete="username"
                placeholder="wss://obsrc.com/obs-relay/remote-controller/12345678-1234-1234-1234-123456789123"
              />
              <input
                id="password"
                bind:value={password}
                class="input"
                type="password"
                name="password"
                autocomplete="current-password"
                placeholder="password"
              />
            </p>
            <p class="control">
              <button class="button is-success">Connect</button>
            </p>
          </div>
        </form>
      </div> 
    {/if}
</section>