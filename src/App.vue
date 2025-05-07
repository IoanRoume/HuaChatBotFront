<template>

  <div v-if="showWelcome" class="welcome-overlay">
    <div class="welcome-box">
      <img src="https://media.tenor.com/_CnW8qoaw1AAAAAi/happy-hello.gif" alt="Welcome" />
      
      <div class="welcome-text-box">
        <h3>Καλώς ήρθατε!</h3> 
        Είμαι εδώ για να σας βοηθήσω με κάθε απορία που έχετε σχετικά με το 
      <strong>Χαροκόπειο Πανεπιστήμιο</strong> και πιο συγκεκριμένα με το 
      <strong>Προπτυχιακό Πρόγραμμα Σπουδών του Τμήματος Πληροφορικής & Τηλεματικής</strong>.
        <p></p>Μη διστάσετε να με ρωτήσετε για προγράμματα σπουδών, μαθήματα, διαδικασίες εγγραφής ή οποιοδήποτε σχετικό θέμα σας ενδιαφέρει.
        <p class="note">
          ⏳ Σε περίπτωση καθυστέρησης στην απάντηση, ενδέχεται να οφείλεται σε αυξημένη επισκεψιμότητα — ευχαριστώ για την υπομονή σας!
        </p>
        
      </div>
      <button @click="startChat">Ξεκίνα την συνομιλία</button>
      <div class="do-not-show-again">
        <input type="checkbox" id="noShowWelcome" v-model="doNotShowWelcome">
        <label for="noShowWelcome">Να μην εμφανιστεί αυτό το μήνυμα ξανά</label>
      </div>
    </div>
  </div>
  
  
    <div id="chatContainer" class="expanded">
      <div class="chatHeader">
        <h4 class="botName"><a href="https://www.hua.gr/"><img src="../HUA-Logo-Informatics-Telematics-EN-RGB.png" class="harokopioImage"></a></h4>
        <h4 class="botName"><a href="https://dit.hua.gr/index.php/el/"><img src="../dit-logo.png" class="ditImage"></a></h4>
      </div>
      <div class="chatBody">
        <div class="messageRow bot">
      <div class="message bot">
        <p>🤖 Χρειάζεστε βοήθεια; Δοκιμάστε να ρωτήσετε για:</p>
        <div class="suggested-questions">
          <div v-for="(question, index) in suggestedQuestions" :key="index" 
              class="suggested-question" @click="selectQuestion(question)">
            {{ question }}
          </div>
        </div>
      </div>
    </div>
        <div class="messages" v-for="message in messages" :key="message.id">
          <div class="messageRow" :class="{ user: message.sender === 'user', bot: message.sender === 'bot' }">
            <div class="message" :class="message.sender">
              <div v-if="message.message === 'loading-indicator'" class="thinkingGif">
                <img src="https://media.tenor.com/KjlP2exHPCgAAAAi/cute-robot.gif" alt="Bot is thinking..." class="thinkingImage"> <!-- Bot is thinking -->
              </div>
              <p v-else v-html="formatMessage(message.message)"></p>
  
            </div>
          </div>
  
  
          <div v-if="message.sender === 'bot' && !message.feedbackGiven && message.generated" class="feedback" style="margin-left: 10px;">
            <span style="color: gray;">Was this answer helpful?</span>
            <button @click="giveFeedback(message.id, 'up')" class="thumbsUp">👍</button>
            <button @click="giveFeedback(message.id, 'down')" class="thumbsDown">👎</button>
          </div>
  
          
  
  
          <div v-if="message.feedbackGiven" class="feedbackMessage" style="margin-left: 10px;">
            <span v-if="message.feedback">Thank you for your feedback!</span>
          </div>
  
  
           <div v-if="message.feedbackGiven && !message.commentGiven" class="commentSection">
            <input v-model="message.commentText" placeholder="Leave a comment..." class="commentInput" />
            <button @click="submitComment(message.id)" class="submitComment">Submit</button>
          </div>
  
          <div v-if="message.commentGiven" class="commentMessage">
            <strong>User comment:</strong> {{ message.comment }}
          </div>
  </div>
      </div>
      <div class="chatFooter">
        <form @submit.prevent="sendMessage()">
          <input 
            v-model="messageContent" 
            id="createMessage" 
            placeholder="Type Here..." 
            class="inputBox" 
            :disabled="isWaiting"
          />
          <input type="submit" :disabled="isWaiting" style="background-color: #1982fc;"/>
        </form>
      </div>
    </div>
  </template>
  
  <script>
  import axios from "axios";
  import { ref, onMounted, onUnmounted } from "vue";
  
  export default {
    name: "App",
    setup() {
      const messages = ref([]),
            messageContent = ref(""),
            isWaiting = ref(false);
      const session_id = ref(generateSessionId());
      const showWelcome = ref(true);
      const doNotShowWelcome = ref(false); // New ref to control the checkbox state
  
      const allQuestions = ref([
        "Ποια μαθήματα προσφέρονται στο πρώτο εξάμηνο;",
        "Πώς μπορώ να επικοινωνήσω με τη γραμματεία;",
        "Πως γίνεται να κάνω εγγραφή στο εξάμηνο;",
        "Τι ώρα και ποια μέρα διεξάγεται το μάθημα της Τεχνητής Νοημοσύνης;",
        "Τι πρέπει να έχω για να πάρω πτυχίο;",
        "Πότε τελειώνει το Εαρινό εξάμηνο;",
        "Τι μαθήματα πρέπει να παρακολουθήσω για να γίνω μηχανικός Τεχνιτής Νοημοσύνης;",
        "Πότε ξεκινάει η εξεταστική του Εαρινού εξαμήνου;",
        "Δώσε μου πληροφορίες για την Πρακτική Άσκηση.",
        "Ποιο είναι το email και τηλέφωνο της γραμματείας;",
        "Ποια είναι η διαδικασία για την πτυχιακή εργασία;",
        "Που βρίσκεται το τμήμα;",
        "Ποια είναι τα μαθήματα του 5ου εξαμήνου;"
      ]);
      const suggestedQuestions = ref(getRandomQuestions());
  
      function getRandomQuestions() {
        const shuffled = [...allQuestions.value].sort(() => 0.5 - Math.random());
        return shuffled.slice(0, 3);
      }
  
      function selectQuestion(question) {
        messageContent.value = question;
        sendMessage();
      }
  
      function generateSessionId() {
        const id = Math.random().toString(36).substr(2, 16);
        return id;
      }
  
      function startChat() {
        if (doNotShowWelcome.value) {
          localStorage.setItem("hideWelcome", "true");
        }
        showWelcome.value = false;
      }
  
  
      async function handleSessionEnd() {
      const url = `http://localhost:9090/sessionEnd?feedback=${encodeURIComponent(JSON.stringify(extractFeedback()))}`;
          try {
              await fetch(url, { method: "GET", keepalive: true });
          } catch (error) {
              console.error("Error ending session:", error);
          }
      }
  
  
  function extractFeedback() {
      return messages.value
          .filter(msg => msg.feedbackGiven && !msg.commentGiven)
          .map(msg => ({
              question: messages.value[msg.id - 1]?.message || "Unknown",
              answer: msg.message,
              feedback: msg.feedback,
              comment: "No comment",
          }));
  }
  
  function scrollToBottom() {
      const chatBody = document.querySelector(".chatBody");
      if (chatBody) {
          setTimeout(() => {
              chatBody.scrollTop = chatBody.scrollHeight;
          }, 50); 
      }
  }
  
  function formatMessage(text) {
    return text
      .replace(/\*\*(.*?)\*\*/g, "<b>$1</b>")  
      .replace(/__(.*?)__/g, "<b>$1</b>")      
      .replace(/\n/g, "<br>")
      .replace(/\[at\]/gi, "@")        
      .replace(/\[dot\]/gi, ".")   
      .replace(/\\\[at\\\]/gi, "@")  
      .replace(/\\\[dot\\\]/gi, ".")
      .replace(/([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})/g, '<a href="mailto:$1">$1</a>')
      //.replace(/(https?:\/\/[^\s<]+)/g, '<a href="$1" target="_blank">$1</a>');
  }
  
      onMounted(() => {
        window.addEventListener("beforeunload", handleSessionEnd);
        const hideWelcome = localStorage.getItem("hideWelcome");
        if (hideWelcome === "true") {
          showWelcome.value = false;
        }
        
        // Add meta viewport tag for better mobile experience
        if (!document.querySelector('meta[name="viewport"]')) {
          const meta = document.createElement('meta');
          meta.name = 'viewport';
          meta.content = 'width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no';
          document.getElementsByTagName('head')[0].appendChild(meta);
        }
      });
  
      onUnmounted(() => {
        window.removeEventListener("beforeunload", handleSessionEnd);
      });
  
  
    async function sendMessage() {
      if (messageContent.value === "" || isWaiting.value) return;
  
      isWaiting.value = true;
      createMessage(messageContent.value, "user");
      scrollToBottom();
  
      const botMessageId = createMessage("loading-indicator", "bot");
  
  
      try {
        let continuePolling = true;
        let previousMessage = "";
  
        while (continuePolling) {
          const { data } = await axios.post("http://localhost:9090/chat", {
            message: messageContent.value,
            history: getLastPairs(),
            session_id: session_id.value
          });
  
          if (data.message.includes("Παρακαλώ περιμένετε")) {
            if (previousMessage != data.message) {
              animateTyping(botMessageId, data.message); // Show initial wait message
              previousMessage = data.message;
            }
            await new Promise((resolve) => setTimeout(resolve, 1000)); // Wait 1 second before polling again
          } else {
            animateTyping(botMessageId, data.message);
            continuePolling = false;
          }
        }
      } catch (error) {
        console.error("Error sending message:", error);
        animateTyping(botMessageId, "Error: Could not reach the server.");
      }
  
      messageContent.value = "";
      isWaiting.value = false;
      scrollToBottom();
  }
  
  
      function getLastPairs() {
      let lastMessages = messages.value.slice(-6); 
  
      let filteredMessages = lastMessages.filter(msg => !(msg.sender === "bot" && msg.message === "..."));
  
      if (messages.value.length <= 2 && messages.value[0].sender === "user") {
          return [];
      }
  
      filteredMessages.pop();
  
      return filteredMessages.map(msg => ({
          role: msg.sender === "user" ? "user" : "bot",
          content: msg.message
      }));
  }
  
      function createMessage(text, sender) {
        let id = messages.value.length;
        messages.value.push({ 
          id, 
          message: text, 
          sender, 
          generated: false
        });
        return id; 
      }
  
      function animateTyping(messageId, fullText) {
      if (!messages.value[messageId]) {
          console.error(`animateTyping: Message with ID ${messageId} not found.`);
          return;
      }
  
      if (typeof fullText !== "string") {
          console.error("animateTyping: fullText is not a valid string.");
          messages.value[messageId].message = "There was an error, please try again.";
          messages.value[messageId].generated = true;
          return;
      }
  
      fullText = fullText.replace("🤖", "");
      const emoji = "🤖 ";
      fullText = emoji + fullText;
  
      const delayPerChar = fullText.includes("Παρακαλώ περιμένετε") ? 0 : 20; // ms per character
  
      let startTime = performance.now();
  
      function updateFrame(now) {
          const elapsed = now - startTime;
          const index = Math.floor(elapsed / delayPerChar);
  
          if (index <= fullText.length) {
              messages.value[messageId].message = fullText.substring(0, index);

              requestAnimationFrame(updateFrame);
          } else {
              messages.value[messageId].message = fullText;
              messages.value[messageId].generated = fullText.includes("Παρακαλώ περιμένετε") || fullText.includes("Error: Could not reach the server.") ? false : true;
              scrollToBottom();
          }
      }
  
      requestAnimationFrame(updateFrame);
  }
  
  
      function giveFeedback(messageId, feedbackType) {
        const message = messages.value.find(msg => msg.id === messageId);
        if (message) {
          message.feedback = feedbackType;
          message.feedbackGiven = true;
        }
      }
  
      async function submitComment(messageId) {
        const message = messages.value.find(msg => msg.id === messageId);
        if (message) {
          if (!message.commentText || message.commentText.trim() === "") {
            alert("Please enter a comment before submitting.");
            return;
          }
          message.comment = message.commentText;
          message.commentGiven = true;
  
          try {
            await axios.post("http://localhost:9090/feedback", {
              question: messages.value[messageId - 1]?.message || "Unknown",
              answer: message.message,
              feedback: message.feedback,
              comment: message.commentText,
            });
          } catch (error) {
            console.error("Error sending feedback:", error);
          }
        }
      }
  
  
      return { messages, messageContent, sendMessage, isWaiting , giveFeedback, submitComment, formatMessage, startChat, showWelcome, suggestedQuestions, getRandomQuestions, selectQuestion, doNotShowWelcome};
    },
  };
  </script>
  
  <style>
  /* Base styles and reset */
  * {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }
  
  html {
    font-size: 16px;
    touch-action: manipulation;
  }
  
  body {
    margin: 0;
    font-family: Avenir, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
    width: 100vw;
    overflow: hidden;
  }
  
  .message p {
    white-space: pre-wrap;
    word-break: break-word;
  }
  
  /* Chat container */
  #chatContainer {
    background-color: white;
    height: 100vh; 
    width: 100vw; 
    margin: 0;
    padding: 0;
    display: flex;
    flex-direction: column;
    position: relative;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  }
  
  #chatContainer.expanded .chatBody {
    height: calc(100% - 100px); 
  }
  
  /* Header */
  .chatHeader {
    box-shadow: 0 4px 8px 0 rgba(0, 0, 0, 0.2);
    color: black;
    display: flex;
    justify-content: space-around;
    padding: 5px;
    align-items: center;
    flex-wrap: wrap;
    min-height: 60px;
  }
  
  .botName {
    margin: 0;
    padding: 5px;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  
  .harokopioImage {
    height: 150px;
    max-width: 100%;
  }
  
  .ditImage {
    height: 80px;
    max-width: 100%;
  }
  
  /* Footer */
  .chatFooter {
    position: fixed; 
    bottom: 0;
    left: 0;
    width: 100%;
    background: white;
    box-shadow: 0 -2px 4px rgba(0, 0, 0, 0.1);
    z-index: 10;
    padding: 8px 0;
  }
  
  .chatFooter form {
    display: flex;
    align-items: center;
    justify-content: space-between;
    width: 95%;
    margin: 0 auto;
    max-width: 100%;
  }
  
  /* Chat body */
  .chatBody {
    overflow-y: auto; 
    height: calc(100% - 130px); 
    position: relative;
    padding-bottom: 60px; 
    -webkit-overflow-scrolling: touch;
  }
  
  .chatBody::before {
    content: "";
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    width: 60%; 
    height: 60%;
    background-image: url('https://huadev.hua.gr/wp-content/uploads/2024/04/cropped-cropped-cropped-HUA-Logo-Blue-RGB-2-1.png');
    background-size: contain;
    background-position: center;
    background-repeat: no-repeat;
    opacity: 0.1; 
    z-index: 0;
    pointer-events: none;
  }
  
  /* Input area */
  #createMessage {
    width: 80%;
    min-height: 40px;
    font-size: 16px; /* Larger text for mobile */
  }
  
  input[type="submit"] {
    height: 40px;
    width: 18%;
    border-radius: 20px;
    border: 0;
    color: white;
    padding: 8px;
    font-size: 16px;
    margin-bottom: 0;
    opacity: 1;
  }
  
  input:not(#createMessage) {
    border-radius: 20px;
    border: 0;
    color: white;
    padding: 8px;
    margin-bottom: 0;
    opacity: 1;
  }
  
  input:not(#createMessage):hover {
    opacity: 0.9;
  }
  
  .inputBox {
    padding: 10px;
    border-radius: 20px;
    border: 1px solid black !important;
    color: black !important;
    font-size: 16px;
  }
  
  .inputBox::placeholder {
    color: black;
    opacity: 1; 
  }
  
  /* Messages */
  .messageRow {
    display: flex;
    margin: 8px;
  }
  
  .messageRow.user {
    justify-content: flex-end;
  }
  
  .messageRow.bot {
    justify-content: flex-start;
  }
  
  .message {
    opacity: 1;
    max-width: 85%; /* Increased from 60% for better mobile display */
    padding: 10px 15px;
    border-radius: 15px;
    text-align: left;
    word-wrap: break-word;
    z-index: 1;
    font-size: 16px;
  }
  
  .message.user {
    background-color: #1982fc;
    color: white;
  }
  
  .message.bot {
    background-color: #f7fcf7;
    color: black;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  }
  
  /* Feedback section */
  .feedback {
    display: flex;
    align-items: center;
    margin-top: 5px;
    font-size: 14px;
    flex-wrap: wrap;
  }
  
  .feedback button {
    background: none;
    border: none;
    cursor: pointer;
    font-size: 22px; /* Larger for easier tapping */
    margin-left: 8px;
    transition: transform 0.2s;
    padding: 5px;
  }
  
  .feedback .thumbsUp:hover, .feedback .thumbsUp:active {
    color: green;
    transform: scale(1.2);
  }
  
  .feedback .thumbsDown:hover, .feedback .thumbsDown:active {
    color: red;
    transform: scale(1.2);
  }
  
  .feedbackMessage {
    font-size: 14px;
    margin-top: 5px;
    color: gray;
  }
  
  /* Comment section */
  .commentSection {
    display: flex;
    margin-top: 5px;
    margin-left: 10px;
    align-items: center;
    flex-wrap: wrap;
  }
  
  .commentInput {
    flex: 1;
    padding: 8px;
    border: 1px solid black !important;
    color: black !important;
    border-radius: 5px;
    max-width: 70%;
    outline: none; 
    font-size: 16px;
  }
  
  .submitComment {
    margin-left: 5px;
    padding: 8px 10px;
    color: black;
    border: none;
    border-radius: 20px;
    cursor: pointer;
    width: auto;
    height: auto;
    min-height: 36px;
    min-width: 75px;
    align-items: center;
    font-size: 16px;
  }
  
  .submitComment:hover, .submitComment:active {
    background-color: #1982fc;
    color: white;
  }
  
  .commentMessage {
    font-size: 14px;
    margin-top: 5px;
    color: grey;
    margin-left: 10px;
    word-wrap: break-word; 
    white-space: pre-wrap; 
    max-width: 80%; 
  }
  
  /* Bot thinking animation */
  .thinkingGif {
    display: flex;
    justify-content: flex-start;
    align-items: center;
    padding: 10px;
  }
  
  .thinkingImage {
    width: 80px;
    height: auto;
  }
  
  /* Welcome modal */
  .welcome-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0, 0, 0, 0.4);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1000;
  }
  
  .welcome-box {
    background: white;
    padding: 20px 15px;
    border-radius: 12px;
    box-shadow: 0 8px 24px rgba(0,0,0,0.2);
    text-align: center;
    width: 90%;
    max-width: 450px;
    max-height: 90vh;       
    overflow-y: auto;
  }
  
  .welcome-box img {
    max-width: 150px;
    margin-bottom: 0px;
  }
  
  .welcome-text-box {
    margin-top: 0px;
    padding-top: 0px;
    background-color: #f9f9f9;
    padding: 15px;
    border-radius: 10px;
    box-shadow: 0 15px 10px rgba(0, 0, 0, 0.1);
    font-size: 0.95rem;
    line-height: 1.5;
    color: #333;
    text-align: center;
    margin-bottom: 20px;
    max-height: 50vh;      
    overflow-y: auto;
  }
  
  .welcome-box h3 {
    font-size: 1.3rem;
    margin-bottom: 10px;
  }
  
  .welcome-box button {
    padding: 10px 20px;
    font-size: 1rem;
    background-color: #005bbb;
    color: white;
    border: none;
    border-radius: 8px;
    cursor: pointer;
    width: 100%;
    max-width: 250px;
  }
  
  .note {
    font-size: 0.9em;
    color: #777;
    margin-top: 1em;
    font-style: italic;
  }
  
  .do-not-show-again {
    font-size: 0.85rem;
    color: grey;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 5px;
    margin-top: 10px;
  }
  
  .do-not-show-again input[type="checkbox"] {
    width: 16px;
    height: 16px;
  }
  
  /* Suggested questions */
  .suggested-questions {
    display: flex;
    flex-direction: column;
    gap: 8px;
    margin-top: 10px;
  }
  
  .suggested-question {
    background-color: #e8f4fc;
    border: 1px solid #cce5ff;
    border-radius: 10px;
    padding: 10px 12px;
    font-size: 0.95rem;
    color: #0066cc;
    cursor: pointer;
    transition: all 0.2s ease;
    text-align: left;
  }
  
  .suggested-question:hover, .suggested-question:active {
    background-color: #d0e8ff;
    transform: translateY(-2px);
  }
  
  /* Media Queries */
  @media (max-width: 768px) {
    .message {
      max-width: 85%;
      font-size: 16px;
    }
    
    .welcome-box {
      width: 95%;
      padding: 15px 10px;
    }
    
    .welcome-text-box {
      font-size: 0.9rem;
      padding: 12px;
    }
    
    .chatHeader {
      padding: 5px 0;
    }
    
    .harokopioImage {
      height: 40px;
    }
    
    .ditImage {
      height: 32px;
    }
    
    .commentInput {
      max-width: 65%;
    }
    
    .feedback {
      font-size: 13px;
    }
    
    .commentMessage {
      max-width: 85%;
    }
  }
  
  @media (max-width: 480px) {
    html {
      font-size: 14px;
    }
    
    .message {
      max-width: 90%;
      padding: 8px 12px;
    }
    
    .chatFooter form {
      width: 98%;
    }
    
    .welcome-box img {
      max-width: 120px;
    }
    
    .harokopioImage {
      height: 35px;
    }
    
    .ditImage {
      height: 28px;
    }
    
    .commentInput {
      max-width: 60%;
    }
    
    input[type="submit"] {
      width: 20%;
    }
    
    .thinkingImage {
      width: 60px;
    }
    
    .suggested-question {
      padding: 8px 10px;
    }
  }
  
  /* Fix iOS input zoom */
  @media screen and (-webkit-min-device-pixel-ratio:0) { 
    select,
    textarea,
    input[type="text"],
    input[type="password"],
    input[type="datetime"],
    input[type="datetime-local"],
    input[type="date"],
    input[type="month"],
    input[type="time"],
    input[type="week"],
    input[type="number"],
    input[type="email"],
    input[type="url"],
    input[type="search"],
    input[type="tel"],
    input[type="color"] {
      font-size: 16px;
    }
  }
  
  /* Touch-friendly buttons */
  button {
    min-height: 44px;
    min-width: 44px;
  }
  
  a {
    min-height: 44px;
    min-width: 44px;
    display: inline-block;
  }
  
  /* Fix for iOS Safari */
  @supports (-webkit-touch-callout: none) {
    #chatContainer {
      height: -webkit-fill-available;
    }
    
    .chatBody {
      height: calc(100vh - 130px);
      height: calc(-webkit-fill-available - 130px);
    }
  }
  </style>