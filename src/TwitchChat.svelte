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



<style>
    .chat-container {
        position: relative;
        max-height: 200px;
        overflow-y: auto;
        background: rgb(30, 30, 30);
        opacity: 0.8;
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
        background-color: rgb(0, 123, 255);
        color: rgb(255, 255, 255);
        border: none;
        border-radius: 5px;
        cursor: pointer;
        opacity: 0.8;
        z-index: 1000;
        text-align: center;
    }

    .jump-to-bottom:hover {
        opacity: 1;
    }
</style>