<script>
  import { onMount } from 'svelte'
  import SourceButton from './SourceButton.svelte'

  export let buttonStyle = 'text' // text, screenshot, icon
  export let tabs = []
  export let currentTab


  function TabClicker (tab) {
    return async function () {
      currentTab = tab.index
    }
  }
</script>

<ol>
  {#each tabs as tab}
  <li>
    <SourceButton name={tab.name}
      on:click={TabClicker(tab)}
      isProgram={currentTab === tab.index}
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
</style>
