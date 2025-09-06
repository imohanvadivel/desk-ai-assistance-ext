<script lang="ts">
    import { onMount, setContext } from "svelte";
    import Markdown from "svelte-markdown";
    import { cleanseConvData, getAllThreadDetails, initApp } from "../lib/util";

    let App;
    setContext("App", App);

    type Prompt = {
        role: "system" | "user" | "assistant";
        content: string;
    };

    let chatWindow: HTMLDivElement;
    let chatInput: HTMLInputElement;
    let currentPrompt = "";
    let session: any;
    let prompts: Prompt[] = [];

    onMount(async () => {
        App = await initApp();

        const handleKeydown = (e: KeyboardEvent) => {
            if (e.key === "Enter") {
                promptStreaming();
            }
        };

        chatInput.addEventListener("keydown", handleKeydown);

        const ticketData = await ZOHODESK.get("ticket");
        const { id: ticketId, subject, createdTime, contactName, email: contactEmail, channel } = ticketData.ticket;

        let threadDetails = await getAllThreadDetails(ticketId);
        let conversation = cleanseConvData(threadDetails);
        let data = {
            subject,
            createdTime,
            contactName,
            contactEmail,
            channel,
            conversation,
        };

        let systemPrompt = `Here is the details of the ticket \n ${JSON.stringify(data)}`;

        await createSession(systemPrompt);

        prompts = [
            {
                role: "assistant",
                content: "Hello, how can I help you with this ticket?",
            },
        ];

        return () => {
            chatInput.removeEventListener("keydown", handleKeydown);
        };
    });

    async function createSession(systemPrompt: string) {
        // @ts-ignore
        const availability = await self.LanguageModel.availability();

        if (availability === "unavailable") {
            console.error("LanguageModel is not available in your browser");
            return;
        }

        const initialPrompts = [
            {
                role: "system",
                content: systemPrompt,
            },
        ];

        function monitor(m: any) {
            m.addEventListener("downloadprogress", (e: any) => {
                console.log(`Downloading Language Model ${Math.round((e.loaded / e.total) * 100)}%`);
            });
        }

        try {
            if (availability === "available") {
                // @ts-ignore
                session = await self.LanguageModel.create({ initialPrompts });
            } else {
                // @ts-ignore
                session = await self.LanguageModel.create({
                    initialPrompts,
                    monitor,
                });
            }
        } catch (error) {
            console.log(error);
        }
    }

    async function promptStreaming() {
        if (!currentPrompt) return;

        prompts = [
            ...prompts,
            {
                role: "user",
                content: currentPrompt,
            },
            {
                role: "assistant",
                content: "",
            },
        ];

        const stream = session.promptStreaming(currentPrompt);
        currentPrompt = "";

        try {
            for await (const chunk of stream) {
                prompts[prompts.length - 1].content += chunk;

                chatWindow?.scrollTo({
                    top: chatWindow.scrollHeight,
                    behavior: "smooth",
                });
            }
        } catch (error) {
            console.error(error);
        }
    }
</script>

<main>
    <div bind:this={chatWindow} class="multiline-cnt chat-window">
        {#each prompts as prompt}
            <span class="role">{prompt.role}</span>
            <div class="message">
                <Markdown source={prompt.content} />
            </div>
        {/each}
    </div>

    <input
        class="prompt-input"
        spellcheck="false"
        type="text"
        bind:this={chatInput}
        id="text"
        bind:value={currentPrompt}
        placeholder="Enter prompt"
    />
</main>

<style>
    :global(html),
    :global(body),
    :global(#app) {
        height: 100%;
        width: 100%;
    }

    main {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        gap: 0.25rem;
        height: 100%;
    }

    .chat-window {
        width: 100%;
        height: 100%;
        flex: 1;
        overflow-y: auto;
        padding: 1rem;
    }

    .prompt-input {
        width: calc(100% - 2rem);
        margin-top: auto;
        margin: 1rem;
        border: 1px solid var(--db-color-border);
        font-size: 0.875rem;
        padding: 0.5rem;
        border-radius: 0.25rem;
        outline: none;
        background-color: var(--db-color-background);
        color: var(--db-color-text);
    }

    .prompt-input:focus {
        border-color: var(--db-color-border-brand-strong);
    }

    .role {
        font-size: 14px;
        color: var(--db-color-text-secondary);
        padding-bottom: 0.25rem;
        display: block;
        font-weight: 400;
    }

    .message {
        padding-bottom: 1.5rem;
        font-size: 14px;
        line-height: 1.3;
        color: var(--db-color-text);
        font-weight: 400;
    }

    .message :global(p) {
        line-height: 1.3;
        font-size: 14px;
        padding-bottom: 0.5rem;
    }

    .message :global(ul) {
        list-style-type: disc;
        padding: 0 1rem;
    }

    .message :global(li) {
        line-height: 1.3;
        padding-bottom: 0.5rem;
        font-size: 14px;
    }

    .message :global(li:last-child) {
        padding-bottom: 0;
    }

    .message :global(ol) {
        list-style-type: decimal;
        padding: 0 1rem;
    }

    .message :global(strong) {
        font-weight: 600;
    }

    .message :global(h2) {
        font-size: 16px;
        font-weight: 600;
    }

    .message :global(h3) {
        font-size: 14px;
        font-weight: 600;
    }

</style>
