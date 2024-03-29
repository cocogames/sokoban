<template>
  <view class="game-page">
    <u-notify ref="notify"></u-notify>
    <game-top-bar
      :gameMap="gameMap"
      :step="gameCore.step"
      :life="gameCore.life"
      :status="gameCore.status"
      :showEdit="routeInfo.type === 'create'"
      :showLike="routeInfo.type === 'workshop' && routeInfo.id !== 'undefined'"
      :showUpload="routeInfo.type === 'workshop' && routeInfo.localId !== 'undefined'"
      @reset="!gameCore.onSlid && regret()"
      @regret="!gameCore.onSlid && regret()"
      @showMenu="$refs.settings.show()"
      @upload="
        saveType = 1
        $refs.save.show()
      "
    ></game-top-bar>
    <game-content
      :staticMap="gameCore.staticMap"
      :activeMap="gameCore.activeMap"
      :direction="gameCore.moveDirection"
      :playerPostion="{ playerX: gameCore.playerX, playerY: gameCore.playerY }"
      :nightMode="gameMap.nightMode"
      ref="game"
    ></game-content>

    <analog-handle @moveFunc="moveFunc" :disabled="gameCore.disabledHandle"></analog-handle>

    <ar-popup type="menu" ref="settings">
      <game-menu></game-menu>
    </ar-popup>

    <ar-popup type="common" ref="tips" :isClosed="false">
      <game-result
        @reset="reset"
        @nextLevel="nextLevel"
        @saveServe="
          saveType = 1
          $refs.save.show()
        "
        @saveLocal="
          saveType = 2
          $refs.save.show()
        "
        :type="routeInfo.type"
      ></game-result>
    </ar-popup>

    <ar-popup ref="save">
      <ar-form :formOptions="formOptions" okText="保存" @ok="saveType === 1 ? saveServe() : saveLocal()" @formCreate="formCreate"></ar-form>
    </ar-popup>
  </view>
</template>

<script>
import GameContent from '@/components/game-content/game-content.vue'
import ArPopup from '@/components/ar-popup.vue'
import ArForm from '@/components/ar-form.vue'
import GameTopBar from './components/game-top-bar/game-top-bar.vue'
import GameMenu from './components/game-menu/game-menu.vue'
import GameResult from './components/game-result/game-result.vue'
import AnalogHandle from './components/analog-handle/analog-handle.vue'

import { basic, expand } from '@/static/js/level/index'
import { deepCloneObjArr } from '@/utils/index'
import { Move } from './utils/move'
import { formOptions } from './config/data'
export default {
  data() {
    return {
      basic: Object.freeze(basic),
      expand: Object.freeze(expand),
      formOptions: Object.freeze(formOptions),
      // 游戏地图
      gameMap: {
        mapData: [],
        mapName: '',
        level: 0,
        regretDisabled: 0, // 禁用撤回
        stepsPas: '**', // 最优步数
        nightMode: 0 // 黑夜模式
      },
      // 游戏核心
      gameCore: {
        staticMap: [[]],
        activeMap: [[]],
        playerX: 0,
        playerY: 0,
        setX: 0,
        setY: 0,
        setBoxX: 0,
        setBoxY: 0,
        staticTarget: 0,
        activeTarget: 0,
        staticBoxTarget: 0,
        activeBoxTarget: 0,
        life: 0,
        endCounter: 0,
        status: {
          poisoning: false,
          drunk: false
        },
        portalExit: [],
        step: 0,
        suc: 0,
        direction: -1,
        moveDirection: -1,
        processData: [],
        disabledHandle: false,
        onSlid: false
      },
      gameRecord: [], // 游戏记录
      routeInfo: {},
      saveType: 0 // 保存方案 1-云端，2-本地
    }
  },
  components: { GameContent, ArPopup, ArForm, GameTopBar, GameMenu, GameResult, AnalogHandle },
  onLoad(option) {
    this.routeInfo = option
  },
  onReady() {
    this.init()
  },
  methods: {
    async init() {
      // 重置初始gameCore
      this.gameCore = deepCloneObjArr(this.$options.data().gameCore)
      if (this.routeInfo.type === 'level') {
        if (this.routeInfo.pack === '0') {
          this.gameMap.mapData = deepCloneObjArr(basic[this.routeInfo.level])
          this.gameMap.mapName = `基础关--${Number(this.routeInfo.level) + 1}`
          this.gameCore.life = '**'
        } else {
          this.gameMap.mapData = deepCloneObjArr(expand[this.routeInfo.level].gameMap)
          this.gameMap.mapName = `拓展关--${Number(this.routeInfo.level) + 1}`
          this.gameCore.life = expand[this.routeInfo.level].life || '**'
        }
      } else if (this.routeInfo.type === 'workshop') {
        if (Number(this.routeInfo.id)) {
          const res = await this.$api.get(`map/${this.routeInfo.id}`)
          if (res.code === 0) {
            this.gameMap = res.data
            this.gameCore.life = res.data.playerHP || '**'
          }
        } else {
          const data = JSON.parse(uni.getStorageSync(`map${this.routeInfo.localId}`))
          this.gameMap.mapName = data.mapName
          this.gameMap.mapData = data.mapData
          this.gameCore.life = data.life || '**'
          this.gameMap.regretDisabled = data.regretDisabled || 0
          this.gameMap.nightMode = data.nightMode || 0
          this.form.mapName = this.gameMap.mapName
        }
      } else if (this.routeInfo.type === 'create') {
        const data = JSON.parse(uni.getStorageSync('createMap'))
        this.gameMap.mapData = data.mapData
        this.gameMap.mapName = '测试地图'
        this.gameCore.life = data.life || '**'
        this.gameMap.regretDisabled = data.regretDisabled
        this.gameMap.nightMode = data.nightMode || 0
      }
      const mapInit = {
        // 玩家坐标
        2: (x, y) => {
          this.gameCore.playerX = Number(x)
          this.gameCore.playerY = Number(y)
        },
        // 终点总数
        4: () => {
          this.gameCore.endCounter++
        },
        // 传送门出口
        11: (x, y) => {
          this.gameCore.portalExit.push({ x, y })
        }
      }
      for (let y = 0; y < this.gameMap.mapData.length; y++) {
        for (let x = 0; x < this.gameMap.mapData[y].length; x++) {
          if (mapInit[this.gameMap.mapData[y][x]]) mapInit[this.gameMap.mapData[y][x]](x, y)
        }
      }
      // 地图分层
      this.gameCore.staticMap = deepCloneObjArr(this.gameMap.mapData)
      this.gameCore.activeMap = deepCloneObjArr(this.gameMap.mapData)

      // 记录游戏记录
      this.gameRecord = []
      this.gameRecord.push(deepCloneObjArr(this.gameCore))
    },
    moveBeforeHook(direction) {
      return this.statusEvent(direction)
    },
    moveFunc(direction, disabledBeforeHook = false, moveCount = 1) {
      this.gameCore.setX = this.gameCore.playerX
      this.gameCore.setY = this.gameCore.playerY
      this.gameCore.setBoxX = this.gameCore.setX
      this.gameCore.setBoxY = this.gameCore.setY
      this.gameCore.direction = direction
      if (!disabledBeforeHook && !this.moveBeforeHook(0)) return
      if (this.gameCore.direction === 0) {
        this.gameCore.setY -= moveCount
        this.gameCore.setBoxY = this.gameCore.setY - 1
      } else if (this.gameCore.direction === 1) {
        this.gameCore.setX += moveCount
        this.gameCore.setBoxX = this.gameCore.setX + 1
      } else if (this.gameCore.direction === 2) {
        this.gameCore.setY += moveCount
        this.gameCore.setBoxY = this.gameCore.setY + 1
      } else if (this.gameCore.direction === 3) {
        this.gameCore.setX -= moveCount
        this.gameCore.setBoxX = this.gameCore.setX - 1
      }
      new Move(this.gameCore, direction, this.gameRecord, this.moveFunc)
      this.moveAfterHook()
    },
    moveAfterHook() {
      if (this.gameCore.suc === 1) return
      if (this.gameCore.life === 0) {
        setTimeout(() => {
          this.$refs.notify.show({ type: 'error', message: '你失败了~' })
          this.reset()
        }, 200)
      }
      setTimeout(() => {
        if (this.gameCore.suc === 1) {
          setTimeout(() => {
            if (!this.$refs.tips.isShow) {
              this.$refs.tips.show()
              this.passProcess()
            }
          }, 150)
        }
      }, 50)
      // 大地图追踪视角
      if (this.gameMap.mapData.length > 12 || this.gameMap.mapData[0].length > 12) {
        const screenScale = uni.getSystemInfoSync().windowWidth / 375
        this.$refs.game.scrollLeft = this.gameCore.playerX * screenScale * 30 - screenScale * 165
        this.$refs.game.scrollTop = this.gameCore.playerY * screenScale * 30 - screenScale * 165
      }
    },
    statusEvent() {
      let flag = true
      // 中毒事件
      if (this.gameCore.status.poisoning) {
        if (this.gameCore.direction === 0) this.gameCore.direction = 2
        else if (this.gameCore.direction === 1) this.gameCore.direction = 3
        else if (this.gameCore.direction === 2) this.gameCore.direction = 0
        else if (this.gameCore.direction === 3) this.gameCore.direction = 1
      }
      // 酗酒事件
      if (this.gameCore.status.drunk) {
        // 随机再进行0~2次移动
        const drunkStep = Math.floor(Math.random() * 3)
        const startRecordLength = this.gameRecord.length
        for (let i = 0; i < drunkStep; i++) {
          this.$nextTick(() => {
            if ([10, 11, 13, 14].includes(this.gameCore.staticTarget)) return // 碰到传送门/解药/滑块则停止
            if (this.gameCore.onSlid) return // 滑行中不触发
            this.moveFunc(this.gameCore.direction, true)
          })
        }
        // 清理酗酒移动途中的记录
        setTimeout(() => {
          const endRecordLength = this.gameRecord.length
          const totalStep = endRecordLength - startRecordLength
          this.gameRecord.splice(startRecordLength, totalStep - 1)
        }, 50)
      }
      return flag
    },
    reset() {
      this.gameCore = deepCloneObjArr(this.gameRecord[0])
      this.$refs.tips.isShow = false
    },
    regret() {
      if (this.gameMap.regretDisabled === 1) return this.$refs.notify.show({ type: 'error', message: '该图作者已禁用撤回' })
      if (this.gameRecord.length === 1) return this.$refs.notify.show({ type: 'error', message: '已经回退到头啦~' })
      this.gameRecord.pop()
      this.gameRecord[this.gameRecord.length - 1].moveDirection = -1
      this.gameCore = deepCloneObjArr(this.gameRecord[this.gameRecord.length - 1])
    },
    nextLevel() {
      this.routeInfo.level++
      this.init()
      this.$refs.tips.isShow = false
    },
    async passProcess() {
      if (this.routeInfo.type !== 'workshop' || this.routeInfo.id === 'undefined' || !this.$store.state.userInfo.username) return
      if (this.gameMap.stepsPas > this.gameCore.step) {
        const res = await this.$api.post('/map/stepsPas', {
          mapId: this.routeInfo.id,
          stepsPas: this.gameCore.step,
          processData: this.gameCore.processData
        })
        if (res.code === 0) {
          this.$refs.notify.show({ type: 'success', message: '您达成了新的最优步数，已将您的通关过程上传至云端' })
        }
      }
    },
    async saveServe() {
      const res = await this.$api.post('map/add', {
        ...this.gameMap,
        ...this.form,
        playerHP: this.$options.data().gameCore.life,
        stepsPas: this.gameCore.step,
        processData: this.gameCore.processData
      })
      if (res.code === 0) {
        this.$refs.notify.show({ type: 'success', message: '上传成功，3秒后将返回首页' })
        this.$refs.save.show()
        this.$refs.tips.show()
        setTimeout(() => {
          uni.redirectTo({ url: '/pages/index/index' })
        }, 3000)
      }
    },
    async saveLocal() {
      for (let i = 0; i < 99; i++) {
        if (i === 99) return this.$refs.notify.show({ type: 'error', message: '本地存储已达上限' })
        const res = uni.getStorageSync(`map${i}`)
        if (res) continue
        uni.setStorage({
          key: `map${i}`,
          data: JSON.stringify({
            localId: i,
            creator: this.form.creator || '匿名',
            mapName: this.form.mapName || '未命名',
            mapData: this.gameMap.mapData,
            playerHP: this.gameCore.life,
            time: new Date(),
            stepsPas: this.gameRecord.step,
            processData: this.gameCore.processData,
            regretDisabled: this.gameMap.regretDisabled
          })
        })
        this.$refs.notify.show({ type: 'success', message: '保存成功，3秒后将返回首页' })
        this.$refs.save.show()
        this.$refs.tips.show()
        setTimeout(() => {
          uni.redirectTo({ url: '/pages/index/index' })
        }, 3000)
        break
      }
    },
    formCreate(form) {
      this.form = form
    }
  }
}
</script>

<style lang="scss">
.game-page {
  height: 100vh;
  padding: 60rpx 0;
  background-image: url('https://source.aring.cc/assets/project/sokoban/imgs/common/bg.png');
  display: flex;
  flex-flow: column nowrap;
  .game-container {
    margin-top: 90rpx;
  }
  .game-action {
    padding: 0 40rpx;
    display: flex;
  }
}
</style>
