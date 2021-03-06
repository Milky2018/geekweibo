<template>
  <div class="mt-2 rounded">
    <div aria-label="create comment form" class="flex flex-col p-3 bg-gray-100 rounded">
      <div class="flex flex-row">
        <img v-if="currentUser" :src="scaledAvatar(currentUser.avatar, currentUser.nickname)" class="object-cover w-8 h-8 rounded-full" />
        <div class="flex flex-col w-full ml-3">
          <div
            contenteditable="true"
            v-html="tweet.new_comment"
            @input="e => updateNewCommentOf(e, tweet)"
            class="w-full p-2 bg-white border-indigo-100 border-solid rounded outline-none"
            v-on:keydown.meta.enter="postComment(tweet)"
            ref="tweet_comment_inbox"
          ></div>
          <div class="flex items-center justify-between">
            <div class="flex items-center">
              <span v-if="replying_comment" class="text-sm text-gray-600">
                回复 @{{ replying_comment.user.nickname }}: {{ replying_comment.body }}
              </span>
            </div>
            <button
              @click="postComment(tweet)"
              class="self-end px-3 py-2 mt-2 text-xs text-white bg-indigo-500 rounded outline-none focus:outline-none active:bg-indigo-600"
            >
              评论
            </button>
          </div>
        </div>
      </div>
    </div>
    <div v-for="comment in tweet.comments" class="flex flex-row mt-3 ml-3" v-bind:key="comment.id">
      <img :src="scaledAvatar(comment.user.avatar, comment.user.nickname)" class="object-cover w-8 h-8 rounded-full" />
      <div class="w-full ml-2">
        <div class="flex items-center justify-between">
          <div class="text-xs font-medium text-gray-900">{{ comment.user.nickname }}</div>
          <svg
            class="w-5 h-5 p-1 text-gray-400 rounded-full hover:bg-gray-200 transition duration-300"
            aria-label="delete comment icon"
            v-if="isMyComment(comment)"
            viewBox="0 0 20 20"
            fill="currentColor"
            @click="confirmDeleteComment(tweet, comment)"
          >
            <path
              fill-rule="evenodd"
              d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z"
              clip-rule="evenodd"
            />
          </svg>
        </div>
        <div
          class="text-sm text-gray-500 cursor-pointer markdown"
          @click="replying_comment = replying_comment ? null : comment"
          v-html="commentContent(comment)"
        ></div>
      </div>
    </div>
    <a
      v-if="tweet.has_more_comments"
      :href="`/tweets/${tweet.id}`"
      class="inline-block w-full pt-3 mt-3 text-sm text-center border-t border-gray-200"
    >
      查看更多&nbsp;&gt;
    </a>
    <v-dialog />
  </div>
</template>

<script>
import LoginDialog from '../../common/LoginDialog.vue'
import DOMPurify from 'dompurify'
import Vue from 'vue/dist/vue.esm'
import VModal from 'vue-js-modal'
Vue.use(VModal, { dialog: true })
import { markdown, scaledAvatar } from '../../common/Utils'

export default {
  props: ['currentUser', 'tweet'],
  data() {
    return { hello: 'world', replying_comment: null }
  },
  methods: {
    markdown,
    scaledAvatar,
    confirmDeleteComment(tweet, comment) {
      this.$modal.show('dialog', {
        title: '确定删除这条评论？',
        text: '此操作无法撤销',
        buttons: [
          {
            title: '取消',
            class: 'focus:outline-none py-3 hover:bg-gray-100 ',
            handler: () => {
              this.$modal.hide('dialog')
            },
          },
          {
            title: '确定',
            default: true,
            class: 'focus:outline-none py-3 hover:bg-gray-100',
            handler: () => {
              this.$modal.hide('dialog')
              window.delete(`/comments/${comment.id}`).then(res => {
                tweet.comments = tweet.comments.filter(c => c.id != comment.id)
              })
            },
          },
        ],
      })
    },
    updateNewCommentOf(e, tweet) {
      tweet.new_comment = e.target.innerText
    },
    commentContent(comment) {
      var content = comment.body
      if (comment.parent_comment) {
        content = `@${comment.parent_comment.user.nickname} ${content}`
      }
      return markdown(DOMPurify.sanitize(content))
    },
    isMyComment(comment) {
      return this.currentUser && this.currentUser.id === comment.user.id
    },
    showLoginDialog() {
      this.$modal.show(LoginDialog, {}, { height: 'auto', width: 400 })
    },
    postComment(tweet) {
      if (!this.currentUser) {
        this.showLoginDialog()
        return
      }
      if (!tweet.new_comment) return

      post('/comments', {
        tweet_id: tweet.id,
        body: tweet.new_comment,
        parent_comment_id: this.replying_comment ? this.replying_comment.id : null,
      }).then(data => {
        tweet.comments.unshift({
          user: this.currentUser,
          body: data.body,
          parent_comment: this.replying_comment,
        })
        this.replying_comment = null
        tweet.new_comment = ''
        this.$refs.tweet_comment_inbox.innerText = ''
      })
    },
  },
}
</script>

<style>
</style>
