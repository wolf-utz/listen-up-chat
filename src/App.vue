<script setup lang="ts">
import { ref, nextTick, watch } from "vue";
import AudioPlayer from "./components/AudioPlayer.vue";

interface Message {
  id: number | string;
  text: string;
  sent: boolean;
  timestamp: Date;
  type?: "text" | "story" | "question" | "evaluation" | "feedback" | "error";
  data?: any;
}

interface StoryData {
  requestId: string;
  story: string;
  questions: string[];
  audioUrl: string;
}

// Initialize messages with welcome message
const messages = ref<Message[]>([
  {
    id: 1,
    text: "Hi, ich bin HörMalZu-Bot und ich helfe dir dein Hörverständnis zu verbessern, indem ich dir eine kleine Übungsaufgabe erstelle. Wähle ein Thema aus der folgenden Liste aus.",
    sent: false,
    timestamp: new Date(),
  },
]);

const topics = [
  "Geschäft",
  "Reisen",
  "Sport",
  "Essen",
  "Familie",
  "Arbeit",
  "Hobbys",
  "Einkaufen",
  "Gesundheit",
  "Wetter",
];

const newMessage = ref("");
const loading = ref(false);
const selectedTopic = ref("");

// Question state
const currentQuestionIndex = ref(-1);
const userAnswers = ref<Record<number, string>>({});
const currentQuestions = ref<Array<{ text: string; id: number }>>([]);
const chatContainer = ref<HTMLElement | null>(null);

// Auto-scroll to bottom when messages change
watch(
  () => messages.value.length,
  () => {
    nextTick(() => {
      if (chatContainer.value) {
        chatContainer.value.scrollTop = chatContainer.value.scrollHeight;
      }
    });
  },
  { immediate: true }
);
const requestId = ref("");
// Removed unused isDragging variable
const waitingForAnswer = ref(false);
const questionsStarted = ref(false);
const currentStory = ref<StoryData | null>(null);

const selectTopic = async (topic: string) => {
  if (selectedTopic.value) return;

  selectedTopic.value = topic;
  const message: Message = {
    id: Date.now(),
    text: topic,
    sent: true,
    timestamp: new Date(),
  };
  messages.value.push(message);

  // Show loading message
  const loadingMessage: Message = {
    id: "loading",
    text: `Gute Wahl! Ich erstelle eine Übungsaufgabe zum Thema "${topic}". Das kann einen Moment dauern...`,
    sent: false,
    timestamp: new Date(),
  };
  messages.value.push(loadingMessage);

  try {
    // Call the backend API to generate the story
    const response = await fetch(
      `${import.meta.env.VITE_API_BASE_URL}/api/generate-story`,
      {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({ topic }),
      }
    );

    if (!response.ok) {
      throw new Error("Failed to generate story");
    }

    const storyData: StoryData = await response.json();
    currentStory.value = storyData;

    // Remove loading message
    messages.value = messages.value.filter((msg) => msg.id !== "loading");

    // Add success message with actions
    const successMessage: Message = {
      id: `story-${storyData.requestId}`,
      text: `Ich habe eine Geschichte zum Thema "${topic}" erstellt. Höre dir die Geschichte an und beantworte dann die Fragen.`,
      sent: false,
      timestamp: new Date(),
      type: "story",
      data: storyData,
    };
    messages.value.push(successMessage);
  } catch (error) {
    console.error("Error generating story:", error);

    // Remove loading message and show error
    messages.value = messages.value.filter((msg) => msg.id !== "loading");

    const errorMessage: Message = {
      id: `error-${Date.now()}`,
      text: "Entschuldigung, es gab ein Problem beim Erstellen der Geschichte. Bitte versuche es später noch einmal.",
      sent: false,
      timestamp: new Date(),
    };
    messages.value.push(errorMessage);
  }

  // Auto-scroll to bottom
  const chatContainer = document.querySelector(".chat-container");
  if (chatContainer) {
    chatContainer.scrollTop = chatContainer.scrollHeight;
  }
};

const sendMessage = async () => {
  console.log("sendMessage called", {
    newMessage: newMessage.value,
    loading: loading.value,
    waitingForAnswer: waitingForAnswer.value,
    currentQuestionIndex: currentQuestionIndex.value,
    currentQuestions: currentQuestions.value,
  });

  if (!newMessage.value.trim()) {
    console.log("Message is empty");
    return;
  }

  if (loading.value) {
    console.log("Message not sent - currently loading");
    return;
  }

  // If we're in question-answering mode
  if (waitingForAnswer.value && currentQuestionIndex.value >= 0) {
    const questionId = currentQuestions.value[currentQuestionIndex.value].id;
    userAnswers.value[questionId] = newMessage.value;

    // Add user's answer to chat
    messages.value.push({
      id: `answer-${Date.now()}`,
      text: newMessage.value,
      sent: true,
      timestamp: new Date(),
    });

    // Move to next question or finish
    if (currentQuestionIndex.value < currentQuestions.value.length - 1) {
      currentQuestionIndex.value++;

      // Add next question
      setTimeout(() => {
        messages.value.push({
          id: `question-${Date.now()}`,
          text: currentQuestions.value[currentQuestionIndex.value].text,
          sent: false,
          timestamp: new Date(),
          type: "question",
        });

        // Scroll to bottom
        nextTick(() => {
          const container = document.querySelector(".chat-container");
          if (container) container.scrollTop = container.scrollHeight;
        });
      }, 500);
    } else {
      // All questions answered
      waitingForAnswer.value = false;

      // Add completion message and evaluate answers
      setTimeout(async () => {
        messages.value.push({
          id: `complete-${Date.now()}`,
          text: "Vielen Dank für deine Antworten! Ich werde deine Antworten jetzt auswerten. Das kann einen Moment dauern...",
          sent: false,
          timestamp: new Date(),
        });

        // Send answers to the backend for evaluation
        loading.value = true;

        try {
          // Prepare the questions and answers in the required format
          const questionsWithAnswers = currentQuestions.value.map(
            (q, index) => ({
              question: q.text,
              answer: userAnswers.value[index] || "",
            })
          );

          // Call the evaluation endpoint
          const response = await fetch(
            `${import.meta.env.VITE_API_BASE_URL}/api/evaluate-answers`,
            {
              method: "POST",
              headers: {
                "Content-Type": "application/json",
              },
              body: JSON.stringify({
                story: currentStory.value?.story || "",
                questions: questionsWithAnswers,
              }),
            }
          );

          if (!response.ok) {
            throw new Error("Failed to evaluate answers");
          }

          const evaluation = await response.json();

          // Add evaluation results to chat
          messages.value.push({
            id: `evaluation-${Date.now()}`,
            text: `Auswertung: ${evaluation.overallScore}% korrekt\n${evaluation.feedback}`,
            sent: false,
            timestamp: new Date(),
            type: "evaluation",
          });

          // Add detailed feedback for each question
          evaluation.evaluations.forEach((item: any) => {
            if (!item.isCorrect) {
              messages.value.push({
                id: `feedback-${Date.now()}-${item.question.substring(0, 10)}`,
                text: `Frage: ${item.question}\nDeine Antwort: ${
                  item.answer
                }\nKorrektur: ${
                  item.correction || "Keine Korrektur nötig"
                }\nErklärung: ${item.explanation}`,
                sent: false,
                timestamp: new Date(),
                type: "feedback",
              });
            }
          });
        } catch (error) {
          console.error("Error evaluating answers:", error);
          messages.value.push({
            id: `error-${Date.now()}`,
            text: "Entschuldigung, bei der Auswertung ist ein Fehler aufgetreten. Bitte versuche es später noch einmal.",
            sent: false,
            timestamp: new Date(),
            type: "error",
          });
        } finally {
          loading.value = false;
        }
      }, 500);
    }
  } else {
    // Normal chat message
    const message: Message = {
      id: Date.now(),
      text: newMessage.value,
      sent: true,
      timestamp: new Date(),
    };
    messages.value.push(message);
  }

  // Clear input and scroll to bottom
  newMessage.value = "";
  nextTick(() => {
    const container = document.querySelector(".chat-container");
    if (container) {
      container.scrollTop = container.scrollHeight;
    }
  });
};

// Format time in MM:SS format
// Handle keyboard shortcuts
const handleKeyPress = (e: KeyboardEvent) => {
  // Only handle if not in an input field
  const target = e.target as HTMLElement;
  if (target.tagName === "INPUT" || target.tagName === "TEXTAREA") {
    return;
  }
  // Space key handling is now managed by the AudioPlayer component
};

const showQuestions = (storyData: StoryData) => {
  console.log("showQuestions called", { storyData });
  currentStory.value = storyData;
  currentQuestions.value = storyData.questions.map((q, i) => ({
    text: q,
    id: i,
  }));
  requestId.value = `req_${Date.now()}`;
  currentQuestionIndex.value = 0;
  waitingForAnswer.value = true;
  questionsStarted.value = true;

  console.log("Questions set up", {
    questionCount: currentQuestions.value.length,
    firstQuestion: currentQuestions.value[0]?.text,
  });

  // Add bot message with first question
  const questionMessage: Message = {
    id: `question-${Date.now()}`,
    text: currentQuestions.value[0]?.text || "No question available",
    sent: false,
    timestamp: new Date(),
    type: "question",
  };

  messages.value.push(questionMessage);

  // Auto-scroll is handled by the watcher
};
</script>

<template>
  <div class="app">
    <div class="chat-container" ref="chatContainer">
      <div
        v-for="message in messages"
        :key="message.id"
        :class="['message', { sent: message.sent, received: !message.sent }]"
      >
        <div class="message-content">
          <template v-if="message.id === 1">
            {{ message.text }}
            <div class="topic-selection">
              <button
                v-for="(topic, index) in topics"
                :key="index"
                @click="selectTopic(topic)"
                :class="[
                  'topic-button',
                  { 'topic-button--selected': selectedTopic === topic },
                ]"
                :disabled="!!selectedTopic"
              >
                {{ topic }}
              </button>
            </div>
          </template>
          <template v-else-if="message.type === 'story' && message.data">
            {{ message.text }}
            <AudioPlayer :audioUrl="message.data.audioUrl" />
            <div class="continue-button-container">
              <button
                @click="showQuestions(message.data)"
                class="continue-button"
              >
                Weiter zu den Fragen
              </button>
            </div>
          </template>
          <template v-else>
            {{ message.text }}
          </template>
        </div>
        <div class="message-time">
          {{
            message.timestamp.toLocaleTimeString([], {
              hour: "2-digit",
              minute: "2-digit",
            })
          }}
        </div>
      </div>
    </div>
    <div class="input-container">
      <textarea
        v-model="newMessage"
        @keydown.enter="handleKeyPress"
        :placeholder="
          !questionsStarted
            ? `Bitte höre dir zuerst die Geschichte an und klicke auf 'Weiter zu den Fragen'`
            : waitingForAnswer
            ? 'Deine Antwort hier...'
            : 'Schreibe deine Nachricht...'
        "
        :disabled="loading || !questionsStarted"
        rows="1"
      ></textarea>
      <button
        @click="sendMessage"
        :disabled="!newMessage.trim() || loading || !questionsStarted"
        :class="{ 'disabled-button': !questionsStarted }"
      >
        {{ loading ? "Sending..." : "Senden" }}
      </button>
    </div>
  </div>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen,
    Ubuntu, Cantarell, "Open Sans", "Helvetica Neue", sans-serif;
}

body,
html,
#app {
  height: 100%;
  width: 100%;
  margin: 0;
  padding: 0;
  background-color: #f5f5f5;
}

#app {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  width: 100%;
  padding: 20px;
}

.app {
  display: flex;
  flex-direction: column;
  width: 100%;
  max-width: 800px;
  height: 90vh;
  max-height: 900px;
  background-color: white;
  border-radius: 16px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
  overflow: hidden;
}

.chat-container {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
  scroll-behavior: smooth;
  display: flex;
  flex-direction: column;
  gap: 15px;
  width: 100%;
}

.message {
  max-width: 85%;
  padding: 12px 16px;
  border-radius: 18px;
  font-size: 15px;
  line-height: 1.6;
  position: relative;
  word-wrap: break-word;
}

.action-buttons {
  margin-top: 12px;
}

.topic-selection {
  margin-top: 12px;
}

.topic-button {
  background-color: #f5f7fa;
  border: 1px solid #e1e4e8;
  padding: 8px 16px;
  border-radius: 16px;
  font-size: 13px;
  cursor: pointer;
  transition: all 0.2s;
  margin: 4px;
}

.topic-button:hover {
  background-color: #f1f3f4;
  border-color: #dadce0;
}

.topic-button--selected {
  background-color: #e8f0fe;
  border-color: #d2e3fc;
  color: #1967d2;
}

.action-button {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  background-color: #1a73e8;
  color: white;
  border: none;
  border-radius: 20px;
  padding: 10px 18px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.1);
}

.action-button:hover {
  background-color: #1557b0;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}

.action-button--active {
  background-color: #0d47a1 !important;
}

.action-button--secondary {
  background-color: #5f6368;
  padding: 10px;
  border-radius: 50%;
  width: 36px;
  height: 36px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

.action-button--secondary:hover {
  background-color: #4a4e52;
}

.action-button svg {
  color: white;
}

.action-button svg {
  flex-shrink: 0;
}

.story-text {
  margin-top: 16px;
  padding: 12px;
  background-color: rgba(255, 255, 255, 0.1);
  border-radius: 8px;
  line-height: 1.6;
}

.continue-button-container {
  margin-top: 16px;
  text-align: right;
}

.continue-button {
  background-color: #28a745;
  color: white;
  border: none;
  border-radius: 20px;
  padding: 10px 20px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}

.continue-button:hover {
  background-color: #218838;
}

.message-content {
  word-wrap: break-word;
}

.topic-selection {
  display: inline-flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-top: 12px;
  max-width: 100%;
}

.topic-button {
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 20px;
  padding: 8px 16px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
  white-space: nowrap;
  margin: 2px;
}

.topic-button:hover {
  background-color: #0056b3;
}

.topic-button:active:not(:disabled) {
  background-color: #004085;
  transform: translateY(1px);
}

.topic-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  background-color: #6c757d;
}

.topic-button--selected {
  background-color: #28a745 !important;
}

.message.sent {
  background-color: #007bff;
  color: white;
  align-self: flex-end;
  border-bottom-right-radius: 4px;
}

.message.received {
  background-color: #e9ecef;
  color: #212529;
  align-self: flex-start;
  border-bottom-left-radius: 4px;
}

.message-time {
  font-size: 11px;
  opacity: 0.8;
  margin-top: 4px;
  text-align: right;
}

.disabled-button {
  opacity: 0.6;
  cursor: not-allowed;
}

.input-container {
  display: flex;
  flex-direction: row;
  padding: 12px 15px;
  background: #fff;
  border-top: 1px solid #e0e0e0;
  position: sticky;
  bottom: 0;
  gap: 10px;
  align-items: flex-end;
}

textarea {
  flex: 1;
  border: 1px solid #dee2e6;
  border-radius: 20px;
  padding: 12px 20px;
  resize: none;
  outline: none;
  font-size: 15px;
  min-height: 80px; /* Height for about two paragraphs */
  max-height: 150px;
  transition: all 0.2s;
  line-height: 1.5;
  overflow-y: auto;
}

textarea:focus {
  border-color: #007bff;
}

button {
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 20px;
  padding: 0 20px;
  height: 44px;
  font-size: 15px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
  white-space: nowrap;
  margin-left: 8px;
  flex-shrink: 0;
}

button:disabled {
  background-color: #6c757d;
  cursor: not-allowed;
}

button:not(:disabled):hover {
  background-color: #0056b3;
}

/* Scrollbar styling */
::-webkit-scrollbar {
  width: 6px;
}

::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 3px;
}

::-webkit-scrollbar-thumb {
  background: #888;
  border-radius: 3px;
}

::-webkit-scrollbar-thumb:hover {
  background: #555;
}

/* Mobile Responsive Styles */
@media (max-width: 600px) {
  .input-container {
    flex-direction: column;
    padding: 10px;
    gap: 8px;
  }

  textarea {
    width: 100%;
    margin: 0;
    padding: 10px 16px;
    /* min-height: 44px;
    max-height: 120px; */
  }

  button {
    /* width: 100%; */
    margin: 0;
    height: 44px;
  }

  .message {
    max-width: 90%;
  }

  .app {
    height: 100vh;
    max-height: 100vh;
    border-radius: 0;
  }

  .chat-container {
    padding: 12px;
  }
}
</style>
