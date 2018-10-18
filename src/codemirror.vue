<template>
  <div class="vue-codemirror" :class="{ merge }">
    <div ref="mergeview" v-if="merge"></div>
    <textarea ref="textarea" :placeholder="placeholder" v-else></textarea>
  </div>
</template>

<script>
 
  import _CodeMirror from 'codemirror'
  
  // 코드미러 인스턴스를 변수에 담거나 없으면 import 한 코드미러를 임포트한다.
  const CodeMirror = window.CodeMirror || _CodeMirror

  // pollfill
  if (typeof Object.assign != 'function') {
    Object.defineProperty(Object, 'assign', {
      value(target, varArgs) {
        if (target == null) {
          throw new TypeError('Cannot convert undefined or null to object')
        }
        const to = Object(target)
        for (let index = 1; index < arguments.length; index++) {
          const nextSource = arguments[index]
          if (nextSource != null) {
            for (const nextKey in nextSource) {
              if (Object.prototype.hasOwnProperty.call(nextSource, nextKey)) {
                to[nextKey] = nextSource[nextKey]
              }
            }
          }
        }
        return to
      },
      writable: true,
      configurable: true
    })
  }

  // export 사용할 수 있게 내보낸다.
  export default {
    name: 'codemirror',
    data() {
      return {
        content: '', // 내용
        codemirror: null,
        cminstance: null // 인스턴스
      }
    },
    props: {
      code: String,
      value: String,
      marker: Function,
      unseenLines: Array,
      placeholder: {
        type: String,
        default: ''
      },
      merge: {
        type: Boolean,
        default: false
      },
      options: {
        type: Object,
        default: () => ({})
      },
      events: {
        type: Array,
        default: () => ([])
      },
      globalOptions: {
        type: Object,
        default: () => ({})
      },
      globalEvents: {
        type: Array,
        default: () => ([])
      }
    },
    watch: {
      // props 을 통해 외부에서 주입된 옵션 속성의 변경이 일어나면
      options: {
        deep: true,
        handler(options) {
          for (const key in options) {
            // 코드 미러 인스턴스에 해당 옵션을 설정한다.
            this.cminstance.setOption(key, options[key])
          }
        }
      },
      merge() {
        this.$nextTick(this.switchMerge)
      },
      // code props 로 새 값이 전달되면
      // 코드미러 인스턴스와 현재 컴포넌트의 내용물 값을 바꾼다.
      code(newVal) {
        this.handerCodeChange(newVal)
      },
      value(newVal) {
        this.handerCodeChange(newVal)
      },
    },
    methods: {
    
    
      initialize() {
        const cmOptions = Object.assign({}, this.globalOptions, this.options)
        if (this.merge) {
          this.codemirror = CodeMirror.MergeView(this.$refs.mergeview, cmOptions)
          this.cminstance = this.codemirror.edit
        } else {
          // 현 컴포넌트 textarea 에다가 옵션을 넣어서 코드미러 인스턴스 만든다.
          this.codemirror = CodeMirror.fromTextArea(this.$refs.textarea, cmOptions)
          // 인스턴스 변수에 담기
          this.cminstance = this.codemirror
          // 그리고는 인스턴스에 코도 혹은 변수를 설정한다. (초기 1회만)
          this.cminstance.setValue(this.code || this.value || this.content)
        }
        // 인스턴스에 이벤트 수신을 설정하는데
        // change 이벤트는 cm 이라는 것을 수신해서
        // cm 의 값을 가져와 현재 컴포넌트의 content 에 넣는다.
        this.cminstance.on('change', cm => {
          this.content = cm.getValue()
          // 대체 여기서 this 가 무엇을 말하는 지 모르겠지만...
          // $emit 이 있다면
          if (this.$emit) {
             // input 이라는 이벤트를 보낸다.
            this.$emit('input', this.content)
          }
        })

        // 所有有效事件（驼峰命名）+ 去重
        const tmpEvents = {}
        const allEvents = [
          'scroll',
          'changes',
          'beforeChange',
          'cursorActivity',
          'keyHandled',
          'inputRead',
          'electricInput',
          'beforeSelectionChange',
          'viewportChange',
          'swapDoc',
          'gutterClick',
          'gutterContextMenu',
          'focus',
          'blur',
          'refresh',
          'optionChange',
          'scrollCursorIntoView',
          'update'
        ]
        .concat(this.events)
        .concat(this.globalEvents)
        .filter(e => (!tmpEvents[e] && (tmpEvents[e] = true)))
        .forEach(event => {
          // 이벤트를 등록한다.
          this.cminstance.on(event, (...args) => {
            // console.log('当有事件触发了', event, args)
            
            // 코드미러 자체 이벤트를 수신받으면
            // 해당 이벤트와 매개변수를 고대로 뷰 이벤트 발신을 한다.
            // 즉, 뷰 컴포넌트쪽에 위의 이벤트 이름으로 수신을 하게 해주면..
            // 근데 왜 이렇게 하지? ;; 이해가 안 됨
            this.$emit(event, ...args)
            const lowerCaseEvent = event.replace(/([A-Z])/g, '-$1').toLowerCase()
            if (lowerCaseEvent !== event) {
              this.$emit(lowerCaseEvent, ...args)
            }
          })
        })


        // 코드 미러에 ready 이벤트를 보낸다.
        this.$emit('ready', this.codemirror)
        this.unseenLineMarkers()

        // prevents funky dynamic rendering
        this.refresh()
      },
      refresh() {
        this.$nextTick(() => {
          this.cminstance.refresh()
        })
      },
      destroy() {
        // garbage cleanup
        const element = this.cminstance.doc.cm.getWrapperElement()
        element && element.remove && element.remove()
      },
      
      handerCodeChange(newVal) {
        // 현재 코드미러 인스턴스의 값(코드내용)을 가져온다.
        const cm_value = this.cminstance.getValue()
        // 새 값이랑 현재 코드미러 인스턴스의 값과 같지 않다면(새 값이라면)
        if (newVal !== cm_value) {
          // cm 인스턴스와 현재 화면에 보이는 content 를 새 값으로 갱신하고
          const scrollInfo = this.cminstance.getScrollInfo()
          this.cminstance.setValue(newVal)
          this.content = newVal
          // cm 인스턴스에 스크롤 정보를 넣어준다.
          this.cminstance.scrollTo(scrollInfo.left, scrollInfo.top)
        }
        this.unseenLineMarkers()
      },
      
      unseenLineMarkers() {
        if (this.unseenLines !== undefined && this.marker !== undefined) {
          this.unseenLines.forEach(line => {
            const info = this.cminstance.lineInfo(line)
            this.cminstance.setGutterMarker(line, 'breakpoints', info.gutterMarkers ? null : this.marker())
          })
        }
      },
      switchMerge() {
        // Save current values
        const history = this.cminstance.doc.history
        const cleanGeneration = this.cminstance.doc.cleanGeneration
        this.options.value = this.cminstance.getValue()

        this.destroy()
        this.initialize()

        // Restore values
        this.cminstance.doc.history = history
        this.cminstance.doc.cleanGeneration = cleanGeneration
      }
    },
    mounted() {
      this.initialize()
    },
    beforeDestroy() {
      this.destroy()
    }
  }
</script>
