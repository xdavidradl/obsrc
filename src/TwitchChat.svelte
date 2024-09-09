<script>
    import { Client } from 'tmi.js'
    import { tick } from 'svelte'

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
    })

    //function isScrolledToBottom() {
    //    return chatDiv.scrollHeight - chatDiv.scrollTop === chatDiv.clientHeight
    //}

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

<div bind:this={chatDiv} class="chat" on:scroll={handleScroll}>
    {#each messages as message}
        <p><strong style="color: {message.color}">{message.username}:</strong> {message.message}</p>
    {/each}
</div>

{#if showJumpButton}
    <button class="jump-to-bottom" on:click={scrollToBottom}>
        Jump to bottom
    </button>
{/if}

<style>
    .chat {
        max-height: 200px;
        overflow-y: scroll;
        background: #b1acac;
        color: black;
        padding: 10px;
        border-radius: 5px;
        font-family: Arial, sans-serif;
    }

    .chat p {
        margin: 5px 0;
        line-height: 1.2;
    }

    .chat::-webkit-scrollbar {
        display: none;
    }

    .chat {
        scrollbar-width: none;
    }

    .jump-to-bottom {
        /*position: fixed;
        bottom: 50px;
        right: 20px;*/
        padding: 10px 20px;
        background-color: #007bff;
        color: white;
        border: none;
        border-radius: 5px;
        cursor: pointer;
        z-index: 1000;
        display: block;
    }

    .jump-to-bottom:hover {
        background-color: #0056b3;
    }
</style>