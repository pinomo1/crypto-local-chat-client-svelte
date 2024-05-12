<script lang="ts">

  
  document.title = 'Chat ';
  import { onMount, afterUpdate } from 'svelte';
  import { io } from 'socket.io-client';
  import { MiniRSA } from '../crypto/miniRSA';
  import { CaesarFunction } from '../crypto/caesar_function';

  const localStorageKey = 'ipAddress';
  const url = localStorage.getItem(localStorageKey);
  const wsUrl = `ws://${url}:8002`;
  const token = localStorage.getItem('token');
  const rsa = new MiniRSA();
  const caesar = new CaesarFunction();
  let messages: { username: string; message: string }[] = [];
  let message = '';
  let chatContainer: HTMLElement | null = null;
  let sessionKey: string;

  let ws = io(wsUrl, { transports : ['websocket'] });
  ws.on("connect", handleOpen);
  ws.on("chat", handleAnonMessage);
  ws.on("joined", handleJoin)
  ws.on("left", handleLeft)
  ws.on("error", handleError)
  ws.on("service", handleService)
  
  function handleOpen() {
    ws.emit("join", token);
    sessionKey = localStorage.getItem('sessionKey') || '';
    rsa.loadKey(JSON.parse(localStorage.getItem('rsa') || ''));
    let isWaiting = localStorage.getItem('waiting');
    if (isWaiting === "false") {
      let encryptedB = localStorage.getItem('encryptedB');
      ws.emit("service", "handshake", encryptedB);
      caesar.loadKey(parseInt(sessionKey));
      console.log(caesar.key);
    }
  }

  function handleService(type: string, message: string) {
    if (type === "handshake") {
      let decrypted = rsa.decrypt(message);
      let obj = JSON.parse(decrypted);
      sessionKey = obj.sessionKey;
      localStorage.setItem('sessionKey', sessionKey);
      caesar.loadKey(parseInt(sessionKey));
      console.log(caesar.key);
      let response = "ping"
      let encrypted = caesar.encrypt(response);
      ws.emit("service", "response", encrypted);
    }
    else if (type === "response") {
      let decrypted = caesar.decrypt(message);
      let response = "pong"
      let encrypted = caesar.encrypt(response);
      if (decrypted === "ping") {
        ws.emit("service", "final", encrypted);
      }
      else {
        prompt(decrypted);
        handleLogout();
      }
    }
    else if (type === "final") {
      let decrypted = caesar.decrypt(message);
      if (decrypted !== "pong") {
        handleLogout();
      }
    }
  }

  function handleError(error: string) {
    window.location.href = '/login';
  }

  function handleJoin(room: string) {
    handleMessage("System", `Anonymous joined the chat`);
  }

  function handleLeft(room: string) {
    handleMessage("System", `Anonymous left the chat`);
  }

  function handleAnonMessage(message: string){
    let decrypted = caesar.decrypt(message);
    handleMessage("Anonymous", decrypted);
  }

  function handleOwnMessage(message: string){
    handleMessage("You", message);
  }

  function handleMessage(username: string, message: string) {
    let chatMessage = { username, message };
    if (messages.length > 1024) {
      messages = messages.slice(256);
    }
    messages = [...messages, chatMessage];
  }

  function handleSend() {
    let encrypted = caesar.encrypt(message);
    ws.emit("chat", encrypted);
    handleOwnMessage(message);
    message = '';
  }

  function handleLogout() {
    // ws.emit("logout");
    localStorage.removeItem('token');
    handleDisconnect();
  }

  function handleDisconnect() {
    window.location.href = '/login';
  }

  function handleKeyPress(event: KeyboardEvent) {
    if (event.key === 'Enter' && !event.shiftKey) {
      event.preventDefault(); // Prevent the Enter key from adding a new line
      scrollToBottom();
      handleSend();
    }
  }

  function scrollToBottom(force: boolean = false) {
    if (chatContainer) {
      if (chatContainer.scrollHeight - chatContainer.scrollTop < 1000 || force) {
        chatContainer.scrollTop = chatContainer.scrollHeight;
      }
    }
  }

  let isHovered = false;

function showHoverPanel() {
  isHovered = true;
}

function hideHoverPanel() {
  isHovered = false;
}

  afterUpdate(scrollToBottom);
</script>
  
  <div class="chat-container">
    <div class="chat-header">
      <div class="chat-header-text">Global Chat - {url}</div>
      <div class="logout-btn">
        <a href="#top" on:click={handleLogout}>Logout
          <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" class="bi bi-box-arrow-right" viewBox="0 0 16 16">
            <path fill-rule="evenodd" d="M10 12.5a.5.5 0 0 1-.5.5h-8a.5.5 0 0 1-.5-.5v-9a.5.5 0 0 1 .5-.5h8a.5.5 0 0 1 .5.5v2a.5.5 0 0 0 1 0v-2A1.5 1.5 0 0 0 9.5 2h-8A1.5 1.5 0 0 0 0 3.5v9A1.5 1.5 0 0 0 1.5 14h8a1.5 1.5 0 0 0 1.5-1.5v-2a.5.5 0 0 0-1 0v2z"/>
            <path fill-rule="evenodd" d="M15.854 8.354a.5.5 0 0 0 0-.708l-3-3a.5.5 0 0 0-.708.708L14.293 7.5H5.5a.5.5 0 0 0 0 1h8.793l-2.147 2.146a.5.5 0 0 0 .708.708l3-3z"/>
          </svg>
        </a>
      </div>
    </div>
    <div class="chat-messages" bind:this={chatContainer}>
      {#each messages as chatMessage, index (index)}
        {#if index === 0 || chatMessage.username !== messages[index - 1].username}
          <div class="message" key={index} role="button" tabindex="0" aria-label="Message Options" on:mouseenter={showHoverPanel} >
              <div class="message-content">
                <div class="sender-name">{chatMessage.username}</div>
              <div class="message-text" style="white-space: pre-line;">{@html chatMessage.message.replace(/\n/g, "<br>")}</div>
            </div>
        </div>
            
        {:else}
        
        <div style="padding: 3px; margin: 3px; padding-left: 2vh;" class="message" key={index} role="button" tabindex="0" aria-label="Message Options" on:mouseenter={showHoverPanel} >
          <div class="message-text" style="white-space: pre-line;">{@html chatMessage.message.replace(/\n/g, "<br>")}</div>
        </div>
        {/if}
      {/each}
    </div>
    <div class="input-container-container">
      
      <div class="input-container">
        <textarea rows="2" placeholder="Type a message..." bind:value={message} on:keydown={handleKeyPress}></textarea>
        <a class="send-btn" href="#top" on:click={handleSend}>
          <svg
            xmlns="http://www.w3.org/2000/svg"
            width="16"
            height="16"
            fill="currentColor"
            class="bi bi-send-fill"
            viewBox="0 0 16 16"
          >
            <path
              d="M15.964.686a.5.5 0 0 0-.65-.65L.767 5.855H.766l-.452.18a.5.5 0 0 0-.082.887l.41.26.001.002 4.995 3.178 3.178 4.995.002.002.26.41a.5.5 0 0 0 .886-.083l6-15Zm-1.833 1.89L6.637 10.07l-.215-.338a.5.5 0 0 0-.154-.154l-.338-.215 7.494-7.494 1.178-.471-.47 1.178Z"
            ></path>
          </svg>
        </a>
      </div>
    </div>
  </div>
  
  <style>
    /* Discord-like theme */
    .chat-container {
      max-width: 100%;
      margin: 0 auto;
      display: flex;
      flex-direction: column;
      background-color: #36393f;
      color: white;
      height: 100vh;
      overflow: hidden; /* Disable any scrollbars outside the chat container */
    }
  
    .chat-messages {
      flex: 1;
      border: 0px solid #ccc;
      /* padding: 10px; */
      overflow-y: auto;
      overflow-x: hidden; /* Disable horizontal scroll */
      background-color: transparent;
      scrollbar-width: thin;
      scrollbar-color: #202225 #202225; /* Scrollbar color and track color */
      padding-bottom: 5vh;
    }
  
    .chat-messages::-webkit-scrollbar {
      width: 8px;
      border-radius: 8px; /* Rounded corners */
    }
  
    .chat-messages::-webkit-scrollbar-thumb {
      background-color: #585d63; /* Thumb color */
      border-radius: 8px; /* Rounded corners */
    }
  
    .message {
      margin: 5px 0;
      padding: 5px;
      padding-left: 2vh;
      margin-left: 0 !important;
      display: flex;
      align-items: center;
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    }
  
    .message:hover {
      background: rgb(2, 0, 36);
      background: linear-gradient(90deg, rgba(2, 0, 36, 1) 0%, rgba(42, 42, 42, 1) 0%, rgba(0, 212, 255, 0) 50%);
    }
    .message-text {
        white-space: pre-wrap;
        overflow-wrap: break-word;
        word-break: break-all;
    }
  
    .message-content {
      text-align: left;
      word-wrap: break-word; /* Wrap long words when resizing */
    }
  
    .sender-name {
      font-weight: bold;
    }
  
    .input-container-container {
      padding: 10px;
      padding-top: 0;
      /* display: flex; */
      justify-content: center;
      align-items: center;
      background-color: #36393f;
      box-shadow: #36393f 0px 0px 100px 0.5px;
      /* max-width: 100vh; */
    }
  
    .input-container {
      padding: 10px;
      padding-top: 0;
      display: flex;
      justify-content: space-between;
      align-items: center;
      background-color: #36393f;
      box-shadow: #36393f 0px 0px 100px 0.5px;
    }
  
    textarea {
      flex: 1;
      padding: 5px;
      padding-top: 6px;
      resize: none;
      background-color: #40444b;
      color: white;
      border: none;
      outline: none;
      border-radius: 5px 0 0 5px;
    }
  
    textarea:focus {
      outline: none;
    }
  
    .send-btn {
      cursor: pointer;
      padding-top: 10px;
      padding-bottom: 9px;
      padding-right: 15px;
      background-color: #40444b;
      border: none;
      color: white;
      text-decoration: none;
      border-radius: 0 5px 5px 0;
      outline: none;
    }
  
    .send-btn svg {
      width: 20px;
      height: 20px;
      display: block;
      rotate: 25deg;
      padding-bottom: 3px;
    }
  
    .send-btn svg:hover {
      rotate: 40deg;
      transition: all 0.2s ease-in-out;
    }
  
    .send-btn:hover {
      text-decoration: none;
      color: white;
      background-color: #202225;
      padding-left: 10px;
      transition: all 0.2s ease-in-out;
    }
    .send-btn:active {
      background-color: #202225;
      transform: scale(0.9);
      transition: all 0.2s ease-in-out;
    }
    .chat-header {
      background-color: #313236;
      display: flex;
      justify-content: space-between;
      padding: 10px;
      box-shadow: #36393f 0px 0px 100px 0.5px;
      /* background: transparent; */
      /* backdrop-filter: blur(20px); */
    }
    
  
    .logout-btn {
      float: right;
      padding: 5px;
      padding-left: 10px;
      padding-right: 10px;
      border-radius: 5px;
      background-color: #40444b;
      color: white;
      text-decoration: none;
      transition: all 0.2s ease-in-out;
      cursor: pointer;
    }
    .logout-btn:hover {
      background-color: #202225;
      transition: all 0.2s ease-in-out;
    }
    .logout-btn:active {
      background-color: #202225;
      transform: scale(0.9);
      transition: all 0.2s ease-in-out;
    }
      
    
    .logout-btn a{
      color: white;
      text-decoration: none;
    }
    .logout-btn a svg{
      color: white;
      text-decoration: none;
      position: relative;
      top: 3px;
      left: 4px;
    }

    .sender-avatar {
      margin-right: 10px;
      border-radius: 50%; 
      overflow: hidden;
      width: 32px;
      height: 32px;
    }
  
    .sender-avatar img {
      width: 100%;
      height: 100%;
      object-fit: cover;
    }
    
    /* Hover panel styles */
.hover-panel {
  position: relative;
  top: -20px; 
  right: -50px; 
  /* background-color: rgba(0, 0, 0, 0.3); */
  border-radius: 5px;
  display: flex;
  gap: 5px;
  padding: 5px;
  margin: 0;
  z-index: 100 !important;
  overflow: visible;
}

.hover-panel button {
  background-color: #40444b;
  border: none;
  color: white;
  padding: 2px 6px;
  border-radius: 3px;
  cursor: pointer;
  overflow:visible;
  z-index: 10000 !important;
}

.hover-panel button:hover {
  background-color: #202225;
}

/* Show/hide hover panel on hover */
.message:hover .hover-panel {
  display: flex
}

/* Hide the hover panel by default */
.hover-panel {
  display: none;
}
  </style>
  
  
  
  
  