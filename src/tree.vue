<template>
  <div :class="classes" role="tree" onselectstart="return false">
    <ul :class="containerClasses" role="group">
      <tree-item v-for="(child, index) in data"
          :key="index"
          :item="child"
          :text-field-name="textFieldName"
          :whole-row="wholeRow"
          :show-checkbox="showCheckbox"
          :height="sizeHight"
          :parent-item="data"
          :draggable="draggable"
          :on-item-click="onItemClick"
          :on-item-toggle="onItemToggle"
          :on-item-drag-start="onItemDragStart"
          :on-item-drag-end="onItemDragEnd"
          :on-item-drop="onItemDrop"
          :klass="index === data.length-1?'tree-last':''">
        <template slot-scope="_">
          <slot :vm="_.vm" :item="_.item">{{ _.item.title }}</slot>
        </template>
      </tree-item>
    </ul>
  </div>
</template>
<script>
  import TreeItem from './tree-item.vue'

  let ITEM_HEIGHT_SMALL = 18
  let ITEM_HEIGHT_DEFAULT = 24
  let ITEM_HEIGHT_LARGE = 32

  export default {
    name: 'VJstree',
    props: {
      data: {type: Array},
      size: {type: String, validator: value => ['large', 'small'].indexOf(value) > -1},
      showCheckbox: {type: Boolean, default: false},
      wholeRow: {type: Boolean, default: false},
      noDots: {type: Boolean, default: false},
      multiple: {type: Boolean, default: false},
      allowBatch: {type: Boolean, default: false},
      textFieldName: {type: String, default: 'text'},
      async: {type: Function},
      loadingText: {type: String, default: 'Loading...'},
      draggable: {type: Boolean, default: false},
      klass: String
    },
    data () {
      return {
        draggedItem: null
      }
    },
    computed: {
      classes () {
        return [
          {'tree': true},
          {'tree-default': !this.size},
          {[`tree-default-${this.size}`]: !!this.size},
          {'tree-checkbox-selection': !!this.showCheckbox},
          {[this.klass]: !!this.klass}
        ]
      },
      containerClasses () {
        return [
          {'tree-container-ul': true},
          {'tree-children': true},
          {'tree-wholerow-ul': !!this.wholeRow},
          {'tree-no-dots': !!this.noDots}
        ]
      },
      sizeHight () {
        switch (this.size) {
          case 'large':
            return ITEM_HEIGHT_LARGE
          case 'small':
            return ITEM_HEIGHT_SMALL
          default:
            return ITEM_HEIGHT_DEFAULT
        }
      }
    },
    methods: {
      initializeData (items) {
        if (items && items.length > 0) {
          for (let i in items) {
            var dataItem = this.initializeDataItem(items[i])
            items[i] = dataItem
            this.initializeData(items[i].children)
          }
        }
      },
      initializeDataItem (item) {
        let model = {
          item: item.item,
          icon: item.icon || '',
          opened: item.opened || false,
          selected: item.selected || false,
          disabled: item.disabled || false,
          loading: item.loading || false,
          children: item.children || []
        }

        model[this.textFieldName] = item[this.textFieldName] || ''
        return model
      },
      initializeLoading () {
        var item = {}
        item[this.textFieldName] = this.loadingText
        item.disabled = true
        item.loading = true
        return this.initializeDataItem(item)
      },
      handleRecursionNodeChilds (node, func) {
        if (node.$children && node.$children.length > 0) {
          for (let childNode of node.$children) {
            if (!childNode.disabled) {
              func(childNode)
              this.handleRecursionNodeChilds(childNode, func)
            }
          }
        }
      },
      onItemClick (oriNode, oriItem) {
        if (this.multiple) {
          if (this.allowBatch) {
            this.handleBatchSelectItems(oriNode, oriItem)
          }
        } else {
          this.handleSingleSelectItems(oriNode, oriItem)
        }
        this.$emit('item-click', oriNode, oriItem)
      },
      handleSingleSelectItems (oriNode, oriItem) {
        this.handleRecursionNodeChilds(this, node => {
          node.model.selected = false
        })
        oriNode.model.selected = true
      },
      handleBatchSelectItems (oriNode, oriItem) {
        this.handleRecursionNodeChilds(oriNode, node => {
          if (node.model.disabled) return
          node.model.selected = oriNode.model.selected
        })
      },
      onItemToggle (oriNode, oriItem) {
        if (oriNode.model.opened) {
          this.handleAsyncLoad(oriNode.model.children, oriNode, oriItem)
        }
      },
      handleAsyncLoad (oriParent, oriNode, oriItem) {
        var self = this
        if (this.async) {
          if (oriParent[0].loading) {
            this.async(oriNode, (data) => {
              if (data.length > 0) {
                for (let i in data) {
                  data[i].children = [self.initializeLoading()]
                  var dataItem = self.initializeDataItem(data[i])
                  self.$set(oriParent, i, dataItem)
                }
              } else {
                oriNode.model.children = []
              }
            })
          }
        }
      },
      onItemDragStart (e, oriNode, oriItem) {
        if (!this.draggable) { return false }

        e.dataTransfer.effectAllowed = 'move'
        e.dataTransfer.setData('text', null)
        this.draggedItem = {
          item: oriItem,
          parentItem: oriNode.parentItem,
          index: oriNode.parentItem.indexOf(oriItem)
        }
      },
      onItemDragEnd (e, oriNode, oriItem) {
        if (!this.draggable) { return false }
        this.draggedItem = null
      },
      onItemDrop (e, oriNode, oriItem) {
        if (!this.draggable) {
          return false
        }

        if (this.draggedItem) {
          if (this.draggedItem.parentItem === oriItem.children ||
            this.draggedItem.item === oriItem ||
            (oriItem.children && oriItem.children.indexOf(this.draggedItem.item) !== -1)) {
            return
          }
          oriItem.children = oriItem.children ? oriItem.children.concat(this.draggedItem.item) : [this.draggedItem.item]
          oriItem.opened = true
          this.$emit('item-drop', this.draggedItem.item, oriItem)

          var self = Object.assign({}, this)
          this.$nextTick(() => {
            self.draggedItem.parentItem.splice(self.draggedItem.index, 1)
          })
        }
      }
    },
    mounted () {
      this.initializeData(this.data)

      if (this.async) {
        this.$set(this.data, 0, this.initializeLoading())
        this.handleAsyncLoad(this.data, this)
      }
    },
    components: {
      TreeItem
    },
    watch: {
      data () {
        this.initializeData(this.data)
      }
    }
  }
</script>
<style lang="less">
    @import "./less/style";
</style>