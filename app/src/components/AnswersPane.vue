<template>
  <BasePane
    class="answers-pane right-pane large"
    icon="comment"
    title="Answers"
    @close="close"
  >
    <QuestionItem
      :question="question"
      hide-actions
      class="item emphasize"
    />

    <template v-if="question.pickedAnswer">
      <AnswerItem
        :answer="question.pickedAnswer"
        :question="question"
        hide-actions
        class="item showcase"
      />

      <div class="more-separator">More answers</div>
    </template>

    <ApolloQuery
      :query="require('../graphql/QuestionAnswers.gql')"
      :variables="{
        questionId: question.id,
      }"
      fetch-policy="cache-and-network"
    >
      <ApolloSubscribeToMore
        :document="require('../graphql/AnswerAdded.gql')"
        :variables="{
          questionId: question.id
        }"
        :update-query="onAnswerAdded"
      />
      <ApolloSubscribeToMore
        :document="require('../graphql/AnswerRemoved.gql')"
        :variables="{
          questionId: question.id
        }"
        :update-query="onAnswerRemoved"
      />
      <ApolloSubscribeToMore
        :document="require('../graphql/AnswerUpdated.gql')"
        :variables="{
          questionId: question.id
        }"
      />

      <template slot-scope="{ result: { data, loading, error } }">
        <BaseLineLoading v-if="loading"/>

        <div v-else-if="error" class="info-block danger">
          <BaseIcon icon="error"/>
          <div>An error occured while fetching the answers.</div>
        </div>

        <template v-if="data && data.question && data.question.answers.length">
          <AnswerItem
            v-for="answer of data.question.answers"
            :key="answer.id"
            :answer="answer"
            :question="question"
            class="item"
          />
        </template>

        <div v-else-if="!loading" class="empty">No answers yet</div>
      </template>
    </ApolloQuery>

    <div
      v-if="user"
      slot="footer"
      class="pane-footer"
    >
      <div v-if="writeAnswer" class="answer-form">
        <div class="form custom">
          <textarea
            ref="contentInput"
            v-model="content"
            class="content-input"
            placeholder="Write an answer (markdown)"
            required
            maxlength="500"
            rows="4"
          />
        </div>

        <div class="actions">
          <BaseButton
            icon="close"
            class="secondary"
            @click="writeAnswer = false"
          >
            Cancel
          </BaseButton>
          <BaseButton
            icon="send"
            :disabled="!formValid"
            @click="sendAnswer"
          >
            Send answer
          </BaseButton>
        </div>
      </div>
      <template v-else>
        <BaseButton
          @click="openAnswerForm"
        >
          Write an answer
        </BaseButton>
      </template>
    </div>
  </BasePane>
</template>

<script>
import { mapGetters, mapActions } from 'vuex'
import { cacheAnswerAdd, cacheAnswerAddToList, cacheAnswerRemoveFromList } from '../cache/answers.js'

import AnswerItem from './AnswerItem.vue'
import QuestionItem from './QuestionItem.vue'

import ANSWER_ADD_MUTATION from '../graphql/AnswerAdd.gql'

export default {
  components: {
    AnswerItem,
    QuestionItem,
  },

  props: {
    question: {
      type: Object,
      required: true,
    },
    session: {
      type: Object,
      required: true,
    },
  },

  data () {
    return {
      content: '',
      writeAnswer: false,
    }
  },

  computed: {
    ...mapGetters('user', [
      'user',
    ]),

    formValid () {
      return this.content
    },
  },

  methods: {
    ...mapActions('ui', [
      'setShowAnswerPane',
    ]),

    close () {
      this.setShowAnswerPane(false)
    },

    openAnswerForm () {
      this.content = ''
      this.writeAnswer = true
      this.$nextTick(() => {
        this.$refs.contentInput.focus()
      })
    },

    sendAnswer () {
      if (this.formValid) {
        this.$apollo.mutate({
          mutation: ANSWER_ADD_MUTATION,
          variables: {
            questionId: this.question.id,
            input: {
              content: this.content,
            },
          },
          update: (store, { data: { answerAdd } }) => {
            cacheAnswerAdd(store, {
              questionId: this.question.id,
            }, answerAdd)
          },
          // optimisticResponse: {
          //   __typename: 'Mutation',
          //   answerAdd: {
          //     __typename: 'Answer',
          //     id: '',
          //     content: this.content,
          //     user: this.user,
          //     date: Date.now(),
          //   },
          // },
        })

        this.writeAnswer = false
      }
    },

    onAnswerAdded (previousResult, { subscriptionData }) {
      const newResult = {
        question: {
          ...previousResult.question,
          answers: [...previousResult.question.answers],
        },
      }
      cacheAnswerAddToList(newResult.question.answers, subscriptionData.data.answerAdded)
      return newResult
    },

    onAnswerRemoved (previousResult, { subscriptionData }) {
      const newResult = {
        question: {
          ...previousResult.question,
          answers: [...previousResult.question.answers],
        },
      }
      cacheAnswerRemoveFromList(newResult.question.answers, subscriptionData.data.answerRemoved)
      return newResult
    },
  },
}
</script>

<style lang="stylus" scoped>
@import "../styles/imports"

.item
  margin 0 32px 12px

.more-separator
  text-align center
  color $color-accent
  margin 42px 32px 12px

.pane-footer,
.actions
  h-box()
  box-center()

.answer-form
  v-box()
  align-items stretch
  width 100%

  .form.custom
    padding 0 0 12px 0

</style>
