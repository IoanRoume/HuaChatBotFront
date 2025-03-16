<template>
  <div id="chatContainer" class="expanded">
    <div class="chatHeader">
      <h4 class="botName"><a href="https://www.hua.gr/"><img src="https://applied.dit.hua.gr/wp-content/uploads/2022/05/HUA_Logo_Blue.png" class="harokopioImage"></a></h4>
      <h4 class="botName"><a href="https://dit.hua.gr/index.php/el/"><img src="../dit-logo.png" class="ditImage"></a></h4>
    </div>
    <div class="chatBody">
      <div class="messageRow bot">
    <div class="message bot">
      <p>🤖 Καλώς ήρθατε!  </p>
      <p> Είμαι εδώ για να σας βοηθήσω με οποιαδήποτε απορία έχετε σχετικά με το <b>Χαροκόπειο Πανεπιστήμιο</b>  και το <b>Τμήμα Πληροφορικής & Τηλεματικής</b>. Ρωτήστε με για πληροφορίες σχετικά με προγράμματα σπουδών, μαθήματα, διαδικασίες εγγραφής και πολλά άλλα. Πώς μπορώ να σας βοηθήσω σήμερα;
      </p>
    </div>
  </div>
      <div class="messages" v-for="message in messages" :key="message.id">
        <div class="messageRow" :class="{ user: message.sender === 'user', bot: message.sender === 'bot' }">
          <div class="message" :class="message.sender">
            <p v-html="formatMessage(message.message)"></p>

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
    .replace("\\[at\\]","@")
    .replace("\\[dot\\]",".");             
}

    onMounted(() => {
      window.addEventListener("beforeunload", handleSessionEnd);
    });

    onUnmounted(() => {
      window.removeEventListener("beforeunload", handleSessionEnd);
    });


    async function sendMessage() {
      if (messageContent.value === "" || isWaiting.value) return;

      isWaiting.value = true; 
      createMessage(messageContent.value, "user"); 
      scrollToBottom();

      const botMessageId = createMessage("...", "bot");

      

      try {
        const { data } = await axios.post("http://localhost:9090/chat", { message: messageContent.value, history:getLastPairs() });

        
        animateTyping(botMessageId, data.message);
      } catch (error) {
        console.error("Error sending message:", error);
        animateTyping(botMessageId, "Error: Could not reach the server");
      }

      messageContent.value = ""; 
      isWaiting.value = false; 

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
      messages.value.push({ id, message: text, sender , generated: false});
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

    let index = 0;
    const emoji = "🤖 ";  
    fullText = emoji + fullText;
    const interval = setInterval(() => {
        if (index <= fullText.length) {
            messages.value[messageId].message = fullText.substring(0, index);
            index++;
        } else {
            clearInterval(interval);
            messages.value[messageId].generated = true;
        }
        scrollToBottom();
    }, 20);
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


    return { messages, messageContent, sendMessage, isWaiting , giveFeedback, submitComment, formatMessage};
  },
};
</script>

<style>
body {
  margin: 0;
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
}

.message p {
  white-space: pre-wrap;
}


#chatContainer {
  background-color: white;
  height: 90vh; 
  width: 90vw; 
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
.chatHeader {
  box-shadow: 0 4px 8px 0 rgba(0, 0, 0, 0.2);
  color: black;
  display: flex;
  justify-content: space-between;
  padding: 10px;
  align-items: center;
}
.chatFooter {
  position: fixed; 
  bottom: 0;
  left: 50%;
  transform: translateX(-50%);
  width: 90vw;
  background: white;
  box-shadow: 0 -2px 4px rgba(0, 0, 0, 0.1);
  z-index: 10;
}
.chatFooter form {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 95%;
  margin: 0 auto;
}
.chatBody {
  overflow-y: auto; 
  height: calc(100% - 80px); 
  position: relative;
  padding-bottom: 60px; 
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
#createMessage {
  width: 80%;
}
input:not(#createMessage) {
  margin-top: 10px;
  border-radius: 20px;
  border: 0;
  color: white;
  padding: 10px;
  margin-bottom: 12px;
  opacity: 1;
}
input:not(#createMessage):hover {
  opacity: 0.5;
}
.messageRow {
  display: flex;
  margin: 10px;
}

.botName {
  margin: 0;
  display: flex;
  align-items: center;
}

.harokopioImage {
  height: 100px; /* Adjust size as needed */
}

.ditImage {
  height: 80px; /* Adjust size as needed */
}


.messageRow.user {
  justify-content: flex-end;
}

.messageRow.bot {
  justify-content: flex-start;
}

.message {
  opacity: 1;
  max-width: 60%;
  padding: 10px 15px;
  border-radius: 15px;
  text-align: left;
  word-wrap: break-word;
  z-index: 1;
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

.chatBody::-webkit-scrollbar {
  width: 0px;
  height: 100%;
}

.botName {
  margin: 0;
  padding: 10px;
}

.inputBox {
  width: 80%;
  padding: 10px;
  border-radius: 20px;
  border: 0;
  margin: 10px;
  margin-left: 0;
  size: 40px;
  border: 1px solid black !important;
  color: black !important;
}

.inputBox::placeholder {
  color: black;
  opacity: 1; 
}

.feedback {
  display: flex;
  align-items: center;
  margin-top: 5px;
  font-size: 14px;
}

.feedback button {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 18px;
  margin-left: 8px;
  transition: transform 0.2s;
}

.feedback .thumbsUp:hover {
  color: green;
  transform: scale(1.2);
}

.feedback .thumbsDown:hover {
  color: red;
  transform: scale(1.2);
}

.feedbackMessage {
  font-size: 14px;
  margin-top: 5px;
  color: gray;
}



.commentSection {
  display: flex;
  margin-top: 5px;
  margin-left: 10px;
  align-items: center;
}

.commentInput {
  flex: 1;
  padding: 5px;
  border: 1px solid black !important;
  color: black !important;
  border-radius: 5px;
  max-width: 250px;
  outline: none; 
}

.commentInput:focus, 
.commentInput:active {
  border: 1px solid black !important; 
  box-shadow: none; 
}

.commentInput::placeholder {
  color: black;
  opacity: 1; 
}

.submitComment {
  margin-left: 5px;
  padding: 5px 10px;
  color: black;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  width: 75px;
  height: 30px;
  border-radius: 20px;
  align-items: center;
}

.submitComment:hover {
  background-color: #1982fc;
  color: white;
}

.feedbackMessage {
  font-size: 14px;
  margin-top: 5px;
  color: gray;
}

.commentMessage {
  font-size: 14px;
  margin-top: 5px;
  color: grey;
  margin-left: 10px;
  word-wrap: break-word; 
  white-space: pre-wrap; 
  max-width: 50%; 
}

</style>