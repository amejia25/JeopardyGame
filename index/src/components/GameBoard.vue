<template>
    <div class="player-scores">
        <div v-for="player in players" :key="player.id" 
        :class="{ 'current-player': currentPlayerIndex === player.id - 1 }">
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
            <td v-for="category in categories" :key="category.id" 
            @click="selectQuestion(category, row)"
            :class="{ answered: isQuestionAnswered(category, row) }">
            {{ isQuestionAnswered(category, row) ? isQuestionAnswered(category, row) : `$${row * 100}` }}
        </td>

        </tr>
      </tbody>
    </table>
  <div v-if="selectedQuestion">
    <p>{{ selectedQuestion.question }}</p>
    <button @click="submitAnswer('True', row * 100)">True</button>
    <button @click="submitAnswer('False', row * 100)">False</button>
    <p>{{ feedbackMessage }}</p> 
</div>

  <div  v-if="allQuestionsAnswered">
        Game Over! Player {{ winningPlayer.id }} has won the game with ${{ winningPlayer.score }}!
  </div>
  <div v-if="allQuestionsAnswered">
        Game Over! Player {{ winningPlayer.id }} has won the game with ${{ winningPlayer.score }}!
        <button @click="resetGame">Reset Game</button>
    </div>
  </template>
<script>
export default {
  data() {
    return {
      categories: [],
      selectedQuestion: null,
      feedbackMessage: '',

      players: [
      { id: 1, score: 0 },
      { id: 2, score: 0 },
      { id: 3, score: 0 }
    ],
    currentPlayerIndex: 0,

    selectedCategoryName: '',
        selectedValue: 0,
    };
  },
  methods: {
    async fetchCategories() {
      try {
        const response = await fetch('https://opentdb.com/api_category.php');
        const data = await response.json();
        return data.trivia_categories;
      } catch (error) {
        console.error("Error fetching categories:", error);
      }
    },
    shuffleArray(array) {
      for (let i = array.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [array[i], array[j]] = [array[j], array[i]];
      }
      return array;
    },
    async fetchQuestions(categoryId, difficulty, amount = 1) {
      try {
        const url = `https://opentdb.com/api.php?amount=${amount}&category=${categoryId}&difficulty=${difficulty}&type=boolean`;
        const response = await fetch(url);
        const data = await response.json();
        return data.results;
      } catch (error) {
        console.error("Error fetching questions:", error);
      }
    },
    selectQuestion(category, row) {
        this.selectedCategoryName = category.name;
        this.selectedValue = row * 100;
        if (row === 1 || row === 2) {
            this.selectedQuestion = category.easyQuestions[row - 1];
        }   else if (row === 3 || row === 4) {
            this.selectedQuestion = category.mediumQuestions[row - 3];
        }   else {
            this.selectedQuestion = category.hardQuestion[0];
        }
   },
   submitAnswer(answer, value) {
        const currentPlayer = this.players[this.currentPlayerIndex];
        if (answer === this.selectedQuestion.correct_answer) {
            currentPlayer.score += value;
            this.feedbackMessage = 'Correct!'; // Set the feedback message
        } else {
            currentPlayer.score -= value;
            this.currentPlayerIndex = (this.currentPlayerIndex + 1) % 3; // Move to the next player
            this.feedbackMessage = 'Incorrect!'; // Set the feedback message
        }
        this.selectedQuestion.answered = true; // Set the question as answered
        this.selectedQuestion = null; // Reset the selected question
    },
    isQuestionAnswered(category, row) {
        let question;
        if (row === 1 || row === 2) {
            question = category.easyQuestions[row - 1];
        } else if (row === 3 || row === 4) {
            question = category.mediumQuestions[row - 3];
        } else {
            question = category.hardQuestion[0];
        }
        return question.answered ? `P${this.currentPlayerIndex + 1}` : ''; // Return player number if answered
    },
    resetGame() {
        // Reset player scores
        this.players.forEach(player => {
            player.score = 0;
        });

        // Reset questions' answered status
        this.categories.forEach(category => {
        [...category.easyQuestions, ...category.mediumQuestions, ...category.hardQuestion].forEach((question, index, arr) => {
            this.$set(arr, index, { ...question, answered: false });
        });
    });

        this.mounted();
    }
},
    computed: {
        allQuestionsAnswered() {
        return this.categories.every(category => 
            [...category.easyQuestions, ...category.mediumQuestions, ...category.hardQuestion].every(question => question.answered)
        );
    },
    winningPlayer() {
        return this.players.reduce((max, player) => max.score > player.score ? max : player);
    }
},
  async mounted() {
    const allCategories = await this.fetchCategories();
    // Randomly select 4 categories
    this.categories = this.shuffleArray(allCategories).slice(0, 4);

    for (const category of this.categories) {
      // Fetch 2 easy questions
      category.easyQuestions = await this.fetchQuestions(category.id, 'easy', 2);
      // Fetch 2 medium questions
      category.mediumQuestions = await this.fetchQuestions(category.id, 'medium', 2);
      // Fetch 1 hard question
      category.hardQuestion = await this.fetchQuestions(category.id, 'hard', 1);
    }
  }
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
  }
  
  th {
    background-color: #252525;
    
  }
  .answered {
    background-color: #00225c;
  }
  .current-player {
    font-weight: bold;
    color: #007BFF; 
  }
  </style>
  