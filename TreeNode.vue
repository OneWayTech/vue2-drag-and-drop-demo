<template>
  <!-- @dragover.prevent is a must (browsers disable dragover by default) -->
  <div
    class="tree-node"
    :class="{ 'empty-node': amIEmptyNode }"
    :draggable="amIEmptyNode ? undefined : true"
    @dragover.prevent
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
      // an empty node is an empty object: {}
      return !this.data.label
    },
    /**
     * Generate adjacent empty nodes for each real node, in order to implement insertion actions
     * e.g. [RN1, RN2, RN3] === displayed as ===> [EN1, RN1, EN2, RN2, EN3, RN3, EN4]
     * (RN means Real Node, EN means Empty Node)
     */
    displayedChildren () {
      const realNodes = this.data.children
      if (!realNodes || !realNodes.length) return [] // empty node or a real node without children

      return realNodes.reduce((displayedChildren, realNode) => {
        displayedChildren.push(realNode, {}/* <--- empty node */)
        return displayedChildren
      }, [{}])
    },
    /**
     * The drop-into node's limitation 1: am I the parent of the dragging node
     * @return {Boolean}
     */
    isDraggingVmMySon () {
      return this === this.shared.draggingVm.$parent
    },
    /**
     * The drop-into node's limitation 2: am I the adjacent empty node of the dragging node
     * @return {Boolean}
     */
    isDraggingVmMyAdjacence () {
      const { draggingVm } = this.shared
      return this.$parent === draggingVm.$parent && Math.abs(this.vmIdx - draggingVm.vmIdx) === 1
    },
    /**
     * The drop-into node's limitation 3: am I exactly the dragging node or it happended to be my ancestor
     * @return {Boolean}
     */
    isDraggingVmMyselfOrMyAncestor () {
      var p = this
      while (p) {
        if (p === this.shared.draggingVm) return true
        p = p.$parent.$options.name === 'TreeNode' ? p.$parent : null
      }
    },
    /**
     * Combination of the limitations above
     * @return {Boolean}
     */
    isAllowedToDrop () {
      return !(this.isDraggingVmMySon || this.isDraggingVmMyAdjacence || this.isDraggingVmMyselfOrMyAncestor)
    }
  },
  methods: {
    /**
     * The context `this` is the dragging instance
     */
    handleDragStart () {
      // this.shared.draggingVm = this // does not ensure reactive
      this.$set(this.shared, 'draggingVm', this) // ensure reactive
      this.$el.classList.add('dragging-node')
    },
    /**
     * The context `this` is the node (instance) to drop into, not the dragging instance
     */
    handleDrop () {
      this.restoreStyle()
      if (!this.isAllowedToDrop) return

      const { draggingVm } = this.shared

      // remove from the original place
      const realIdxOfOrigin = (draggingVm.vmIdx - 1) / 2
      draggingVm.$parent.data.children.splice(realIdxOfOrigin, 1)

      // case 1: drop into an empty node
      if (this.amIEmptyNode) {
        this.$parent.data.children.splice(this.vmIdx / 2, 0, draggingVm.data)
        return
      }

      // case 2: drop into a real node to be its child
      if (!this.data.children) {
        this.$set(this.data, 'children', []) // ensure reactive
      }
      this.data.children.push(draggingVm.data)
    },
    handleDragEnter () {
      this.$el.classList.add(this.isAllowedToDrop ? 'drop-allowed' : 'drop-not-allowed')
    },
    handleDragLeave () {
      if (this.shared.draggingVm !== this) {
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
