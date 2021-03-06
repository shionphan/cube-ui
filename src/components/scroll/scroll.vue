<template>
  <div ref="wrapper" class="cube-scroll-wrapper">
    <div class="cube-scroll-content">
      <div ref="listWrapper" class="cube-scroll-list-wrapper">
        <slot>
          <ul class="cube-scroll-list">
            <li
              class="cube-scroll-item border-bottom-1px"
              v-for="(item, index) in data"
              :key="index"
              @click="clickItem(item)">{{item}}</li>
          </ul>
        </slot>
      </div>
      <slot name="pullup" :pullUpLoad="pullUpLoad" :isPullUpLoad="isPullUpLoad">
        <div class="cube-pullup-wrapper" v-if="pullUpLoad">
          <div class="before-trigger" v-if="!isPullUpLoad">
            <span>{{ pullUpTxt }}</span>
          </div>
          <div class="after-trigger" v-else>
            <loading></loading>
          </div>
        </div>
      </slot>
    </div>
    <slot
      name="pulldown"
      :pullDownRefresh="pullDownRefresh"
      :pullDownStyle="pullDownStyle"
      :beforePullDown="beforePullDown"
      :isPullingDown="isPullingDown"
      :bubbleY="bubbleY">
      <div class="cube-pulldown-wrapper" :style="pullDownStyle" v-if="pullDownRefresh">
        <div class="before-trigger" v-if="beforePullDown">
          <bubble :y="bubbleY"></bubble>
        </div>
        <div class="after-trigger" v-else>
          <div v-if="isPullingDown" class="loading">
            <loading></loading>
          </div>
          <div v-else><span>{{ refreshTxt }}</span></div>
        </div>
      </div>
    </slot>
  </div>
</template>

<script type="text/ecmascript-6">
  import BScroll from 'better-scroll'
  import Loading from '../loading/loading.vue'
  import Bubble from '../bubble/bubble.vue'
  import scrollMixin from '../../common/mixins/scroll'
  import { getRect } from '../../common/helpers/dom'
  import { camelize, kebab } from '../../common/lang/string'
  import { tip } from '../../common/helpers/debug'

  const COMPONENT_NAME = 'cube-scroll'
  const DIRECTION_H = 'horizontal'
  const DIRECTION_V = 'vertical'
  const DEFAULT_REFRESH_TXT = 'Refresh success'
  const PULL_DOWN_ELEMENT_INITIAL_HEIGHT = -50

  const EVENT_CLICK = 'click'
  const EVENT_PULLING_DOWN = 'pulling-down'
  const EVENT_PULLING_UP = 'pulling-up'

  const EVENT_SCROLL = 'scroll'
  const EVENT_BEFORE_SCROLL_START = 'before-scroll-start'
  const EVENT_SCROLL_END = 'scroll-end'

  const SCROLL_EVENTS = [EVENT_SCROLL, EVENT_BEFORE_SCROLL_START, EVENT_SCROLL_END]

  const DEFAULT_OPTIONS = {
    observeDOM: true,
    click: true,
    probeType: 1,
    scrollbar: false,
    pullDownRefresh: false,
    pullUpLoad: false
  }

  export default {
    name: COMPONENT_NAME,
    mixins: [scrollMixin],
    props: {
      data: {
        type: Array,
        default() {
          return []
        }
      },
      scrollEvents: {
        type: Array,
        default() {
          return []
        },
        validator(arr) {
          return arr.every((item) => {
            return SCROLL_EVENTS.indexOf(item) !== -1
          })
        }
      },
      // TODO: plan to remove at 1.10.0
      listenScroll: {
        type: Boolean,
        default: false
      },
      listenBeforeScroll: {
        type: Boolean,
        default: false
      },
      direction: {
        type: String,
        default: DIRECTION_V
      },
      refreshDelay: {
        type: Number,
        default: 20
      }
    },
    data() {
      return {
        beforePullDown: true,
        isPullingDown: false,
        isPullUpLoad: false,
        pullUpDirty: true,
        bubbleY: 0,
        pullDownStyle: ''
      }
    },
    computed: {
      pullDownRefresh() {
        return this.options.pullDownRefresh
      },
      pullUpLoad() {
        return this.options.pullUpLoad
      },
      pullUpTxt() {
        const pullUpLoad = this.pullUpLoad
        const txt = pullUpLoad && pullUpLoad.txt
        const moreTxt = (txt && txt.more) || ''
        const noMoreTxt = (txt && txt.noMore) || ''

        return this.pullUpDirty ? moreTxt : noMoreTxt
      },
      refreshTxt() {
        const pullDownRefresh = this.pullDownRefresh
        return (pullDownRefresh && pullDownRefresh.txt) || DEFAULT_REFRESH_TXT
      },
      finalScrollEvents() {
        const finalScrollEvents = this.scrollEvents.slice()

        if (!finalScrollEvents.length) {
          this.listenScroll && finalScrollEvents.push(EVENT_SCROLL)
          this.listenBeforeScroll && finalScrollEvents.push(EVENT_BEFORE_SCROLL_START)
        }
        return finalScrollEvents
      }
    },
    watch: {
      data() {
        setTimeout(() => {
          this.forceUpdate(true)
        }, this.refreshDelay)
      },
      pullDownRefresh: {
        handler(newVal, oldVal) {
          if (newVal) {
            this.scroll.openPullDown(newVal)
            if (!oldVal) {
              this._onPullDownRefresh()
              this._calculateMinHeight()
            }
          }

          if (!newVal && oldVal) {
            this.scroll.closePullDown()
            this._offPullDownRefresh()
            this._calculateMinHeight()
          }
        },
        deep: true
      },
      pullUpLoad: {
        handler(newVal, oldVal) {
          if (newVal) {
            this.scroll.openPullUp(newVal)
            if (!oldVal) {
              this._onPullUpLoad()
              this._calculateMinHeight()
            }
          }

          if (!newVal && oldVal) {
            this.scroll.closePullUp()
            this._offPullUpLoad()
            this._calculateMinHeight()
          }
        },
        deep: true
      }
    },
    activated() {
      /* istanbul ignore next */
      this.enable()
    },
    deactivated() {
      /* istanbul ignore next */
      this.disable()
    },
    mounted() {
      this.$nextTick(() => {
        this.initScroll()
      })
      this._checkDeprecated()
    },
    beforeDestroy() {
      this.destroy()
    },
    methods: {
      initScroll() {
        if (!this.$refs.wrapper) {
          return
        }
        this._calculateMinHeight()

        let options = Object.assign({}, DEFAULT_OPTIONS, {
          scrollY: this.direction === DIRECTION_V,
          scrollX: this.direction === DIRECTION_H,
          probeType: this.scrollEvents.indexOf(EVENT_SCROLL) !== -1 ? 3 : 1
        }, this.options)

        this.scroll = new BScroll(this.$refs.wrapper, options)

        this._listenScrollEvents()

        if (this.pullDownRefresh) {
          this._onPullDownRefresh()
        }

        if (this.pullUpLoad) {
          this._onPullUpLoad()
        }
      },
      disable() {
        this.scroll && this.scroll.disable()
      },
      enable() {
        this.scroll && this.scroll.enable()
      },
      refresh() {
        this._calculateMinHeight()
        this.scroll && this.scroll.refresh()
      },
      destroy() {
        this.scroll && this.scroll.destroy()
        this.scroll = null
      },
      scrollTo() {
        this.scroll && this.scroll.scrollTo.apply(this.scroll, arguments)
      },
      scrollToElement() {
        this.scroll && this.scroll.scrollToElement.apply(this.scroll, arguments)
      },
      clickItem(item) {
        this.$emit(EVENT_CLICK, item)
      },
      forceUpdate(dirty = false) {
        if (this.pullDownRefresh && this.isPullingDown) {
          this.isPullingDown = false
          this._reboundPullDown().then(() => {
            this._afterPullDown(dirty)
          })
        } else if (this.pullUpLoad && this.isPullUpLoad) {
          this.isPullUpLoad = false
          this.scroll.finishPullUp()
          this.pullUpDirty = dirty
          dirty && this.refresh()
        } else {
          dirty && this.refresh()
        }
      },
      resetPullUpTxt() {
        this.pullUpDirty = true
      },
      _listenScrollEvents() {
        this.finalScrollEvents.forEach((event) => {
          this.scroll.on(camelize(event), (...args) => {
            this.$emit(event, ...args)
          })
        })
      },
      _calculateMinHeight() {
        if (this.$refs.listWrapper) {
          this.$refs.listWrapper.style.minHeight = this.pullDownRefresh || this.pullUpLoad ? `${getRect(this.$refs.wrapper).height + 1}px` : 0
        }
      },
      _onPullDownRefresh() {
        this.scroll.on('pullingDown', this._pullDownHandle)
        this.scroll.on('scroll', this._pullDownScrollHandle)
      },
      _offPullDownRefresh() {
        this.scroll.off('pullingDown', this._pullDownHandle)
        this.scroll.off('scroll', this._pullDownScrollHandle)
      },
      _pullDownHandle() {
        if (this.resetPullDownTimer) {
          clearTimeout(this.resetPullDownTimer)
        }
        this.beforePullDown = false
        this.isPullingDown = true
        this.$emit(EVENT_PULLING_DOWN)
      },
      _pullDownScrollHandle(pos) {
        if (this.beforePullDown) {
          this.bubbleY = Math.max(0, pos.y + PULL_DOWN_ELEMENT_INITIAL_HEIGHT)
          this.pullDownStyle = `top:${Math.min(pos.y + PULL_DOWN_ELEMENT_INITIAL_HEIGHT, 10)}px`
        } else {
          this.bubbleY = 0
          this.pullDownStyle = `top:${Math.min(pos.y - 30, 10)}px`
        }
      },
      _onPullUpLoad() {
        this.scroll.on('pullingUp', this._pullUpHandle)
      },
      _offPullUpLoad() {
        this.scroll.off('pullingUp', this._pullUpHandle)
      },
      _pullUpHandle() {
        this.isPullUpLoad = true
        this.$emit(EVENT_PULLING_UP)
      },
      _reboundPullDown() {
        const {stopTime = 600} = this.pullDownRefresh
        return new Promise((resolve) => {
          setTimeout(() => {
            this.scroll.finishPullDown()
            resolve()
          }, stopTime)
        })
      },
      _afterPullDown(dirty) {
        this.resetPullDownTimer = setTimeout(() => {
          this.pullDownStyle = `top:${PULL_DOWN_ELEMENT_INITIAL_HEIGHT}px`
          this.beforePullDown = true
          dirty && this.refresh()
        }, this.scroll.options.bounceTime)
      },
      _checkDeprecated() {
        const deprecatedKeys = ['listenScroll', 'listenBeforeScroll']
        deprecatedKeys.forEach((key) => {
          this[key] && tip(`The property "${kebab(key)}" is deprecated, please use the recommended property "scroll-events" to replace it. Details could be found in https://didi.github.io/cube-ui/#/en-US/docs/scroll#cube-Propsconfiguration-anchor`, COMPONENT_NAME)
        })
      }
    },
    components: {
      Loading,
      Bubble
    }
  }
</script>

<style lang="stylus" rel="stylesheet/stylus">
  @require "../../common/stylus/variable.styl"

  .cube-scroll-wrapper
    position: relative
    height: 100%
    overflow: hidden

  .cube-pulldown-wrapper
    position: absolute
    width: 100%
    left: 0
    display: flex
    justify-content: center
    align-items: center
    transition: all
    .after-trigger
      margin-top: 5px

  .cube-pullup-wrapper
    width: 100%
    display: flex
    justify-content: center
    align-items: center
    .before-trigger
      padding: 22px 0
      min-height: 1em
    .after-trigger
      padding: 19px 0

  .cube-scroll-content
    position: relative
    z-index: 1

  .cube-scroll-item
    height: 60px
    line-height: 60px
    font-size: $fontsize-large-x
    padding-left: 20px
</style>
