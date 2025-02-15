<script setup lang="ts">
import { defineProps, PropType, ref } from "vue"
import { CommentSection, CommentSectionSort, Post } from "../../models/models";
import { parseCommentSection, parsePost, parseCommentTypeAndObjectId } from "../../utils/parsers"
import PostCard from "./postCard.vue"
import { useRouter } from "vue-router"
import { useStore } from "vuex";
const router = useRouter()
const store = useStore()
const path = router.currentRoute.value.fullPath

const props = defineProps({ 
  postId: { type: String, required: true },
  sort: { type: Number as PropType<CommentSectionSort>, required: false, default: CommentSectionSort.Hot }
});

const commentSection = ref<CommentSection>(null)
const comments = ref<Post[]>([])

let post: Post = null
let type = -1
let oid: any = null

fetch(`https://api.vc.bilibili.com/dynamic_svr/v1/dynamic_svr/get_dynamic_detail?dynamic_id=${props.postId}`)
    .then((res) => res.json())
    .then((data) => {
        post = parsePost(data.data.card)
        const results = parseCommentTypeAndObjectId(post)
        type = results[0]
        oid = results[1]

        return fetch(`https://api.bilibili.com/x/v2/reply/main?jsonp=jsonp&next=0&type=${type}&oid=${oid}&mode=${props.sort}&plat=1`)
    })
    .then((res) => res.json())
    .then((data) => {
        commentSection.value = parseCommentSection(data.data, post)

        const fetchedComments = commentSection.value.comments
        fetchedComments.forEach(it => store.commit('cachePost', it))
        comments.value.push(...fetchedComments)
        console.log(commentSection.value)
    });

const onShowReplies = (comment: Post) => {
    router.push({ name: 'comment', params: { type: comment.commentType, objectId: comment.commentObjectId, rootId: comment.commentRootId, targetId: comment.id, threadId: comment.commentThreadId }})
}

const loadMoreComments = async $state => {
  if (router.currentRoute.value.fullPath != path) {
    $state.loaded()
    return;
  }

  console.log("Loading more comments");
  console.log(commentSection.value)
  try {
    const response = await fetch(`https://api.bilibili.com/x/v2/reply/main?jsonp=jsonp&next=${commentSection.value.cursor.next}&type=${type}&oid=${oid}&mode=${props.sort}&plat=1`);
    const data = await response.json()
    if (data.code != 0) {
      console.log('Error!')
      console.log(data)
      $state.error()
    } else {
      commentSection.value = parseCommentSection(data.data, post)
      
      const fetchedComments = commentSection.value.comments
      fetchedComments.forEach(it => store.commit('cachePost', it))
      comments.value.push(...fetchedComments)
      if (commentSection.value.comments.length > 0) {
        $state.loaded()
      } else {
        $state.complete()
      }
    }
  } catch (error) {
    console.log(error)
    $state.error()
  }
};

</script>

<template>
  <div class="comment-section-container">
    <div v-if="comments">
        <div v-for="comment in comments" :key="comment.id" class="post-border">
            <ul class="post-ul">
                <li class="post-li">
                    <PostCard :postId="comment.id" :resolvePostId="false" :link="true" /> 
                </li>
                <li v-for="chainedComment in comment.chainedPosts" :key="chainedComment.id" class="post-li">
                    <PostCard :postId="chainedComment.id" :resolvePostId="false" :link="true" />
                </li>
                <div v-if="comment.commentCount > comment.chainedPosts?.length ?? 0" class="post-show-replies flex flex-row px-3 justify-content-center align-items-center" @click="onShowReplies(comment)">
                    <div class="flex flex-column justify-content-between align-items-center" style="width: 48px; height: 16px;">
                      <div class="post-show-replies-dot" />
                      <div class="post-show-replies-dot" />
                      <div class="post-show-replies-dot" />
                    </div>
                    <div style="flex: 1; padding-left: 14px;">
                      查看更多
                    </div>
                </div>
            </ul>
        </div>
        <InfiniteLoading v-if="comments.length > 0" :comments="comments" @infinite="loadMoreComments" class="flex justify-content-center py-4">
            <template #complete>
              <div style="padding-bottom: 100%;" />
            </template>
        </InfiniteLoading>
    </div>
  </div>
</template>

<style scoped>
.post-container {
  width: 100%;
  padding: 12px 16px;
}
.post-container-link {
  transition: background-color 0.2s ease-out 0s;
  cursor: pointer;
}
.post-container-link:hover {
  background-color: rgba(0, 0, 0, 0.03);
}
.post-show-replies {
  transition: background-color 0.2s ease-out 0s;
  height: 36px;
  font-size: 15px;
  color: rgb(251, 114, 153);
  cursor: pointer;
}
.post-show-replies:hover {
  background-color: rgba(0, 0, 0, 0.03);
}
.post-show-replies-dot {
  height: 2px;
  width: 2px;
  background-color: rgb(207, 217, 222);
}
.post-username {
  font-size: 15px;
  line-height: 20px;
  font-weight: bold;
  color: #0f1419;
}
.post-timestamp {
  font-size: 15px;
  line-height: 20px;
  color: #536471;
}
.post-stats {
  font-size: 15px;
  line-height: 20px;
  border-top-color: rgb(239, 243, 244);
  border-top-width: 1px;
  border-top-style: solid;
  color: #536471;
}
.post-actions {
  font-size: 22px;
  border-top-color: rgb(239, 243, 244);
  border-top-width: 1px;
  border-top-style: solid;
  color: rgb(83, 100, 113);
}
.post-stats-number {
  color: #0f1419;
  font-weight: bold;
}
.post-text {
  font-size: 15px;
  line-height: 20px;
  color: #0f1419;
}
.post-text.large {
  font-size: 23px;
  line-height: 28px;
  color: #0f1419;
}
.post-single-image-container {
}
.post-single-image-container img {
  border-color: rgb(207, 217, 222);
  border-width: 1px;
  border-style: solid;
  border-radius: 16px;
  max-width: 100%;
}
.post-double-image-container {
}
.post-double-image-container.left img {
  border-color: rgb(207, 217, 222);
  border-width: 1px;
  border-style: solid;
  border-radius: 16px;
  max-width: 100%;
  border-right: none;
  border-radius: 16px 0 0 16px;
  width: 251px;
  height: 284px;
}
.post-double-image-container.right img {
  padding-left: 2px;
  border-color: rgb(207, 217, 222);
  border-width: 1px;
  border-style: solid;
  border-radius: 16px;
  max-width: 100%;
  border-left: none;
  border-radius: 0 16px 16px 0;
  width: 251px;
  height: 284px;
}
.post-triple-image-container {
  height: 168px;
}
.post-triple-image-container.large {
  height: 188px;
}
.post-triple-image-container img {
  border-color: rgb(207, 217, 222);
  border-width: 1px;
  border-style: solid;
  border-radius: 16px;
  max-width: 100%;
  width: 168px;
  height: 168px;
}
.post-triple-image-container.large img {
  width: 188px;
  height: 188px;
}
.post-video-container {
  border-color: rgb(207, 217, 222);
  border-width: 1px;
  border-style: solid;
  border-radius: 16px;
}
.post-column-container {
  border-color: rgb(207, 217, 222);
  border-width: 1px;
  border-style: solid;
  border-radius: 16px;
}
.post-embedded-post-container {
  border-color: rgb(207, 217, 222);
  border-width: 1px;
  border-style: solid;
  border-radius: 16px;
}
</style>