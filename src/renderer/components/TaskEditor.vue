<template>
  <transition name="modal">
    <div class="modal-mask" v-if="showEditor">
      <div class="modal-wrapper">
        <div class="modal-container">
          <div class="two-columns">
            <div class="left-column">
              <div class="modal-header">
                <slot name="header">
                  <h2>Name:</h2>
                  <template v-if="isChangingName">
                    <input v-model="name" @keydown.enter="changeName()" @blur="changeName()" ref="r1" />
                  </template>
                  <template v-else="">
                    <span @click="focusName">{{name}}</span>
                  </template>
                </slot>
              </div>
              <div class="modal-footer">
                <slot name="footer">
                  <div>
                    <h2>Prevs:</h2>
                    <dep-selector :type="'prev'" :event-bus="prevDepEventBus" :dep-ids-init="prevDepIds" :all-task-ids="allTaskIds" :self-task-id="taskId"/>
                    <h2>Nexts:</h2>
                    <dep-selector :type="'next'" :event-bus="nextDepEventBus" :dep-ids-init="nextDepIds" :all-task-ids="allTaskIds" :self-task-id="taskId"/>
                    <br/>
                  </div>
                  <div>
                    <el-button type="default" size="mini" icon="el-icon-close" v-on:click="close"></el-button>
                    <template v-if="task.isArchived">
                      <el-button size="mini" v-on:click="isArchived = false">Unarchive</el-button>
                      <el-button type="danger" size="mini" icon="el-icon-delete" v-on:click="removeTask"></el-button>
                    </template>
                    <template v-else="">
                      <el-button size="mini" v-on:click="isArchived = true">Archive</el-button>
                    </template>
                  </div>
                </slot>
              </div>
            </div>
            <div class="right-column">
              <h2>Description:</h2>
              <markdown-editor :text-input="task.description" :event-bus="descriptionEventBus"/>
              {{description}}
            </div>
          </div>
        </div>
      </div>
    </div>
  </transition>
</template>

<script>
import EventBus from './../event-bus'
import DepSelector from './DepSelector'
import MarkdownEditor from './MarkdownEditor'
import Vue from 'vue'

export default{
  name: 'task-editor',
  components: { DepSelector, MarkdownEditor },
  data: function () {
    return {
      showEditor: false,
      isChangingName: false,
      targetTasks: [],
      prevDepEventBus: new Vue(),
      nextDepEventBus: new Vue(),
      descriptionEventBus: new Vue(),
      lists: [],
      taskId: null,
      prevDepIds: [],
      nextDepIds: [],
      description: '',
      name: ''
    }
  },
  computed: {
    task: function () {
      return this.$store.getters.task(this.taskId)
    },
    allTaskIds: function () {
      let ids = this.$store.getters.allTaskIds
      return ids
    },
    isArchived: {
      get: function () {
        return this.task.isArchived
      },
      set: function (f) {
        if (f && !this.$store.getters.isArchivable(this.taskId)) {
          if (!confirm('Archive task?')) {
            return
          }
        }
        this.$store.commit('changeTask', {taskId: this.taskId, isArchived: f})
      }
    }
  },
  methods: {
    cancelChangingName: function () {
      this.isChangingName = false
    },
    changeName: function () {
      if (!this.showEditor) return
      if (this.name.length === 0) {
        this.name = 'Enter task name'
      }
      this.isChangingName = false
    },
    open: function (payload) {
      console.log('opended')
      this.taskId = payload.taskId
      this.name = this.task.name
      if (payload.isFocus) {
        this.focusName()
      }

      this.prevDepIds.splice(0)
      this.nextDepIds.splice(0)
      for (let id of this.task.prevIds) {
        this.prevDepIds.push(id)
      }
      for (let id of this.task.nextIds) {
        this.nextDepIds.push(id)
      }

      this.showEditor = true
    },
    focusName: function () {
      this.isChangingName = true
      this.$nextTick(function () { this.$refs.r1.focus() })
    },
    close: function () {
      this.changeName()

      this.showEditor = false

      this.$store.commit('changeTask', {taskId: this.taskId, name: this.name})

      this.isChangingName = false
    },
    removeTask: function () {
      if (confirm('Remove task?')) {
        this.$store.dispatch('removeTask', {taskId: this.taskId})
        this.isChangingName = false
        this.targetTasks = []
        this.showEditor = false
      }
    },
    escapeKeyListener: function () {
      if (event.keyCode === 27 && this.showEditor) {
        this.close()
      }
    }
  },
  mounted: function () {
    EventBus.$on('open-task-editor', this.open)
    EventBus.$on('close-task-editor', this.close)
    document.addEventListener('keyup', this.escapeKeyListener)

    this.prevDepEventBus.$on('addDepsToTask', depId => {
      this.$store.commit('addDep', {prev: depId, next: this.taskId})
    })
    this.nextDepEventBus.$on('addDepsToTask', depId => {
      this.$store.commit('addDep', {prev: this.taskId, next: depId})
    })

    this.prevDepEventBus.$on('removeDepsToTask', depId => {
      this.$store.commit('removeDep', {prev: depId, next: this.taskId})
    })
    this.nextDepEventBus.$on('removeDepsToTask', depId => {
      this.$store.commit('removeDep', {prev: this.taskId, next: depId})
    })
    this.descriptionEventBus.$on('change', text => {
      this.$store.commit('changeDescription', {taskId: this.taskId, description: text})
    })
  }
}
</script>

<style>
.modal-mask {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, .5);
  display: table;
  transition: opacity .3s ease;
}

.modal-wrapper {
  display: table-cell;
  vertical-align: middle;
}

.modal-container {
  width: 800px;
  margin: 0px auto;
  padding: 30px 30px;
  background-color: #fff;
  border-radius: 2px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, .33);
  transition: all .3s ease;
}

.modal-header {
  margin-top: 0;
}

button, input, select, textarea {
  font-family : inherit;
  font-size   : 100%;
} 

.modal-body {
  margin: 20px 0;
}

.modal-default-button {
  float: right;
}

/*
 * The following styles are auto-applied to elements with
 * transition="modal" when their visibility is toggled
 * by Vue.js.
 *
 * You can easily play with the modal transition by editing
 * these styles.
 */

.modal-enter {
  opacity: 0;
}

.modal-leave-active {
  opacity: 0;
}

.modal-enter .modal-container,
.modal-leave-active .modal-container {
  -webkit-transform: scale(1.1);
  transform: scale(1.1);
}
.two-columns{
  display: flex;
  flex-direction: row;
  justify-content: space-between;
}
.left-column{
    width: 50%;
    padding: 20px;
}

.right-column{
    width: 50%;
    padding: 20px;
    display: flex;
    flex-direction: column;
    justify-content: flex-start;
}
</style>
