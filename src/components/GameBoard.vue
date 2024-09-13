<template>
  <div>
    <div v-if="!gameStarted" class="start-message">
      Please select a number of players and categories to load the game.
    </div>

    <div class="config-section" v-if="!gameStarted">
      <label>Number of players: </label>
      <select v-model="numPlayers">
        <option v-for="n in 5" :key="n" :value="n+1">{{ n+1 }}</option>
      </select>
      <br>
      <label>Number of categories: </label>
      <select v-model="numCategories">
        <option v-for="n in 24" :key="n" :value="n">{{ n }}</option>
      </select>
      <br>
      <button @click="startGame" :disabled="isLoading">{{ isLoading ? 'Loading...' : 'Start Game' }}</button>
    </div>

    <div v-if="isLoading" class="loading-bar">
      <div class="loading-bar-progress" :style="{ width: loadingProgress + '%' }"></div>
      <p>Loading... {{ loadingProgress }}%</p>
    </div>

    <div v-if="gameStarted && !isLoading">
      <div class="player-scores">
        <div v-for="player in players" :key="player.id" :class="{ 'current-player': currentPlayerIndex === player.id - 1 }">
          Player {{ player.id }}: ${{ player.score }}
        </div>
      </div>
      <table>
        <thead>
          <tr>
            <th v-for="category in categories" :key="category.id">{{ category.name }}</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="row in 5" :key="row">
            <td v-for="category in categories" :key="category.id" @click="selectQuestion(category, row)" :class="isQuestionAnswered(category, row)">
              {{ isQuestionAnswered(category, row).includes('P') ? `P${isQuestionAnswered(category, row).charAt(isQuestionAnswered(category, row).length - 1)}` : `$${row * 100}` }}
            </td>
          </tr>
        </tbody>
      </table>
      <div v-if="selectedQuestion">
        <p>Selected: {{ selectedCategoryName }} for ${{ selectedValue }}</p>
        <p>{{ selectedQuestion.question }}</p>
        <button @click="submitAnswer('True', selectedValue)">True</button>
        <button @click="submitAnswer('False', selectedValue)">False</button>
        <p>{{ feedbackMessage }}</p>
      </div>
      <div v-if="allQuestionsAnswered">
        Game Over! Player {{ winningPlayer.id }} has won the game with ${{ winningPlayer.score }}!
        <button @click="resetGame">Reset Game</button>
      </div>
    </div>
  </div>
</template>
<script>
export default {
  data() {
    return {
      categories: [],
      allCategories: [],
      selectedQuestion: null,
      feedbackMessage: null,
      doubleJeopardyValue: null,
      numPlayers: 2,
      numCategories: 4,
      gameStarted: false,
      isLoading: false,
      loadingProgress: 0,
      players: [],
      currentPlayerIndex: 0,
      selectedCategoryName: '',
      selectedValue: 0,
      unsupportedCategories: [10, 13, 26, 29, 21, 27, 30, 32],
      questionCache: {},
    };
  },
  async created() {
    await this.preloadCategories();
  },
  methods: {
    async preloadCategories() {
      const cachedCategories = localStorage.getItem('trivia_categories');
      if (cachedCategories) {
        this.allCategories = JSON.parse(cachedCategories);
      } else {
        this.allCategories = await this.fetchCategories();
        localStorage.setItem('trivia_categories', JSON.stringify(this.allCategories));
      }
    },
    async fetchCategories() {
      try {
        const response = await fetch('https://opentdb.com/api_category.php');
        const data = await response.json();
        return data.trivia_categories;
      } catch (error) {
        console.error("Error fetching categories:", error);
        return [];
      }
    },
    shuffleArray(array) {
      for (let i = array.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [array[i], array[j]] = [array[j], array[i]];
      }
      return array;
    },
    async fetchQuestions(categoryId, amount = 5) {
      const cacheKey = `trivia_${categoryId}`;
      
      if (!this.questionCache[cacheKey]) {
        try {
          const url = `https://opentdb.com/api.php?amount=${amount}&category=${categoryId}&type=boolean`;
          const response = await fetch(url);
          if (response.status === 429) {
            await new Promise(resolve => setTimeout(resolve, 2000));
            return this.fetchQuestions(categoryId, amount);
          }
          const data = await response.json();
          this.questionCache[cacheKey] = data.results;
          localStorage.setItem(cacheKey, JSON.stringify(data.results));
        } catch (error) {
          console.error("Error fetching questions:", error);
          this.questionCache[cacheKey] = [];
        }
      }
      
      return this.questionCache[cacheKey];
    },
    async startGame() {
      console.log("startGame method executed");
      this.gameStarted = true;
      this.isLoading = true;
      this.loadingProgress = 0;

      try {
        this.players = Array.from({ length: this.numPlayers }, (_, i) => ({ id: i + 1, score: 0 }));

        const filteredCategories = this.allCategories.filter(category => !this.unsupportedCategories.includes(category.id));
        this.categories = this.shuffleArray(filteredCategories).slice(0, this.numCategories);

        const totalCategories = this.categories.length;
        let fetchedCategories = 0;

        await Promise.all(this.categories.map(async (category) => {
          let questions = await this.fetchQuestions(category.id, 5);
          
          const easyQuestions = questions.filter(q => q.difficulty === 'easy');
          const mediumQuestions = questions.filter(q => q.difficulty === 'medium');
          const hardQuestions = questions.filter(q => q.difficulty === 'hard');

          category.easyQuestions = easyQuestions.length >= 2 ? easyQuestions.slice(0, 2) : this.fillMissingQuestions(easyQuestions, 2);
          category.mediumQuestions = mediumQuestions.length >= 2 ? mediumQuestions.slice(0, 2) : this.fillMissingQuestions(mediumQuestions, 2);
          category.hardQuestion = hardQuestions.length >= 1 ? [hardQuestions[0]] : this.fillMissingQuestions(hardQuestions, 1);

          fetchedCategories++;
          this.loadingProgress = Math.floor((fetchedCategories / totalCategories) * 100);

          return category;
        }));

        console.log("Game loaded successfully");
      } catch (error) {
        console.error("Error starting game:", error);
        alert("An error occurred while starting the game. Please try again.");
      } finally {
        this.isLoading = false;
      }
    },
    fillMissingQuestions(questions, required) {
      while (questions.length < required) {
        questions.push({
          question: "Placeholder question due to API limitation",
          correct_answer: Math.random() < 0.5 ? "True" : "False",
          difficulty: "medium"
        });
      }
      return questions;
    },
    selectQuestion(category, row) {
      if (!category || !category.easyQuestions || !category.mediumQuestions || !category.hardQuestion) {
        console.log("Invalid category or missing questions.");
        return;
      }

      this.feedbackMessage = null;
      this.selectedCategoryName = category.name;
      this.selectedValue = row * 100;

      if (row === 1 || row === 2) {
        this.selectedQuestion = category.easyQuestions[row - 1];
      } else if (row === 3 || row === 4) {
        this.selectedQuestion = category.mediumQuestions[row - 3];
      } else {
        this.selectedQuestion = category.hardQuestion[0];
      }

      if (!this.selectedQuestion) {
        console.log("Question not found for selected cell.");
        return;
      }

      const randomNumber = Math.random();
      if (randomNumber <= 0.10) {
        this.initiateDoubleJeopardy();
      }
    },
    submitAnswer(answer, value) {
      const currentPlayer = this.players[this.currentPlayerIndex];
      const valueToAdjust = this.doubleJeopardyValue || this.selectedValue;

      if (answer === this.selectedQuestion.correct_answer) {
        currentPlayer.score += valueToAdjust;
        this.feedbackMessage = 'Correct!';
        this.selectedQuestion.isCorrect = true; 
      } else {
        currentPlayer.score -= valueToAdjust;
        this.currentPlayerIndex = (this.currentPlayerIndex + 1) % this.players.length;
        this.feedbackMessage = 'Incorrect!';
        this.selectedQuestion.isCorrect = false; 
      }
      this.selectedQuestion.answeredBy = currentPlayer.id; 
      this.selectedQuestion.answered = true;

      this.doubleJeopardyValue = null;

      setTimeout(() => {
        this.selectedQuestion = null;
        this.feedbackMessage = null;
      }, 2000);
    },
    isQuestionAnswered(category, row) {
      if (!category || !category.easyQuestions || !category.mediumQuestions || !category.hardQuestion) {
        return '';
      }

      let question;
      if (row === 1 || row === 2) {
        question = category.easyQuestions[row - 1];
      } else if (row === 3 || row === 4) {
        question = category.mediumQuestions[row - 3];
      } else {
        question = category.hardQuestion[0];
      }

      if (question && question.answered) {
        return question.isCorrect ? `correct P${question.answeredBy}` : `incorrect P${question.answeredBy}`;
      }
      return '';
    },
    resetGame() {
      this.players.forEach(player => {
        player.score = 0;
      });
      this.categories.forEach(category => {
        [...category.easyQuestions, ...category.mediumQuestions, ...category.hardQuestion].forEach(question => {
          question.answered = false;
        });
      });
      this.currentPlayerIndex = 0;
      this.startGame();
    },
    initiateDoubleJeopardy() {
      const currentPlayer = this.players[this.currentPlayerIndex];
      const faceValue = this.selectedValue;

      if (currentPlayer.score < faceValue) {
        this.doubleJeopardyValue = faceValue;
      } else {
        const wager = parseInt(prompt("Double Jeopardy wager: "));
        if (wager > currentPlayer.score || wager <= 0) {
          alert("Invalid wager. Please enter a value up to your current balance.");
          this.initiateDoubleJeopardy();
        } else {
          this.doubleJeopardyValue = wager;
        }
      }
    },
  },
  computed: {
    allQuestionsAnswered() {
      if (this.categories.length === 0) return false;
      return this.categories.every(category => 
        [...category.easyQuestions, ...category.mediumQuestions, ...category.hardQuestion].every(question => question.answered)
      );
    },
    winningPlayer() {
      return this.players.reduce((max, player) => max.score > player.score ? max : player);
    }
  },
};
</script>
<style scoped>
table {
  width: 100%;
  border-collapse: collapse;
}

th, td {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: center;
  background-color: #000474; 
  color: #ffb300; 
}

th {
  background-color: #000474;
  color: rgba(255, 255, 255, 0.862);
}

.answered {
  background-color: #060ce9;
}

.current-player {
  font-size: 15px;
  font-weight: bold;
  color: #ffb300; 
}

.correct {
  background-color: green;
}

.incorrect {
  background-color: red;
}

td {
  font-size: 24px; 
  font-weight: bold; 
}

.start-message {
  font-size: 18px;
  font-weight: bold;
  margin-bottom: 20px;
}

.loading-bar {
  width: 100%;
  background-color: #ddd;
  height: 25px;
  margin-top: 10px;
}

.loading-bar-progress {
  height: 100%;
  background-color: #4caf50;
}
</style>
