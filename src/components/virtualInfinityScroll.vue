<template>
  <div ref="container" class="virtual-scroll-container" @scroll="handleScroll">
    <div :style="spacerStyle"></div>
    <div :style="scrollBoxStyle" :class="props.containerClass" ref="scrollBox">
      <!-- <TransitionGroup :name="props.transitionName"> -->
        <template v-if="props.items.length">
          <div
            class="virtual-scroll-item"
            v-for="(item, index) in visibleItems"
            :key="item[props.uniqueKey]"
            :data-key="item[props.uniqueKey]"
          >
            <slot :item="item" name="item">
              {{ item[props.uniqueKey] }}
            </slot>
          </div>
        </template>
      <!-- </TransitionGroup> -->
      <template v-if="!props.pendingState && !items.length">
        <slot name="empty"> No items available to show! </slot>
      </template>
      <template v-if="props.pendingState">
        <slot name="loader"> Loading... </slot>
      </template>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, nextTick, onMounted, watch } from "vue";

const props = defineProps({
  items: {
    type: Array,
    required: true,
    default: []
  },
  totalItems: {
    type: Number,
    default: 1000,
  },
  // Must provide to render it properly
  uniqueKey: { 
    type: String,
    default: "id",
  },
  bufferSize: {
    type: Number,
    default: 5,
  },
  containerClass: String,
  pendingState: Boolean,
  // transitionName: {
  //   type:  String,
  //   default: "list"
  // }
});

const emit = defineEmits(["getNewItems"]);

//Other needed refs
const container = ref(null);
const scrollBox = ref(null);

// Start and End index to determine which items should be visible
const visibleStartIndex = ref(0);
const visibleEndIndex = ref(1000);

// Array to store item heights
const itemHeights = ref([]);

// Visible items in the viewport
const visibleItems = computed(() =>
  props.items.slice(visibleStartIndex.value, visibleEndIndex.value)
);

// Style for the spacer, it helps keep the scroll bar positioned properly
const spacerStyle = computed(() => {
  return {
    height: `${itemHeights.value.reduce((acc, height) => acc + height.value, 0)}px`,
  };
});

// Update the top of the contentContainer so that it stays
// in the right position when scrolled
const scrollBoxStyle = computed(() => ({
  position: "absolute",
  top: `${itemHeights.value
    .slice(0, Math.max(0, visibleStartIndex.value))
    .reduce((acc, height) => acc + height.value, 0)}px`,
  left: "0",
  right: "0",
}));

// Insert item heights for new items
const updateItemHeights = (item) => {
  const key = item.dataset.key;
  const iH = itemHeights.value.find(height=> height[props.uniqueKey] == key);
  if (iH) {
    // If already height exist update
    iH.value = item.offsetHeight;
  } else {
    // If not add new Height
    itemHeights.value.push({
      [props.uniqueKey]: key,
      value: item.offsetHeight
    });

  }
};

// When new items added
const onItemsAdd = () => {
  visibleEndIndex.value = props.items.length;
  // Get the heights immedidiately after dom updates
  nextTick(() => {
    Array.from(scrollBox.value.children).forEach((item) => {
      updateItemHeights(item);
    });
  });
};

// remove specific items from the items array
const onItemsDelete = () => {
  itemHeights.value = itemHeights.value.filter(height=> {
    let state = false;
    props.items.forEach(item=> {
      if (height[props.uniqueKey] == item[props.uniqueKey]) {
        state = true;
      }
    });
    return state;
  })
};

let prevLen = 0;
// Set new items when added
watch(
  () => props.items,
  (newItems) => {
    const newLen = newItems.length;
    // If the items array empty
    if (!newLen) {
      itemHeights.value = [];
      visibleStartIndex.value = 0;
      visibleEndIndex.value = 1000;
    }
    // New items added
    else if (newLen > prevLen) {
      onItemsAdd();
      prevLen = newLen;
    } else {
      // items removed
      onItemsDelete();
    }
  }, {
    deep: true
  }
);

const containerHeight = computed(() => container.value?.clientHeight || 0);
// Function to handle the scroll event
function handleScroll() {
  const { scrollTop: sT, clientHeight: cH, scrollHeight: sH } = container.value;

  let heightSum = 0;
  let startIndex = 0;
  let endIndex = 1000;
  itemHeights.value.forEach((height, index) => {
    heightSum += height.value;
    if (heightSum < sT) {
      startIndex = index;
    }
    if (heightSum < sT + containerHeight.value) {
      endIndex = index;
    }
  });

  if (sT + cH >= sH) {
    // Get new items when at the bottom and a request is not pending yet
    if (props.totalItems > props.items.length && !props.pendingState) {
      emit("getNewItems");
    }
  } else {
    // Set start and end index only when there's still space to scroll down
    visibleEndIndex.value = Math.min(
      props.items.length,
      endIndex + props.bufferSize
    );
    visibleStartIndex.value = Math.max(0, startIndex - props.bufferSize);
  }
}
onMounted(() => {
  emit("getNewItems");
});
</script>
<style>
.virtual-scroll-container {
  overflow-y: auto;
  position: relative;
}
</style>
