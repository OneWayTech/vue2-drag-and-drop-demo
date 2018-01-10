<template>
  <div
    :class="['tree-node', { 'empty-node': amIEmptyNode }]"
    :draggable="amIEmptyNode ? undefined : true"
    @dragover.prevent="/* @dragover.prevent is a must (browsers disable dragover by default) */"
    @dragstart.stop="handleDragStart"
    @drop.stop="handleDrop"
    @dragenter.stop="handleDragEnter"
    @dragleave.stop="handleDragLeave"
    @dragend.prevent="handleDragEnd">
    {{ data.label /* undefined if empty node */ }}
    <div v-if="displayedChildren.length" class="tree-node-children">
      <tree-node
        v-for="(child, idx) in displayedChildren"
        :data="child"
        :shared="shared"
        :vm-idx="idx">
      </tree-node>
    </div>
  </div>
</template>
<script>
export default {
  name: 'TreeNode', // as a recursive component
  props: {
    data: { type: Object, required: true }, // { label: String, children: [{ label, children }] } or an empty object
    shared: { type: Object, default: () => ({/* draggingVm: VueInstance */}) }, // shared data for all the instances
    vmIdx: Number // current instance's index in v-for (if exists)
  },
  computed: {
    amIEmptyNode () {
      // data of an empty node is an empty object: {}
      return !this.data.label
    },
    /**
     * Generate adjacent empty nodes for each real node, in order to implement insertion actions
     * e.g. [R1, R2, R3] === displayed as ===> [E1, R1, E2, R2, E3, R3, E4]
     * (R means Real node, E means Empty node)
     */
    displayedChildren () {
      const realNodes = this.data.children
      if (!realNodes || !realNodes.length) return [] // an empty node or a real node without children

      return realNodes.reduce((displayedChildren, realNode) => {
        displayedChildren.push(realNode, {}/* <--- empty node */)
        return displayedChildren
      }, [{}])
    }
  },
  methods: {
    /**
     * @context {this} - instance of dragging node
     */
    handleDragStart () {
      // this.shared.draggingVm = this // cannot ensure reactive
      this.$set(this.shared, 'draggingVm', this) // ensure reactive
      this.$el.classList.add('dragging-node')
    },
    /**
     * @context {this} - instance of drop-into node
     */
    isAllowedToDrop () {
      let vm = this
      const { draggingVm } = vm.shared

      // limitation 1: this cannot be the parent of the dragging node
      if (vm === draggingVm.$parent) {
        return false
      }

      // limitation 2: this cannot be the adjacent empty node of the dragging node
      if (vm.$parent === draggingVm.$parent && Math.abs(vm.vmIdx - draggingVm.vmIdx) === 1) {
        return false
      }

      // limitation 3: this cannot be the dragging node itself or its descendant
      while (vm) {
        if (vm === draggingVm) return false
        vm = vm.$parent.$options.name === 'TreeNode' ? vm.$parent : null
      }

      return true
    },
    /**
     * @context {this} - instance of drop-into node
     */
    handleDrop () {
      this.restoreStyle()
      if (!this.isAllowedToDrop()) return

      const { draggingVm } = this.shared

      // remove from the original place
      const realIdxOfOrigin = (draggingVm.vmIdx - 1) / 2
      draggingVm.$parent.data.children.splice(realIdxOfOrigin, 1)

      // case 1: drop into an empty node
      if (this.amIEmptyNode) {
        this.$parent.data.children.splice(this.vmIdx / 2, 0, draggingVm.data)
        return
      }

      // case 2: drop into a real node as its child
      if (!this.data.children) {
        this.$set(this.data, 'children', []) // ensure reactive
      }
      this.data.children.push(draggingVm.data)
    },
    handleDragEnter () {
      this.$el.classList.add(this.isAllowedToDrop() ? 'drop-allowed' : 'drop-not-allowed')
    },
    handleDragLeave () {
      if (this.shared.draggingVm !== this) { // avoid removing .dragging-node
        this.restoreStyle()
      }
    },
    handleDragEnd () {
      this.shared.draggingVm = null
      this.restoreStyle()
    },
    restoreStyle () {
      this.$el.classList.remove('dragging-node', 'drop-allowed', 'drop-not-allowed')
    }
  }
}
</script>
<style>
.tree-node {
  display: list-item;
  padding-left: 2px;
  list-style: none;
  line-height: 20px;
  border-left: 1px dashed #ddd;
  transition: height .5s ease;
}
.empty-node {
  height: 10px;
}
.tree-node-children {
  margin-left: 30px; /* indention */
}
.dragging-node {
  color: #fff;
}
.drop-allowed {
  height: 30px;
  border: 1px dashed #ddd;
  border-radius: 5px;
  background-color: yellow;
}
.drop-not-allowed {
  cursor: not-allowed;
  background-color: #ddd;
}
</style>
