<template>
<div
  class="tree"
  :class="{
    'tree-highlight-current': highlightCurrent,
    'is-dragging': !!dragState.draggingNode,
    'is-drop-not-allow': !dragState.allowDrop,
    'is-drop-inner': dragState.dropType === 'inner'
  }"
  role="tree"
>
  <tree-node
    v-for="child in root.childNodes"
    :node="child"
    :props="props"
    :render-after-expand="renderAfterExpand"
    :key="getNodeKey(child)"
    :render-content="renderContent"
    @node-expand="handleNodeExpand">
  </tree-node>

  <div
    v-show="dragState.showDropIndicator"
    class="tree-drop-indicator"
    ref="dropIndicator">
  </div>
</div>
</template>

<script>
import TreeStore from './model/tree-store';
import { getNodeKey, findNearestComponent } from './model/util';
import TreeNode from './tree-node.vue';
import tools from '../../../mixins/tools.js';

export default {
  name: 'Tree',

  mixins: [tools],

  components: {
    TreeNode
  },

  data() {
    return {
      store: null,
      root: null,
      currentNode: null,
      treeItems: null,
      checkboxItems: [],
      dragState: {
        showDropIndicator: false,
        draggingNode: null,
        dropNode: null,
        allowDrop: true
      }
    };
  },

  props: {
    data: {
      type: Array
    },
    renderAfterExpand: {
      type: Boolean,
      default: true
    },
    nodeKey: String,
    checkStrictly: Boolean,
    defaultExpandAll: Boolean,
    expandOnClickNode: {
      type: Boolean,
      default: true
    },
    checkDescendants: {
      type: Boolean,
      default: false
    },
    autoExpandParent: {
      type: Boolean,
      default: true
    },
    defaultCheckedKeys: Array,
    defaultExpandedKeys: Array,
    renderContent: Function,
    showCheckbox: {
      type: Boolean,
      default: false
    },
    draggable: {
      type: Boolean,
      default: false
    },
    allowDrag: Function,
    allowDrop: Function,
    props: {
      default() {
        return {
          children: 'children',
          label: 'label',
          icon: 'icon',
          disabled: 'disabled'
        };
      }
    },
    lazy: {
      type: Boolean,
      default: false
    },
    highlightCurrent: Boolean,
    load: Function,
    filterNodeMethod: Function,
    accordion: Boolean,
    indent: {
      type: Number,
      default: 18
    }
  },

  computed: {
    children: {
      set(value) {
        this.data = value;
      },
      get() {
        return this.data;
      }
    },

    treeItemArray() {
      return Array.prototype.slice.call(this.treeItems);
    }
  },

  watch: {
    defaultCheckedKeys(newVal) {
      this.store.defaultCheckedKeys = newVal;
      this.store.setDefaultCheckedKey(newVal);
    },

    defaultExpandedKeys(newVal) {
      this.store.defaultExpandedKeys = newVal;
      this.store.setDefaultExpandedKeys(newVal);
    },

    data(newVal) {
      this.store.setData(newVal);
    },

    checkboxItems(val) {
      Array.prototype.forEach.call(val, (checkbox) => {
        checkbox.setAttribute('tabindex', -1);
      });
    }
  },

  methods: {
    filter(value) {
      if (!this.filterNodeMethod)
        throw new Error('[Tree] filterNodeMethod is required when filter');
      this.store.filter(value);
    },

    getNodeKey(node) {
      return getNodeKey(this.nodeKey, node.data);
    },

    getNodePath(data) {
      if (!this.nodeKey)
        throw new Error('[Tree] nodeKey is required in getNodePath');
      const node = this.store.getNode(data);
      if (!node) return [];
      const path = [node.data];
      let parent = node.parent;
      while (parent && parent !== this.root) {
        path.push(parent.data);
        parent = parent.parent;
      }
      return path.reverse();
    },

    getCheckedNodes(leafOnly) {
      return this.store.getCheckedNodes(leafOnly);
    },

    getCheckedKeys(leafOnly) {
      return this.store.getCheckedKeys(leafOnly);
    },

    getCurrentNode() {
      const currentNode = this.store.getCurrentNode();
      return currentNode ? currentNode.data : null;
    },

    getCurrentKey() {
      if (!this.nodeKey)
        throw new Error('[Tree] nodeKey is required in getCurrentKey');
      const currentNode = this.getCurrentNode();
      return currentNode ? currentNode[this.nodeKey] : null;
    },

    setCheckedNodes(nodes, leafOnly) {
      if (!this.nodeKey)
        throw new Error('[Tree] nodeKey is required in setCheckedNodes');
      this.store.setCheckedNodes(nodes, leafOnly);
    },

    setCheckedKeys(keys, leafOnly) {
      if (!this.nodeKey)
        throw new Error('[Tree] nodeKey is required in setCheckedKeys');
      this.store.setCheckedKeys(keys, leafOnly);
    },

    setChecked(data, checked, deep) {
      this.store.setChecked(data, checked, deep);
    },

    getHalfCheckedNodes() {
      return this.store.getHalfCheckedNodes();
    },

    getHalfCheckedKeys() {
      return this.store.getHalfCheckedKeys();
    },

    setCurrentNode(node) {
      if (!this.nodeKey)
        throw new Error('[Tree] nodeKey is required in setCurrentNode');
      this.store.setUserCurrentNode(node);
    },

    setCurrentKey(key) {
      if (!this.nodeKey)
        throw new Error('[Tree] nodeKey is required in setCurrentKey');
      this.store.setCurrentNodeKey(key);
    },

    getNode(data) {
      return this.store.getNode(data);
    },

    remove(data) {
      this.store.remove(data);
    },

    append(data, parentNode) {
      this.store.append(data, parentNode);
    },

    insertBefore(data, refNode) {
      this.store.insertBefore(data, refNode);
    },

    insertAfter(data, refNode) {
      this.store.insertAfter(data, refNode);
    },

    handleNodeExpand(nodeData, node, instance) {
      this.$emit('node-expand', nodeData, node, instance);
    },

    updateKeyChildren(key, data) {
      if (!this.nodeKey)
        throw new Error('[Tree] nodeKey is required in updateKeyChild');
      this.store.updateChildren(key, data);
    },

    initTabIndex() {
      this.treeItems = this.$el.querySelectorAll(
        '.is-focusable[role=treeitem]'
      );
      this.checkboxItems = this.$el.querySelectorAll('input[type=checkbox]');
      const checkedItem = this.$el.querySelectorAll(
        '.is-checked[role=treeitem]'
      );
      if (checkedItem.length) {
        checkedItem[0].setAttribute('tabindex', 0);
        return;
      }
      this.treeItems[0] && this.treeItems[0].setAttribute('tabindex', 0);
    },

    handelKeydown(ev) {
      const currentItem = ev.target;
      if (currentItem.className.indexOf('tree-node') === -1) return;
      ev.preventDefault();
      const keyCode = ev.keyCode;
      this.treeItems = this.$el.querySelectorAll(
        '.is-focusable[role=treeitem]'
      );
      const currentIndex = this.treeItemArray.indexOf(currentItem);
      let nextIndex;
      if ([38, 40].indexOf(keyCode) > -1) {
        // up、down
        if (keyCode === 38) {
          // up
          nextIndex = currentIndex !== 0 ? currentIndex - 1 : 0;
        } else {
          nextIndex =
            currentIndex < this.treeItemArray.length - 1 ? currentIndex + 1 : 0;
        }
        this.treeItemArray[nextIndex].focus(); // 选中
      }
      if ([37, 39].indexOf(keyCode) > -1) {
        // left、right 展开
        currentItem.click(); // 选中
      }
      const hasInput = currentItem.querySelector('[type="checkbox"]');
      if ([13, 32].indexOf(keyCode) > -1 && hasInput) {
        // space enter选中checkbox
        hasInput.click();
      }
    }
  },

  created() {
    this.isTree = true;

    this.store = new TreeStore({
      key: this.nodeKey,
      data: this.data,
      lazy: this.lazy,
      props: this.props,
      load: this.load,
      currentNodeKey: this.currentNodeKey,
      checkStrictly: this.checkStrictly,
      checkDescendants: this.checkDescendants,
      defaultCheckedKeys: this.defaultCheckedKeys,
      defaultExpandedKeys: this.defaultExpandedKeys,
      autoExpandParent: this.autoExpandParent,
      defaultExpandAll: this.defaultExpandAll,
      filterNodeMethod: this.filterNodeMethod
    });

    this.root = this.store.root;

    let dragState = this.dragState;
    this.$on('tree-node-drag-start', (event, treeNode) => {
      if (
        typeof this.allowDrag === 'function' &&
        !this.allowDrag(treeNode.node)
      ) {
        event.preventDefault();
        return false;
      }
      event.dataTransfer.effectAllowed = 'move';

      try {
        event.dataTransfer.setData('text/plain', '');
      } catch (e) {
        console.log(e);
      }

      dragState.draggingNode = treeNode;
      this.$emit('node-drag-start', treeNode.node, event);
    });

    this.$on('tree-node-drag-over', (event, treeNode) => {
      const dropNode = findNearestComponent(event.target, 'TreeNode');
      const oldDropNode = dragState.dropNode;
      if (oldDropNode && oldDropNode !== dropNode) {
        this.removeClass(oldDropNode.$el, 'is-drop-inner');
      }
      const draggingNode = dragState.draggingNode;
      if (!draggingNode || !dropNode) return;

      let allowDrop = true;
      if (
        typeof this.allowDrop === 'function' &&
        !this.allowDrop(draggingNode.node, dropNode.node)
      ) {
        allowDrop = false;
      }
      dragState.allowDrop = allowDrop;
      event.dataTransfer.dropEffect = allowDrop ? 'move' : 'none';
      if (allowDrop && oldDropNode !== dropNode) {
        if (oldDropNode) {
          this.$emit(
            'node-drag-leave',
            draggingNode.node,
            oldDropNode.node,
            event
          );
        }
        this.$emit('node-drag-enter', draggingNode.node, dropNode.node, event);
      }

      if (allowDrop) {
        dragState.dropNode = dropNode;
      }

      let dropPrev = allowDrop;
      let dropInner = allowDrop;
      let dropNext = allowDrop;

      if (dropNode.node.nextSibling === draggingNode.node) {
        dropNext = false;
      }
      if (dropNode.node.previousSibling === draggingNode.node) {
        dropPrev = false;
      }
      if (dropNode.node.contains(draggingNode.node, false)) {
        dropInner = false;
      }
      if (
        draggingNode.node === dropNode.node ||
        draggingNode.node.contains(dropNode.node)
      ) {
        dropPrev = false;
        dropInner = false;
        dropNext = false;
      }

      const targetPosition = dropNode.$el
        .querySelector('.tree-node-content')
        .getBoundingClientRect();

      const treePosition = this.$el.getBoundingClientRect();

      let dropType;
      const prevPercent = dropPrev
        ? dropInner
          ? 0.25
          : dropNext
            ? 0.5
            : 1
        : -1;
      const nextPercent = dropNext
        ? dropInner
          ? 0.75
          : dropPrev
            ? 0.5
            : 0
        : 1;

      let indicatorTop = -9999;
      const distance = event.clientY - targetPosition.top;

      if (distance < targetPosition.height * prevPercent) {
        dropType = 'before';
      } else if (distance > targetPosition.height * nextPercent) {
        dropType = 'after';
      } else if (dropInner) {
        dropType = 'inner';
      } else {
        dropType = 'none';
      }

      const dropIndicator = this.$refs.dropIndicator;
      if (dropType === 'before') {
        indicatorTop = targetPosition.top - treePosition.top;
      } else if (dropType === 'after') {
        indicatorTop = targetPosition.bottom - treePosition.top;
      }
      dropIndicator.style.top = indicatorTop + 'px';
      dropIndicator.style.left =
        targetPosition.right - treePosition.left + 'px';

      if (dropType === 'inner') {
        this.addClass(dropNode.$el, 'is-drop-inner');
      } else {
        this.removeClass(dropNode.$el, 'is-drop-inner');
      }

      dragState.showDropIndicator =
        dropType === 'before' || dropType === 'after';
      dragState.dropType = dropType;
      this.$emit('node-drag-over', draggingNode.node, dropNode.node, event);
    });

    this.$on('tree-node-drag-end', (event) => {
      const { draggingNode, dropType, dropNode } = dragState;
      event.preventDefault();
      event.dataTransfer.dropEffect = 'move';

      if (draggingNode && dropNode) {
        const data = draggingNode.node.data;
        if (dropType === 'before') {
          draggingNode.node.remove();
          dropNode.node.parent.insertBefore({ data }, dropNode.node);
        } else if (dropType === 'after') {
          draggingNode.node.remove();
          dropNode.node.parent.insertAfter({ data }, dropNode.node);
        } else if (dropType === 'inner') {
          dropNode.node.insertChild({ data });
          draggingNode.node.remove();
        }
        this.removeClass(dropNode.$el, 'is-drop-inner');

        this.$emit(
          'node-drag-end',
          draggingNode.node,
          dropNode.node,
          dropType,
          event
        );
        if (dropType !== 'none') {
          this.$emit(
            'node-drop',
            draggingNode.node,
            dropNode.node,
            dropType,
            event
          );
        }
      }
      if (draggingNode && !dropNode) {
        this.$emit('node-drag-end', draggingNode.node, null, dropType, event);
      }

      dragState.showDropIndicator = false;
      dragState.draggingNode = null;
      dragState.dropNode = null;
      dragState.allowDrop = true;
    });
  },

  mounted() {
    this.initTabIndex();
    this.$el.addEventListener('keydown', this.handelKeydown);
  },

  updated() {
    this.treeItems = this.$el.querySelectorAll('[role=treeitem]');
    this.checkboxItems = this.$el.querySelectorAll('input[type=checkbox]');
  }
};
</script>

<style lang="stylus">
.tree {
  position: relative;
  color: #606266;
}

.tree-drop-indicator {
  position: absolute;
  right: 0;
  left: 0 !important;
  background-color: #409EFF;
  height: 3px;
}

.tree-node {
  white-space: nowrap;
  outline: 0;
}

.tree-node-label:hover {
  background-color: transparent;
}

.tree-node:focus > .tree-node-content {
  background-color: #f5f5f5;
}

.tree-node.is-drop-inner > .tree-node-content {
  background-color: #409EFF;
  padding: 8px 0;
  color: #fff;
}

.tree-node-content {
  display: flex;
  align-items: center;
  height: auto;
  cursor: pointer;
  padding: 5px;
}

.tree-node-content > .tree-node-expand-icon {
  padding: 6px;
}

.tree.is-dragging .tree-node-content {
  cursor: move;
}

.tree.is-dragging .tree-node-content * {
  pointer-events: none;
}

.tree.is-dragging.is-drop-not-allow .tree-node-content {
  cursor: not-allowed;
}

.tree-node-expand-icon {
  cursor: pointer;
  color: #c0c4cc;
  font-size: 12px;
  transform: rotate(0);
  transition: transform .3s ease-in-out;
}

.tree-node-expand-icon.expanded {
  transform: rotate(90deg);
}

.tree-node-expand-icon.is-leaf {
  color: transparent;
  cursor: default;
}

.tree-node-label {
  font-size: 14px;
  margin-left: 5px;
}

.tree-node > .tree-node-children {
  overflow: hidden;
  background-color: transparent;
}

.tree-node.is-expanded > .tree-node-children {
  display: block;
}

.tree-highlight-current .tree-node.is-current > .tree-node-content {
  background-color: #f0f7ff;
}
</style>
