<template>
  <transition-group :tag="tagName" name="lazy-component" style="position: relative;"
    @before-enter="(el) => $emit('before-enter', el)"
    @before-leave="(el) => $emit('before-leave', el)"
    @after-enter="(el) => $emit('after-enter', el)"
    @after-leave="(el) => $emit('after-leave', el)"
  >
    <div v-if="isInit" key="component">
      <slot :loading="loading"></slot>
    </div>
    <div v-else-if="$slots.skeleton" key="skeleton">
      <slot name="skeleton"></slot>
    </div>
    <div v-else key="loading">
    </div>
  </transition-group>
</template>

<script>
  export default {
    name: 'VueLazyComponent',

    props: {
      timeout: {
        type: Number
      },
      tagName: {
        type: String,
        default: 'div'
      },
      viewport: {
        type: typeof window !== 'undefined' ? window.HTMLElement : Object,
        default: () => null
      },
      threshold: {
        type: String,
        default: '0px'
      },
      direction: {
        type: String,
        default: 'vertical'
      },
      maxWaitingTime: {
        type: Number,
        default: 50
      }
    },

    data () {
      return {
        isInit: false,
        timer: null,
        io: null,
        loading: false,
        isFeatureSupported:false
      }
    },

    created () {
      this.browserFeatureDetection();
      if (this.isFeatureSupported && this.timeout) {
        this.timer = setTimeout(() => {
          this.init()
        }, this.timeout)
      }
    },

    mounted () {
      this.browserFeatureDetection();
      if (this.isFeatureSupported && !this.timeout) {
        let rootMargin
        switch (this.direction) {
          case 'vertical':
            rootMargin = `${this.threshold} 0px`
            break
          case 'horizontal':
            rootMargin = `0px ${this.threshold}`
            break
        }

        this.io = new window.IntersectionObserver(this.intersectionHandler, {
          rootMargin,
          root: this.viewport,
          threshold: [ 0, Number.MIN_VALUE, 0.01]
        })
        this.io.observe(this.$el)
      }
    },

    beforeDestroy () {
      if (this.isFeatureSupported && this.io) {
        this.io.unobserve(this.$el)
      }
    },

    methods: {
      browserFeatureDetection(){
        // Safari browser does, at time of writing 28-11-2018 not support IntersectionObserver.
        // The fallback is that we just disable lazy loading.
        	if (!'IntersectionObserver' in window && !'IntersectionObserverEntry' in window && !'intersectionRatio' in window.IntersectionObserverEntry.prototype) {
              this.isFeatureSupported = false
              // feature is not supportet so set isInit to true 
              this.isInit = true;
          }
          else {
            this.isFeatureSupported = true
          }
      },
      
      // 交叉情况变化处理函数
      intersectionHandler (entries) {
        if (
          // 正在交叉
          entries[0].isIntersecting ||
          // 交叉率大于0
          entries[0].intersectionRatio
        ) {
          this.init()
          this.io.unobserve(this.$el)
        }
      },


      init () {
        if (isFeatureSupported){
            this.$emit('beforeInit')
            this.$emit('before-init')
            this.loading = true

            this.requestAnimationFrame(() => {
              this.isInit = true
              this.$emit('init')
            })
        }
        
      },

      requestAnimationFrame (callback) {
        // 防止等待太久没有执行回调
        // 设置最大等待时间
        setTimeout(() => {
          if (this.isInit) return
          callback()
        }, this.maxWaitingTime)

        // 兼容不支持requestAnimationFrame 的浏览器
        return (window.requestAnimationFrame || ((callback) => setTimeout(callback, 1000 / 60)))(callback)
      }
    }
  }
</script>

