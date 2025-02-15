<script setup lang="ts">
import { CommentSection, Post } from "./../models/models";
import { defineProps, ref, nextTick, computed } from "vue";
import { parseComment, parseCommentSection, parseCommentSectionByTypeAndObjectId, parsePost } from "../utils/parsers";
import PostCard from "./components/postCard.vue";
import TopBar from "./components/topBar.vue";
import { useRouter } from "vue-router";
import { useStore } from "vuex";
const router = useRouter()
const store = useStore()
const path = router.currentRoute.value.fullPath

const props = defineProps({ 
  type: { type: Number, required: true },
  objectId: { type: String, required: true },
  rootId: { type: String, required: true },
  postId: { type: String, required: true },
  targetId: { type: String, required: true },
  threadId: { type: String, required: false, default: '0' }
});

const commentSection = ref<CommentSection>(null)
const abovePosts = ref<Post[]>([])
const mainPost = ref<Post>(null)
const belowPosts = ref<Post[]>([])

const loaded = ref(false)
const mainPostDiv = ref(null)

const containerHeightPixels = ref(0)
const containerHeight = computed(() => containerHeightPixels.value + 'px')

fetch(`https://api.bilibili.com/x/v2/reply/detail?type=${props.type}&oid=${props.objectId}&root=${props.targetId}`)
  .then((res) => res.json())
  .then((data) => {
    console.log(data)
    const fetchedCommentSection: CommentSection = parseCommentSectionByTypeAndObjectId(data.data, props.type, props.objectId)
    const fetchedComments = fetchedCommentSection.comments
    fetchedComments.forEach(it => store.commit('cachePost', it))
    commentSection.value = fetchedCommentSection

    mainPost.value = fetchedComments[0]
    belowPosts.value = fetchedComments.slice(1)

    console.log(mainPost.value)
    
    const abovePost = store.getters.getCachedPost(props.postId)
    if (abovePost) {
      return Promise.resolve(abovePost)
    } else {
      return fetch(`https://api.vc.bilibili.com/dynamic_svr/v1/dynamic_svr/get_dynamic_detail?dynamic_id=${props.postId}`)
        .then((res) => res.json())
        .then((data) => {
          const fetchedPost = parsePost(data.data.card)
          store.commit('cachePost', fetchedPost)

          return Promise.resolve(fetchedPost)
        });
    }
  })
  .then((postObject) => {
    abovePosts.value.push(postObject)
    // Load root comment
    if (!props.rootId || props.rootId === props.targetId) {
      return Promise.resolve(null)
    } else {
      return fetch(`https://api.bilibili.com/x/v2/reply/detail?type=${props.type}&oid=${props.objectId}&root=${props.rootId}`)
        .then((res) => res.json())
        .then((data) => {
          const fetchedComment = parseComment(data.data.root, props.type, props.objectId)
          store.commit('cachePost', fetchedComment)

          return Promise.resolve(fetchedComment)
        });
    }
  })
  .then((rootCommentObject) => {
    if (rootCommentObject) {
      abovePosts.value.push(rootCommentObject)
    }
    if (props.threadId !== '0') {
      // Load up the entire thread
      return fetch(`http://api.bilibili.com/x/v2/reply/dialog/cursor?type=${props.type}&oid=${props.objectId}&root=${props.rootId}&dialog=${props.threadId}&size=9999`)
          .then((res) => res.json())
          .then((data) => {
            const fetchedCommentSection = parseCommentSection(data.data, abovePosts.value[0])
            fetchedCommentSection.comments.forEach(it => store.commit('cachePost', it))

            return Promise.resolve(fetchedCommentSection)
          });
    } else {
      return Promise.resolve(null)
    }
  })
  .then((threadObject?: CommentSection) => {
    if (threadObject) {
      belowPosts.value = []

      let addBelow = false
      for (let comment of threadObject.comments) {
        if (addBelow) {
          belowPosts.value.push(comment)
        } else if (comment.id !== props.targetId) {
          abovePosts.value.push(comment)
        }
        if (comment.id === props.targetId) {
          addBelow = true
        }
      }
    }

    // Done
    loaded.value = true
    nextTick(() => {
      containerHeightPixels.value = mainPostDiv.value.offsetTop - 53 * 2 + window.innerHeight
      console.log(mainPostDiv.value.offsetTop - 53)
      nextTick(() => {
        window.scrollTo(0, mainPostDiv.value.offsetTop - 53) // TODO: Change hardcode?
      })
    })
  })

const loadMoreComments = async $state => {
  if (router.currentRoute.value.fullPath != path) {
    $state.loaded()
    return;
  }

  console.log("Loading more comments");
  console.log(commentSection.value)
  try {
    const response = await fetch(
      `https://api.bilibili.com/x/v2/reply/detail?type=${props.type}&oid=${props.objectId}&root=${props.targetId}&next=${commentSection.value.cursor.next}`
    );
    const data = await response.json()
    if (data.code != 0) {
      console.log('Error!')
      console.log(data)
      $state.error()
    } else {
      commentSection.value = parseCommentSectionByTypeAndObjectId(data.data, props.type, props.objectId)

      const fetchedComments = commentSection.value.comments.slice(1)
      fetchedComments.forEach(it => store.commit('cachePost', it))
      belowPosts.value.push(...fetchedComments)
      if (commentSection.value.comments.length > 1) {
        $state.loaded();
      } else {
        $state.complete()
      }
    }
  } catch (error) {
    $state.error()
  }
};

</script>

<template>
  <div class="flex flex-column">
    <TopBar title="评论" />
    <div class="comment-page-container flex flex-column">
      <div v-show="loaded">
        <ul class="post-ul">
          <li v-for="post in abovePosts" :key="post.id" class="post-li">
            <PostCard :postId="post.id" :link="true" /> 
          </li>
          <li class="post-li" v-if="mainPost" ref="mainPostDiv">
            <PostCard :postId="mainPost.id" :large="true" /> 
          </li>
        </ul>
        <div v-if="belowPosts">
          <ul class="post-ul">
            <li v-for="post in belowPosts" :key="post.id" class="post-li">
              <PostCard :postId="post.id" :link="true" /> 
            </li>
          </ul>
          <InfiniteLoading v-if="loaded" :belowPosts="belowPosts" @infinite="loadMoreComments" class="flex justify-content-center py-4">
            <template #complete>
              &nbsp;
            </template>
          </InfiniteLoading>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.comment-page-container {
  min-height: v-bind(containerHeight);
}
</style>