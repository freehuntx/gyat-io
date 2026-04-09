<script lang="ts">
    import { onMount, tick } from 'svelte';
    import { decode, encode } from '@msgpack/msgpack';

    type Message = {
        id: number;
        time: string;
        server: string;
        sender: string;
        isPrivate: boolean;
        text: string;
    };

    let messages: Message[] = [];
    let nextId = 1;
    let listContainer: HTMLDivElement;

    let ws1Status = 'connecting';
    let ws2Status = 'connecting';

    let senderName = 'System';    
    let selectedServer = 'Server1';    
    let chatInput = '';

    let ws1Global: WebSocket;
    let ws2Global: WebSocket;

    async function addMessage(server: string, sender: string, text: string) {
        const isPrivate = text.indexOf(':') >= 0
        const time = new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', second: '2-digit' });

        messages = [...messages, { id: nextId++, time, server, sender, isPrivate, text }];
        
        // Keep only the last 1000 messages to prevent memory issues
        if (messages.length > 1000) {
            messages = messages.slice(messages.length - 1000);
        }

        try {
            localStorage.setItem('chatHistory', JSON.stringify(messages));
            localStorage.setItem('chatNextId', nextId.toString());
        } catch (e) {
            console.warn('Failed to save to localStorage', e);
        }

        await tick();
        if (listContainer) {
            listContainer.scrollTop = listContainer.scrollHeight;
        }
    }

    onMount(() => {
        try {
            const savedHistory = localStorage.getItem('chatHistory');
            const savedNextId = localStorage.getItem('chatNextId');
            if (savedHistory) {
                messages = JSON.parse(savedHistory);
            }
            if (savedNextId) {
                nextId = parseInt(savedNextId, 10);
            }
        } catch (e) {
            console.warn('Failed to load from localStorage', e);
        }

        tick().then(() => {
            if (listContainer) {
                listContainer.scrollTop = listContainer.scrollHeight;
            }
        });

        let isUnmounted = false;
        let ws1: WebSocket;
        let ws2: WebSocket;

        const handleMessage = (event: MessageEvent, server: string) => {
            if (event.data instanceof ArrayBuffer) {
                const buffer = new Uint8Array(event.data);
                if (buffer.length === 0) return;
                
                const opcode = buffer[0];
                if (opcode === 7) {
                    try {
                        const msgpackData = buffer.slice(1);
                        const data = decode(msgpackData) as any;
                        
                        if (data && data.Text && data.ID) {
							const [name, text] = data.Text.match(/^([^:]*): (.*)/).slice(1)

                            addMessage(server, name + '#' + data.ID, text);
                        }
                    } catch(e) {
                         console.error(`Invalid msgpack from ${server}:`, e);
                    }
                }
            }
        };

        const connectWs1 = () => {
            if (isUnmounted) return;
            ws1Status = 'connecting';
            ws1 = new WebSocket('wss://gyat.io/ws');
            ws1Global = ws1;
            ws1.binaryType = 'arraybuffer';
            ws1.addEventListener('open', () => ws1Status = 'connected');
            ws1.addEventListener('close', () => {
                ws1Status = 'closed';
                if (!isUnmounted) setTimeout(connectWs1, 3000);
            });
            ws1.addEventListener('error', () => {
                ws1Status = 'closed';
            });
            ws1.onmessage = (event) => handleMessage(event, 'Server1');
        };

        const connectWs2 = () => {
            if (isUnmounted) return;
            ws2Status = 'connecting';
            ws2 = new WebSocket('wss://gyat.io/ws2');
            ws2Global = ws2;
            ws2.binaryType = 'arraybuffer';
            ws2.addEventListener('open', () => ws2Status = 'connected');
            ws2.addEventListener('close', () => {
                ws2Status = 'closed';
                if (!isUnmounted) setTimeout(connectWs2, 3000);
            });
            ws2.addEventListener('error', () => {
                ws2Status = 'closed';
            });
            ws2.onmessage = (event) => handleMessage(event, 'Server2');
        };

        connectWs1();
        connectWs2();

        return () => {
            isUnmounted = true;
            if (ws1) ws1.close();
            if (ws2) ws2.close();
        };
    });

    function sendChatMessage(serverType: string, sender: string, text: string) {
        let wsTarget = serverType === 'Server1' ? ws1Global : ws2Global;
        if (wsTarget && wsTarget.readyState === WebSocket.OPEN) {
            wsTarget.send(new Uint8Array([21]));
            wsTarget.send(new Uint8Array([13, ...encode({ Name: sender + '; ' + text + '\0' })]));
            wsTarget.send(new Uint8Array([7, ...encode({ Text: ' ' })]));
            wsTarget.send(new Uint8Array([17]));
        } else {
            console.warn(`Cannot send message. ${serverType} is not connected.`);
        }
    }

    function handleSend() {
        if (!chatInput.trim()) return;
        
        sendChatMessage(selectedServer, senderName || 'Unknown', chatInput);
        addMessage(selectedServer, senderName || 'Unknown', chatInput);
        
        chatInput = '';
    }

    function handleKeydown(e: KeyboardEvent) {
        if (e.key === 'Enter') {
            e.preventDefault();
            handleSend();
        }
    }
</script>

<div class="chat-container">
    <div class="header">
        <h1>Chat History</h1>
        <div class="status-labels">
            <span class="status-label {ws1Status}">
                {#if ws1Status === 'connecting'}
                    <span class="spinner"></span>
                {/if}
                Server1
            </span>
            <span class="status-label {ws2Status}">
                {#if ws2Status === 'connecting'}
                    <span class="spinner"></span>
                {/if}
                Server2
            </span>
        </div>
    </div>
    <div class="messages-list" bind:this={listContainer}>
        {#each messages as msg (msg.id)}
            <div class="message {msg.isPrivate ? 'private' : ''}">
                <span class="time">[{msg.time}]</span>
                <span class="server">[{msg.server}]</span>
                {#if msg.isPrivate}
                    <span class="icon" title="Private Message">🔒</span>
                {/if}
                <span class="sender">{msg.sender}:</span>
                <span class="text">{msg.text}</span>
            </div>
        {/each}
        {#if messages.length === 0}
            <div class="empty">No messages yet...</div>
        {/if}
    </div>
    
    <div class="chat-input-bar">
        <select class="server-select" bind:value={selectedServer}>
            <option value="Server1">Server1</option>
            <option value="Server2">Server2</option>
        </select>
        <input 
            type="text" 
            class="sender-input" 
            placeholder="Name" 
            bind:value={senderName} 
            maxlength="20"
        />
        <input 
            type="text" 
            class="message-input" 
            placeholder="Type a message..." 
            bind:value={chatInput} 
            on:keydown={handleKeydown}
        />
        <button class="send-btn" on:click={handleSend}>Send</button>
    </div>
</div>

<style>
    :global(body) {
        margin: 0;
        padding: 0;
        height: 100vh;
        width: 100vw;
        overflow: hidden;
    }

    .chat-container {
        width: 100%;
        height: 100vh;
        font-family: system-ui, -apple-system, sans-serif;
        display: flex;
        flex-direction: column;
        background: #fff;
    }

    .header {
        display: flex;
        align-items: center;
        background: #333;
        padding: 1rem;
    }
    
    h1 {
        margin: 0;
        color: #fff;
        font-size: 1.2rem;
        flex-grow: 1;
        padding: 0;
        background: transparent;
    }

    .status-labels {
        display: flex;
        gap: 1rem;
    }

    .status-label {
        display: flex;
        align-items: center;
        gap: 0.5rem;
        padding: 0.25rem 0.75rem;
        border-radius: 12px;
        font-size: 0.85rem;
        font-weight: 600;
        color: #fff;
    }

    .status-label.connecting {
        background: #f39c12; /* yellow */
    }

    .status-label.connected {
        background: #2ecc71; /* green */
    }

    .status-label.closed {
        background: #e74c3c; /* red */
    }

    .spinner {
        width: 12px;
        height: 12px;
        border: 2px solid rgba(255,255,255,0.3);
        border-radius: 50%;
        border-top-color: #fff;
        animation: spin 1s ease-in-out infinite;
    }

    @keyframes spin {
        to { transform: rotate(360deg); }
    }
    
    .messages-list {
        flex-grow: 1;
        overflow-y: auto;
        padding: 1rem;
        display: flex;
        flex-direction: column;
        gap: 0.5rem;
        background-color: #f5f5f5;
    }
    
    .message {
        padding: 0.5rem;
        border-radius: 4px;
        background: white;
        box-shadow: 0 1px 2px rgba(0,0,0,0.05);
        display: flex;
        align-items: baseline;
        gap: 0.4rem;
        word-break: break-word;
    }
    
    .message.private {
        background-color: #fff3cd;
        border: 1px solid #ffe69c;
    }
    
    .time {
        color: #999;
        font-size: 0.8em;
        font-family: monospace;
    }

    .server {
        color: #666;
        font-size: 0.8em;
        font-weight: 500;
    }
    
    .icon {
        font-size: 0.9em;
    }
    
    .sender {
        font-weight: 600;
        color: #2c3e50;
    }
    
    .text {
        color: #333;
    }

    .empty {
        text-align: center;
        color: #888;
        font-style: italic;
        padding: 2rem;
    }

    .chat-input-bar {
        display: flex;
        padding: 1rem;
        background: #f9f9f9;
        border-top: 1px solid #e0e0e0;
        gap: 0.5rem;
        align-items: center;
    }

    .server-select {
        padding: 0.5rem;
        border: 1px solid #ccc;
        border-radius: 4px;
        font-size: 0.9em;
        background-color: white;
    }

    .sender-input {
        width: 100px;
        padding: 0.5rem;
        border: 1px solid #ccc;
        border-radius: 4px;
        font-size: 0.9em;
    }

    .message-input {
        flex-grow: 1;
        padding: 0.5rem;
        border: 1px solid #ccc;
        border-radius: 4px;
        font-size: 0.9em;
    }

    .send-btn {
        padding: 0.5rem 1rem;
        background-color: #3498db;
        border: none;
        border-radius: 4px;
        color: white;
        font-size: 0.9em;
        font-weight: bold;
        cursor: pointer;
        transition: background-color 0.2s;
    }

    .send-btn:hover {
        background-color: #2980b9;
    }
</style>