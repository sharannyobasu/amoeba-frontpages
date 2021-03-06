<template>
  <div class="br-10 p-4">
    <div class="font-sm pb-4">
        {{quizDescription}}
    </div>

    <VAsync :task="fetchQuestion" :emberStyle="true" > 
      <template v-slot="{ value: question }">
        <div class="row no-gutters align-items-center justify-content-between mb-4 mobile-timer">
          <div class="flex-1 pr-4">
            <div class="font-md bold">
              <VMarkdown :markdown="question.description"/>
            </div>
          </div>
          <div class="s-60x60 border b-pink round">
            <div class="t-align-c pink">
              <h2 class="bold"> {{timer}} </h2>
              <div class="font-sm">SEC</div>
            </div>
          </div>
        </div>

        <Choice
          :choice="choice"
          :index="index"
          v-for="(choice, index) in question.choices"
          :onSelect="(choiceId) => submitQuestion.run(question.id, choiceId)"
          :submissionResponse="submissionResponse"
          :key="choice.id" />

          <div class="row no-gutters justify-content-between align-items-center mt-5">
              <div class="col-sm-8 col-6">
                  <div class="med-grey font-sm"> {{questionsRemaining}} Questions Remaining</div>
                  <div class="mt-4">
                      <progress class="progress-orange" :value="currentIndex" :max="total"></progress>
                  </div>
              </div>
          </div>

      </template>

    </VAsync>
  </div>
</template>

<script>
import VAsync from '~/components/Base/VAsync'
import Choice from './Choice'
import VMarkdown from '~/components/Base/VMarkdown.vue'

export default {
  name: 'Question',
  props: {
    questionId: {
      type: String,
      required: true
    },
    switchToNextQuestion: Function,
    userResponses: Array,
     total: Number,
     currentIndex: Number,
     quizDescription: String
  },
  components: {
    Choice,
    VAsync,
    VMarkdown
  },
  data () {
    return {
      oldQuestion: {},
      submissionResponse: null,
      changeAt: Date.now() + (30*1000),
      timerId: null,
      timer: 0
    }
  },
  methods: {
    afterSelectClass(choiceId) {
      if (!this.submissionResponse)
        return ''
      const {correctlyAnswered, incorrectlyAnswered } = this.submissionResponse
      if (correctlyAnswered.includes(choiceId)) {
        return 'bg-green white'
      } else if(incorrectlyAnswered.includes(choiceId)) {
        return 'bg-red white'
      } else {
        return ''
      }
    },
    resetTimer () {
      this.changeAt = Date.now() + (30 * 1000)
      clearInterval(this.timerId)
      this.timerId = setInterval(() => {
        this.timer = Math.floor((this.changeAt - Date.now())/1000)
        if (this.timer <= 0) {
          clearInterval(this.timerId)
          this.switchToNextQuestion()
        }
      })
    }
  },
  computed: {
    isCorrectlyAnswered () {
      return this.submissionResponse.correctlyAnswered.length > 0
    },
    isIncorrectlyAnswered () {
      return !this.isCorrectlyAnswered
    },
    questionsRemaining () {
      return this.total - this.currentIndex
    }
  },
  tasks (t) {
    return {
      fetchQuestion: t(function * () {
        const response = yield this.$axios.get(`/questions/${this.questionId}`, {
          params: {
            include: 'choices'
          }
        })

        this.oldQuestion = this.$jsonApiStore.sync(response.data)
        this.resetTimer()
        return this.oldQuestion
      }).runWith('questionId'),
      submitQuestion: t(function *(questionId, choiceId) {
        if (this.userResponses.find((p) => p.id === questionId))
          return (void 0)

        const { data } = yield this.$axios.post(`/questions/${questionId}/submit?showAnswers=true`, {
          markedChoices: [choiceId]
        })

        this.submissionResponse = data
        this.userResponses.push({id: questionId, markedChoices:[choiceId]})
        window.setTimeout(this.switchToNextQuestion, 1000)
      })
    }
  }
}
</script>
<style>
@media only screen and (max-width: 768px) {
    .mobile-timer {
        flex-direction: column-reverse;
    }
}
</style>
