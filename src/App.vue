<script setup lang="ts">
import { ref, nextTick, watch } from "vue";
import AudioPlayer from "./components/AudioPlayer.vue";

interface Choice {
  text: string;
  isCorrect: boolean;
}

interface Question {
  question: string;
  choices: Choice[];
}

interface StoryData {
  requestId: string;
  story: string;
  questions: Question[];
  audioUrl: string;
}

interface Message {
  id: string | number;
  text: string;
  sent: boolean;
  timestamp: Date;
  type?: "text" | "story" | "question" | "evaluation" | "feedback" | "error" | "answer-type-selection" | "restart";
  data?: any;
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
const answerType = ref<"text" | "multiple">("text");
const showAnswerTypeSelection = ref(false);

// Question state
const currentQuestionIndex = ref(-1);
const currentQuestions = ref<Question[]>([]);
const selectedChoices = ref<{ [key: number]: number | null }>({});
const answeredQuestions = ref<{ [key: number]: boolean }>({});
const waitingForAnswer = ref(false);
const questionsStarted = ref(false);
const currentStory = ref<StoryData | null>(null);
// Counters for tracking progress (currently unused)
// const count = ref<number>(0);
// const total = ref<number>(0);

const chatContainer = ref<HTMLElement | null>(null);

const showCurrentQuestion = () => {
  if (
    currentQuestionIndex.value < 0 ||
    currentQuestionIndex.value >= currentQuestions.value.length
  )
    return;

  const question = currentQuestions.value[currentQuestionIndex.value];

  // Parse the question if it's a string (might be JSON)
  let questionText = "";
  let choices: Choice[] = [];

  try {
    // If question is already in the correct format
    if (question && typeof question === 'object' && 'question' in question) {
      questionText = String(question.question || '');
      choices = Array.isArray(question.choices) ? question.choices : [];
    } 
    // If question is a string that might contain JSON
    else if (typeof question === 'string') {
      try {
        const parsed = JSON.parse(question);
        if (parsed && typeof parsed === 'object') {
          // Handle case where the string is a question object
          if ('question' in parsed) {
            questionText = String(parsed.question || '');
            choices = Array.isArray(parsed.choices) ? parsed.choices : [];
          } 
          // Handle case where the string is a full question set
          else if (Array.isArray(parsed.questions)) {
            currentQuestions.value = parsed.questions.map((q: any) => ({
              question: String(q.question || ''),
              choices: Array.isArray(q.choices) ? q.choices : [],
            }));
            return; // Exit early since we've updated the questions
          }
        }
      } catch (e) {
        console.error('Error parsing question JSON:', e);
        questionText = 'Error: Could not parse question';
      }
    }
  } catch (e) {
    console.error('Error processing question:', e);
    questionText = 'Error: Could not load question';
  }

  // Add question to chat
  messages.value.push({
    id: `question-${Date.now()}`,
    text: questionText,
    sent: false,
    timestamp: new Date(),
    type: "question",
    data: {
      questionIndex: currentQuestionIndex.value,
      choices: choices,
    },
  });

  // Set waiting for answer
  waitingForAnswer.value = true;
};

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

  // Ask for answer type
  showAnswerTypeSelection.value = true;
  const answerTypeMessageId = `answer-type-${Date.now()}`;
  messages.value.push({
    id: answerTypeMessageId,
    text: "Möchtest du Multiple-Choice-Fragen oder Freitext-Antworten?",
    sent: false,
    timestamp: new Date(),
    type: "answer-type-selection",
    data: { id: answerTypeMessageId },
  });
};

const selectAnswerType = async (type: "text" | "multiple") => {
  answerType.value = type;
  showAnswerTypeSelection.value = false;

  // Add user's choice to chat
  messages.value.push({
    id: `answer-type-selection-${Date.now()}`,
    text: type === "multiple" ? "Multiple Choice" : "Freitext",
    sent: true,
    timestamp: new Date(),
  });

  // Show loading message
  const loadingMessage: Message = {
    id: "loading",
    text: `Gute Wahl! Ich erstelle eine Übungsaufgabe zum Thema "${
      selectedTopic.value
    }" mit ${
      type === "multiple" ? "Multiple-Choice-Fragen" : "Freitext-Antworten"
    }. Das kann einen Moment dauern...`,
    sent: false,
    timestamp: new Date(),
  };
  messages.value.push(loadingMessage);

  try {
    // Call the backend API to generate the story with the selected answer type
    const response = await fetch(
      `${import.meta.env.VITE_API_BASE_URL}/api/generate-story`,
      {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          topic: selectedTopic.value,
          answerType: type,
        }),
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
      text: `Ich habe eine Geschichte zum Thema "${selectedTopic.value}" erstellt. Höre dir die Geschichte an und beantworte dann die Fragen.`,
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
  nextTick(() => {
    const chatContainer = document.querySelector(".chat-container");
    if (chatContainer) {
      chatContainer.scrollTop = chatContainer.scrollHeight;
    }
  });
};

// Unused function - keeping for reference
/*
const submitMultipleChoiceAnswer = async () => {
  const questionIndex = currentQuestionIndex.value;
  const selectedChoiceIndex = selectedChoices.value[questionIndex];

  if (selectedChoiceIndex === null || selectedChoiceIndex === undefined) return;

  const question = currentQuestions.value[questionIndex];
  const selectedChoice = question.choices[selectedChoiceIndex];

  // Add user's answer to chat
  messages.value.push({
    id: `${Date.now()}-answer`,
    text: selectedChoice.text,
    sent: true,
    timestamp: new Date(),
  });

  // Show feedback immediately for multiple choice
  const feedbackMessage: Message = {
    id: `feedback-${Date.now()}`,
    text: selectedChoice.isCorrect
      ? "Richtig! "
      : `Leider falsch. Die richtige Antwort wäre: ${
          question.choices.find((c) => c.isCorrect)?.text
        }`,
    sent: false,
    timestamp: new Date(),
  };
  messages.value.push(feedbackMessage);

  // Move to next question or finish
  goToNextQuestion();
};
*/

const submitTextAnswer = async (answerText?: string) => {
  const answer = answerText || newMessage.value.trim();
  if (!answer) return;

  newMessage.value = "";
  waitingForAnswer.value = false;

  // Mark current question as answered
  const currentIndex = currentQuestionIndex.value;
  answeredQuestions.value = {
    ...answeredQuestions.value,
    [currentIndex]: true,
  };

  // Store the answer
  selectedChoices.value = {
    ...selectedChoices.value,
    [currentIndex]: 0, // For text answers, we just need to mark it as answered
  };

  // Add user's answer to chat
  messages.value.push({
    id: `${Date.now()}-answer`,
    text: answer,
    sent: true,
    timestamp: new Date(),
  });

  // Move to next question with isTextAnswer flag
  goToNextQuestion(true);

  // If all questions are answered, evaluate them
  if (currentQuestionIndex.value >= currentQuestions.value.length) {
    await evaluateAnswers();
  }
};

const goToNextQuestion = (isTextAnswer: boolean = false) => {
  const currentIndex = currentQuestionIndex.value;

  // Only process answer if this is a multiple choice question
  if (!isTextAnswer && selectedChoices.value[currentIndex] !== null) {
    const question = currentQuestions.value[currentIndex];
    const selectedChoice = question.choices[selectedChoices.value[currentIndex]!];

    // Mark this question as answered
    answeredQuestions.value = {
      ...answeredQuestions.value,
      [currentIndex]: true,
    };

    // For text answers, we've already added the message in submitTextAnswer
    if (!isTextAnswer) {
      messages.value.push({
        id: `answer-${Date.now()}`,
        text: selectedChoice.text,
        sent: true,
        timestamp: new Date(),
      });
    }
  }


  // Move to next question or finish
  currentQuestionIndex.value++;

  if (currentQuestionIndex.value < currentQuestions.value.length) {
    // Show next question
    showCurrentQuestion();
    waitingForAnswer.value = true;
  } else {
    // All questions answered
    const correctCount = Object.values(selectedChoices.value).reduce<number>(
      (acc, choiceIndex, index) => {
        if (choiceIndex === null) return acc;
        return (
          acc +
          (currentQuestions.value[index].choices[choiceIndex]?.isCorrect
            ? 1
            : 0)
        );
      },
      0
    );

    const completionMessage: Message = {
      id: `completion-${Date.now()}`,
      text: `Glückwunsch! Du hast alle Fragen beantwortet. Du hast ${correctCount} von ${currentQuestions.value.length} Fragen richtig beantwortet.`,
      sent: false,
      timestamp: new Date(),
    };
    messages.value.push(completionMessage);
    questionsStarted.value = false;
  }

  // Scroll to bottom
  nextTick(() => {
    const container = document.querySelector(".chat-container");
    if (container) container.scrollTop = container.scrollHeight;
  });
};

const evaluateAnswers = async () => {
  // Add completion message
  messages.value.push({
    id: `complete-${Date.now()}`,
    text: "Vielen Dank für deine Antworten! Ich werde deine Antworten jetzt auswerten. Das kann einen Moment dauern...",
    sent: false,
    timestamp: new Date(),
  });

  // Send answers to the backend for evaluation
  loading.value = true;

  try {
    if (answerType.value === "multiple") {
      // Handle multiple choice evaluation
      const questionsWithAnswers = currentQuestions.value.map((q, index) => ({
        question: q.question,
        answer:
          selectedChoices.value[index] !== null
            ? q.choices[selectedChoices.value[index]!].text
            : "",
        isCorrect:
          selectedChoices.value[index] !== null
            ? q.choices[selectedChoices.value[index]!].isCorrect
            : false,
      }));

      // Calculate score for multiple choice
      const correctAnswers = questionsWithAnswers.filter(
        (q) => q.isCorrect
      ).length;
      const totalQuestions = questionsWithAnswers.length;
      const score = Math.round((correctAnswers / totalQuestions) * 100);

      // Add evaluation results to chat
      const evaluationMessage: Message = {
        id: `evaluation-${Date.now()}`,
        text: `Auswertung: ${score}% korrekt (${correctAnswers} von ${totalQuestions} Fragen richtig beantwortet)`,
        sent: false,
        timestamp: new Date(),
        type: "evaluation",
      };

      // Add detailed feedback for incorrect answers
      const feedbackMessages: Message[] = [];
      questionsWithAnswers.forEach((item, index) => {
        const question = currentQuestions.value[index];
        const selectedChoiceIndex = selectedChoices.value[index];
        const selectedChoice =
          selectedChoiceIndex !== null
            ? question.choices[selectedChoiceIndex]
            : null;
        const correctChoice = question.choices.find(
          (choice) => choice.isCorrect
        );

        if (!item.isCorrect && selectedChoice && correctChoice) {
          feedbackMessages.push({
            id: `feedback-${Date.now()}-${index}`,
            text: `Frage: ${question.question}<br>Deine Antwort: ${selectedChoice.text}<br>Richtige Antwort: ${correctChoice.text}`,
            sent: false,
            timestamp: new Date(),
          });
        }
      });

      // Add completion message
      const completionMessage = {
        id: `completion-${Date.now()}`,
        text: `Glückwunsch! Du hast alle Fragen beantwortet.`,
        sent: false,
        timestamp: new Date(),
      };

      // Add all messages at once to prevent race conditions
      messages.value.push(evaluationMessage);
      if (feedbackMessages.length > 0) {
        messages.value.push(...feedbackMessages);
      }

      messages.value.push(completionMessage);
    } else {
      // Handle Freitext evaluation - send to API
      const userAnswers = currentQuestions.value.map((question) => {
        const answer = messages.value
          .filter((msg) => msg.sent)
          .find(
            (msg) =>
              msg.text &&
              msg.text.trim() &&
              messages.value.indexOf(msg) >
                messages.value.findIndex((m) => m.text === question.question)
          );
        return answer?.text || "";
      });

      // Prepare the request body with question-answer pairs
      const questionsWithAnswers = currentQuestions.value.map(
        (question, index) => ({
          question: question.question,
          answer: userAnswers[index] || "",
        })
      );

      const requestBody = {
        story: currentStory.value?.story || "",
        questions: questionsWithAnswers,
      };

      // Send to API for evaluation
      const response = await fetch(
        `${import.meta.env.VITE_API_BASE_URL}/api/evaluate-answers`,
        {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify(requestBody),
        }
      );

      if (!response.ok) {
        const errorData = await response.json().catch(() => ({}));
        throw new Error(errorData.error || "Failed to evaluate answers");
      }

      const result = await response.json();

      // Show evaluation results
      if (result.feedback) {
        // Add the overall feedback
        messages.value.push({
          id: `evaluation-${Date.now()}`,
          text: `Auswertung: ${result.overallScore}% korrekt\n${result.feedback}`,
          sent: false,
          timestamp: new Date(),
        });

        // Add detailed feedback for each question
        if (Array.isArray(result.evaluations)) {
          result.evaluations.forEach((evaluation: any, index: number) => {
            const messageText = [
              `Frage: ${evaluation.question}`,
              `Deine Antwort: ${evaluation.answer}`,
              "", // Empty line for spacing
              evaluation.isCorrect
                ? "✅ Richtig!"
                : `❌ Falsch. ${evaluation.correction}`,
              "", // Empty line for spacing
              evaluation.explanation,
            ].join("<br>");

            messages.value.push({
              id: `answer-${Date.now()}-${index}`,
              text: messageText,
              sent: false,
              timestamp: new Date(),
            });
          });
        }
      }
    }

    // Add restart button
    messages.value.push({
      id: `restart-${Date.now()}`,
      text: "Möchtest du es noch einmal versuchen?",
      sent: false,
      timestamp: new Date(),
      type: "restart",
    });
  } catch (error) {
    console.error("Error evaluating answers:", error);
    messages.value.push({
      id: `error-${Date.now()}`,
      text: "Entschuldigung, es gab ein Problem bei der Auswertung. Bitte versuche es später noch einmal.",
      sent: false,
      timestamp: new Date(),
    });
  } finally {
    loading.value = false;
    questionsStarted.value = false;
  }
};

const restartQuiz = () => {
  window.location.reload();
};

const allQuestionsAnswered = () => {
  return Object.values(selectedChoices.value).every(
    (choice) => choice !== null
  );
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
    // Handle text answer in Freitext mode
    if (answerType.value === "text") {
      const answer = newMessage.value.trim();
      if (answer) {
        // Submit the text answer with the answer text
        await submitTextAnswer(answer);
      }
    }
    // Handle multiple choice answer
    else {
      const currentQuestion =
        currentQuestions.value[currentQuestionIndex.value];
      const selectedChoice = currentQuestion.choices.find(
        (_, index) =>
          selectedChoices.value[currentQuestionIndex.value] === index
      );

      if (selectedChoice) {
        // Add user's answer to chat
        messages.value.push({
          id: `answer-${Date.now()}`,
          text: selectedChoice.text,
          sent: true,
          timestamp: new Date(),
        });

        // Move to next question or finish
        goToNextQuestion();

        // If all questions are answered, evaluate them
        if (allQuestionsAnswered()) {
          setTimeout(evaluateAnswers, 500);
        }
      }
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

// Handle keyboard shortcuts
const handleKeyPress = (e: KeyboardEvent) => {
  const target = e.target as HTMLElement;

  // Handle Enter key in textarea
  if (target.tagName === "TEXTAREA" && e.key === "Enter" && !e.shiftKey) {
    e.preventDefault();
    if (
      answerType.value === "text" &&
      questionsStarted.value &&
      waitingForAnswer.value
    ) {
      submitTextAnswer();
    } else {
      sendMessage();
    }
    return;
  }

  // Space key handling is now managed by the AudioPlayer component
};

const showQuestions = (storyData: StoryData) => {
  console.log("showQuestions called", { storyData });
  currentStory.value = storyData;

  // Process questions - they might be strings or objects
  currentQuestions.value = storyData.questions.map((q: any, i: number) => {
    // If it's already an object with question and choices
    if (q && typeof q === "object" && "question" in q) {
      return {
        id: i,
        question: String(q.question || ""),
        choices: Array.isArray(q.choices) ? q.choices : [],
      };
    }

    // If it's a string that starts with {, try to parse as JSON
    const str = String(q);
    if (str.trim().startsWith("{")) {
      try {
        const parsed = JSON.parse(str);
        return {
          id: i,
          question: String(parsed.question || ""),
          choices: Array.isArray(parsed.choices) ? parsed.choices : [],
        };
      } catch (e) {
        console.error("Error parsing question as JSON:", e);
        // Fall through to plain text handling
      }
    }

    // Handle as plain text question (for Freitext mode)
    return {
      id: i,
      question: String(q),
      choices: [],
    };
  });

  requestId.value = `req_${Date.now()}`;
  currentQuestionIndex.value = 0;
  waitingForAnswer.value = true;
  questionsStarted.value = true;

  console.log("Questions set up", {
    questionCount: currentQuestions.value.length,
    firstQuestion: currentQuestions.value[0]?.question,
  });

  // Show the first question
  showCurrentQuestion();

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
            <div>
              {{ message.text }}
            </div>
            <div v-if="!showAnswerTypeSelection" class="topics-container">
              <button
                v-for="(topic, index) in topics"
                :key="index"
                @click="selectTopic(topic)"
                class="topic-button"
                :disabled="!!selectedTopic"
              >
                {{ topic }}
              </button>
            </div>
          </template>
          <template v-else-if="message.type === 'answer-type-selection'">
            <div>
              {{ message.text }}
            </div>
            <div class="answer-type-container">
              <button
                @click="selectAnswerType('multiple')"
                class="answer-type-button"
              >
                <span>Multiple Choice</span>
                <small>Wähle aus vorgegebenen Antworten</small>
              </button>
              <button
                @click="selectAnswerType('text')"
                class="answer-type-button"
              >
                <span>Freitext</span>
                <small>Schreibe deine eigene Antwort</small>
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
          <template
            v-else-if="message.type === 'question' && message.data?.choices"
          >
            <div class="question-text">{{ message.text }}</div>
            <div class="choices-container">
              <ul class="choices-list">
                <li
                  v-for="(choice, index) in message.data.choices"
                  :key="index"
                  class="choice-item"
                >
                  <label class="choice-option">
                    <input
                      type="radio"
                      :name="'question-' + message.data.questionIndex"
                      :value="index"
                      v-model="selectedChoices[message.data.questionIndex]"
                      :disabled="answeredQuestions[message.data.questionIndex]"
                      class="choice-radio"
                    />
                    <span class="choice-text">{{ choice.text }}</span>
                  </label>
                </li>
              </ul>
              <div v-if="answerType === 'multiple'" class="question-actions">
                <button
                  @click="goToNextQuestion()"
                  :disabled="
                    selectedChoices[message.data.questionIndex] === null ||
                    answeredQuestions[message.data.questionIndex]
                  "
                  class="next-question-button"
                >
                  Nächste Frage
                </button>
              </div>
            </div>
          </template>
          <template v-else-if="message.type === 'restart'">
            <div>{{ message.text }}</div>
            <button @click="restartQuiz" class="restart-button">
              Neu starten
            </button>
          </template>
          <template v-else>
            <div v-html="message.text"></div>
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
    <div
      class="input-container"
      v-if="questionsStarted && answerType === 'text' && waitingForAnswer"
    >
      <textarea
        v-model="newMessage"
        @keydown.enter.exact.prevent="handleKeyPress"
        placeholder="Schreibe deine Nachricht..."
        :disabled="loading || !selectedTopic || answerType !== 'text'"
      ></textarea>
      <button
        @click="sendMessage"
        :disabled="!newMessage.trim() || loading || answerType !== 'text'"
      >
        Senden
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
  opacity: 0.7;
  cursor: not-allowed;
  background-color: #6c757d;
}

.topic-button--selected {
  background-color: #28a745 !important;
}

.answer-type-container {
  display: flex;
  gap: 12px;
  margin-top: 12px;
  flex-wrap: wrap;
}

.answer-type-button {
  background-color: #f8f9fa;
  color: #1a73e8;
  border: 1px solid #dadce0;
  border-radius: 20px;
  padding: 10px 20px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-width: 120px;
  text-align: center;
}

.answer-type-button:hover {
  background-color: #1a73e8;
  border-color: #1a73e8;
  color: white;
}

.answer-type-button:hover small {
  color: rgba(255, 255, 255, 0.9) !important;
}

/* Multiple Choice Styles */
.choices-container {
  margin-top: 12px;
  width: 100%;
}

.choices-list {
  list-style: none;
  padding: 0;
  margin: 0 0 16px 0;
}

.choice-item {
  margin: 8px 0;
  padding: 0;
}

.choice-option {
  display: flex;
  align-items: center;
  cursor: pointer;
  padding: 10px 12px;
  border-radius: 8px;
  transition: background-color 0.2s;
}

.choice-option:hover {
  background-color: #f5f5f5;
}

.choice-radio {
  margin-right: 12px;
  width: 18px;
  height: 18px;
  cursor: pointer;
}

.choice-text {
  font-size: 15px;
  line-height: 1.4;
  color: #202124;
}

.question-actions {
  display: flex;
  justify-content: flex-end;
  margin-top: 16px;
}

.next-question-button {
  background-color: #1a73e8;
  color: white;
  border: none;
  border-radius: 20px;
  padding: 10px 24px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.2s;
}

.next-question-button:hover:not(:disabled) {
  background-color: #1765cc;
}

.next-question-button:disabled {
  background-color: #e0e0e0;
  color: #9e9e9e;
  cursor: not-allowed;
}

.restart-button {
  background-color: #1a73e8;
  color: white;
  border: none;
  border-radius: 20px;
  padding: 10px 24px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  margin-top: 12px;
  transition: background-color 0.2s;
}

.restart-button:hover {
  background-color: #1765cc;
}

.question-text {
  font-size: 16px;
  font-weight: 500;
  margin-bottom: 16px;
  line-height: 1.4;
  color: #202124;
}

.answer-type-button:active {
  background-color: #e8f0fe;
  transform: translateY(1px);
}

.answer-type-button span {
  display: block;
  margin-bottom: 2px;
}

.answer-type-button small {
  font-size: 11px;
  color: #5f6368;
  font-weight: normal;
  display: block;
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
