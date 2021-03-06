<template lang="pug">
  .app(v-if="room")
    v-toolbar.primary.app-bar(dark, fixed)
      img(src="/img/nekokoro32.png")
      v-toolbar-title
        v-layout(row)
          v-avatar.mr-1(v-if="room && room.isLocked" size="24px")
            v-icon(dark) lock_outline
          div {{room && room.title}}
      v-spacer
      room-menu.mr-0
    transition(name="neko-slide")
      main(v-if="room")
        chat-tab(:members="members" :room="room" v-show="roomTab === '0'")
        memo-list(v-show="roomTab === '1'")
        character-tab(
          :room="room"
          v-show="roomTab === '2'"
        )
        map-tab(v-show="roomTab === '3'")
        dice-panel
        v-bottom-nav(
          color="white"
          :active.sync="roomTab"
          :class="{ 'no-shadow': roomTab !== '1' || roomTab !== '2' }"
          :fixed="true"
          :value="true"
        )
          v-btn(flat color="primary" value="0")
            span チャット
            v-icon mdi-forum
          v-btn(flat color="primary" value="1")
            span 共有メモ
            v-icon mdi-note-multiple
          v-btn(flat color="primary" value="2")
            span キャラクター
            v-icon mdi-account-multiple
          v-btn(flat color="primary" value="3")
            span マップ
            v-icon mdi-map-marker-radius
      loading(v-else)
</template>

<script>
import _ from 'lodash';
import { mapGetters } from 'vuex';
import { UnauthorizedError, NotFoundError } from '../backend/Backend';
import config from '../config';
import * as RouteNames from '../constants/route';
import sessionStorage from '../utilities/sessionStorage';
import { IntervalTimer } from '../utilities/timer';
import DicePanel from '@/browser/moleculers/DicePanel.vue';
import Loading from '@/browser/atoms/Loading.vue';
import MapTab from '@/browser/organisims/MapTab.vue';
import MemoList from '@/browser/components/MemoList.vue';
import RoomMenu from '@/browser/components/RoomMenu.vue';
import CharacterTab from '@/browser/organisims/CharacterTab.vue';
import ChatTab from '@/browser/organisims/ChatTab.vue';
import run from '@/browser/task';
import { bindAsObject } from '@/browser/models';

const saveRoomTab = _.debounce((roomId, roomTab) => {
  sessionStorage.setItem(`nekotaku:${roomId}:roomTab`, roomTab);
}, 1000);

export default {
  mixins: [
    bindAsObject('members', false),
    bindAsObject('room', false),
  ],
  components: {
    CharacterTab,
    ChatTab,
    DicePanel,
    Loading,
    MapTab,
    MemoList,
    RoomMenu,
    RouteNames,
  },
  data() {
    return {
      drawer: false,
      roomTab: '0',
      prevRoomTab: '0',
      timer: null,
    };
  },
  computed: {
    ...mapGetters([
      'chatControl',
    ]),
    title() {
      return this.room ? `${this.room.title} - ${config.title}` : config.title;
    },
  },
  methods: {
    async updateMember() {
      const {
        name,
        color,
      } = this.chatControl;
      await this.$models.members.update(this.roomId, { name, color });
    },
  },
  watch: {
    room({ id }) {
      document.title = this.title;

      const roomTab = sessionStorage.getItem(`nekotaku:${id}:roomTab`);
      if (roomTab) this.roomTab = roomTab;
    },
    roomTab(roomTab, oldValue) {
      this.prevRoomTab = oldValue;

      saveRoomTab(this.roomId, roomTab);
    },
  },
  created() {
    run(async () => {
      try {
        await this.updateMember();
        await this.bindModels();
        this.width = window.innerWidth;
        this.timer = new IntervalTimer(() => this.updateMember(), 5 * 1000);
      } catch (e) {
        if (e instanceof UnauthorizedError) this.$router.push({ name: RouteNames.RoomPassword });
        else if (e instanceof NotFoundError) this.$router.push({ name: RouteNames.NotFound });
        else {
          console.error(e);
          this.$router.push({ name: RouteNames.Lobby });
        }
      }
    });
  },
  destroyed() {
    if (this.timer) this.timer.stop();
  },
};
</script>
