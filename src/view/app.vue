<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import IconButton from './components/iconButton.vue';
import { useRouter } from 'vue-router'
import { useStore } from 'vuex';
import { fetchAndParseUserProfile } from '../utils/appRequests';
import { FollowingGroup, User } from '../models/models';
import { range } from '../utils/helpers';
const router = useRouter()
const store = useStore()
const mainColumnComponent = ref(null)

const authenticated = ref(false)

chrome.cookies
  .get({ name: "DedeUserID", url: "https://bilibili.com" })
  .then((cookie) => {
    store.commit('setUser', { uid: cookie.value });
    return chrome.cookies.get({ name: "bili_jct", url: "https://bilibili.com" })
  })
  .then((cookie) => {
    store.commit('setCsrf', cookie.value);
    return fetchAndParseUserProfile(store.state.user.uid)
  })
  .then((user: User) => {
    store.commit('setUser', user)
  })
  .then(() => {
    authenticated.value = true
    fetchFollowingGroups()
  })

const fetchFollowingGroups = () => {
  fetch("https://api.bilibili.com/x/relation/tags?jsonp=jsonp").then(res => res.json())
    .then((data) => {
      let groups = data.data.filter(it => it.tagid != 0).map(it => ({ id: it.tagid, name: it.name, count: it.count }))
      return Promise.all(groups.map(fetchFollowingGroup))
    })
    .then((groups: any[]) => {
      groups.unshift({ default: true, id: '0', name: '所有关注', count: store.state.user.followingCount })

      console.log("Fetched following groups")
      console.log(groups)
      store.commit('setUser', { ...store.state.user, followingGroups: groups })

      selectedGroup.value = groups[0]
    })
}

const fetchFollowingGroup = (group: FollowingGroup): Promise<FollowingGroup> => {
  group.users = new Map()
  const requests = range(1, Math.ceil(group.count / 50))
    .map(page => fetch(`https://api.bilibili.com/x/relation/tag?mid=${store.state.user.uid}&tagid=${group.id}&pn=${page}&ps=50&jsonp=jsonp`)
                  .then(res => res.json())
                  .then(data => data.data.map(it => ({ uid: it.mid, name: it.uname, avatarUrl: it.face, bio: it.sign })))
                  .then(users => users.forEach(user => group.users.set(user.uid, user))))
  return Promise.all(requests).then(() => group)
}

onMounted(async () => {
  while (!document.getElementById('mainColumn')) {
    await new Promise(r => setTimeout(r, 10));
  }
  // Disable middle click scrolling
  // document.getElementById('mainColumn').onmousedown = function(e) { if (e.button === 1) return false; }
})

const onHome = () => {
  if (router.currentRoute.value.path == '/') {
    mainColumnComponent.value.refresh()
  }
  router.push('/')
}
const path = computed(() => router.currentRoute.value.path)
const selectedGroup = ref<FollowingGroup>()
</script>

<template>
  <div class="flex flex-row justify-content-center">
    <div class="flex flex-column navigation align-items-center justify-content-center">
      <IconButton src="logo.png" hoverBackgroundColor="rgba(251, 114, 153, 0.1)" activeBackgroundColor="rgba(251, 114, 153, 0.1)" :size="50" :padding="4" @click="onHome()" class="navigation-icon" />
      <IconButton :icon="['fas', 'home']" :color="path === '/' ? '#0f1419' : 'rgb(207, 217, 222)'" hoverBackgroundColor="rgba(15, 20, 25, 0.1)" activeBackgroundColor="rgba(15, 20, 25, 0.1)" :size="50" :font-size="22" @click="onHome()" class="navigation-icon" />
      <IconButton v-if="store.state.user" :icon="['fas', 'user']" :color="path === `/profile/${store.state.user.uid}` ? '#0f1419' : 'rgb(207, 217, 222)'" hoverBackgroundColor="rgba(15, 20, 25, 0.1)" activeBackgroundColor="rgba(15, 20, 25, 0.1)" :size="50" :font-size="22" @click="router.push(`/profile/${store.state.user.uid}`)" class="navigation-icon" />
    </div>
    <router-view v-if="authenticated" id="mainColumn" class="flex main-column" v-slot="{ Component }">
      <keep-alive>
        <component :key="$route.fullPath" :is="Component" ref="mainColumnComponent" :groupFilter="(selectedGroup == null || selectedGroup.default) ? null : selectedGroup" />
      </keep-alive>
    </router-view>
    <div class="flex flex-column sidebar">
      <div v-if="selectedGroup && path === `/`" class="sidebar-item">
        <div class="sidebar-item-header">
          我的关注
        </div>
        <div style="padding: 12px 16px 0;">
          <div v-for="group in store.state.user.followingGroups" :key="group.id" class="field-radiobutton">
            <RadioButton :id="group.id" name="group" :value="group" v-model="selectedGroup" />
            <label :for="group.id">{{ group.name }} <span class="sidebar-secondary-text">({{ group.count }})</span></label>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
html {
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
.main-column {
  width: 600px;
  border-style: solid;
  border-width: 0;
  border-left-width: 1px;
  border-left-color: rgb(239, 243, 244);
  border-right-width: 1px;
  border-right-color: rgb(239, 243, 244);
}
.navigation {
  width: 64px;
  margin: 0 12px;
  position: sticky;
  top: 0;
  height: fit-content;
}
.navigation-icon {
  margin: 4px 0;
}
.sidebar {
  color: #0F1419;
  margin-left: 20px;
  width: 288px;
  padding-top: 4px;
  position: sticky;
  top: 0;
  height: fit-content;
}
.sidebar-item {
  background: #F7F9F9;
  border-radius: 16px;
  margin-bottom: 16px;
}
.sidebar-item-header {
  font-size: 20px;
  line-height: 25px;
  padding: 12px 16px;
  font-weight: bold;
}
.sidebar-secondary-text {
  color: rgb(83, 100, 113);
}
.field-radiobutton label {
  font-size: 15px;
  line-height: 20px;
}
</style>
