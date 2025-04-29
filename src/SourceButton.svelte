<script>
  export let name
  export let isActive = false
  export let isGroup = false
  export let isGroupChild = false

  import { createEventDispatcher } from 'svelte'
  const dispatch = createEventDispatcher()
</script>

<button
  class:title={true}
  class:active={isActive}
  class:isGroupChild={isGroupChild}
  on:click={() => dispatch('click')}
  >
  {#if isGroup}
    <span class="group-icon">📁</span>
  {/if}
  {name}
</button>

<style>
  button {
    border: none;
    text-align: center;
    cursor: pointer;
    background: #666666 no-repeat center center / cover;
    color: #fff;
    border-radius: 5px;
    font-size: 1rem;
    padding: 0.5rem;
  }
  button.active {
    background-color: #3e8ed0;
  }
  button.isGroupChild {
    margin-left: 1rem;
  }
</style>
