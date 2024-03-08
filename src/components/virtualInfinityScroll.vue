<template>
  <div ref="container" class="virtual-scroll-container" @scroll="handleScroll">
    <div style="position: fixed; top: 0">
      {{ visibleItems.length }}
    </div>
    <div :style="spacerStyle"></div>
    <div :style="scrollBoxStyle" :class="props.containerClass" ref="scrollBox">
      <template v-if="Items.length">
        <div
          class="virtual-scroll-item"
          v-for="(item, index) in visibleItems"
          :key="item.id"
          :data-key="item[props.uniqueKey]"
        >
          <slot :item="item" name="item">
            {{ item[props.uniqueKey] }}
          </slot>
        </div>
      </template>
      <template v-if="!props.pendingState && !Items.length">
        <slot name="noData"> No items available to show! </slot>
      </template>
      <template v-if="props.pendingState">
        <slot name="loader"> Loading... </slot>
      </template>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, nextTick, onMounted, toRefs, watch } from "vue";

const props = defineProps({
  Items: {
    type: Array,
    required: true,
  },
  totalItems: {
    type: Number,
    default: 1000,
  },
  uniqueKey: {
    type: String,
    default: "id",
  },
  bufferSize: {
    type: Number,
    default: 5,
  },
  deletedItems: Array,
  containerClass: String,
  pendingState: Boolean,
});

const { Items, deletedItems } = toRefs(props);

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
  props.Items.slice(visibleStartIndex.value, visibleEndIndex.value)
);

// Style for the spacer, it helps keep the scroll bar positioned properly
const spacerStyle = computed(() => {
  return {
    height: `${itemHeights.value.reduce((acc, height) => acc + height, 0)}px`,
  };
});

// Update the top of the contentContainer so that it stays
// in the right position when scrolled
const scrollBoxStyle = computed(() => ({
  position: "absolute",
  top: `${itemHeights.value
    .slice(0, Math.max(0, visibleStartIndex.value))
    .reduce((acc, height) => acc + height, 0)}px`,
  left: "0",
  right: "0",
}));

// Insert item heights for new items
const updateItemHeights = (item) => {
  const key = item.dataset.key;
  const index = props.Items.findIndex((i) => i[props.uniqueKey] == key);
  itemHeights.value[index] = item.offsetHeight;
};

// When new items added
const onItemsAdd = () => {
  visibleEndIndex.value = props.Items.length;
  // Get the heights immedidiately after dom updates
  nextTick(() => {
    Array.from(scrollBox.value.children).forEach((item) => {
      updateItemHeights(item);
    });
  });
};

// remove specific items from the items array
const onItemsDelete = () => {
  props.deletedItems.forEach((deletableItem) => {
    const index = props.Items.findIndex(
      (item) => item[props.uniqueKey] === deletableItem[props.uniqueKey]
    );
    props.Items.splice(index, 1);
  });
};

// Set new items when added
watch(
  () => props.Items,
  () => {
    if (props.Items?.length) {
      onItemsAdd();
    } else {
      itemHeights.value = [];
      visibleStartIndex.value = 0;
      visibleEndIndex.value = 1000;
    }
  }, {
    deep: true
  }
);

watch(deletedItems, () => {
  if (props.deletedItems?.length) {
    onItemsDelete();
  }
});

const containerHeight = computed(() => container.value?.clientHeight || 0);
// Function to handle the scroll event
function handleScroll() {
  const { scrollTop: sT, clientHeight: cH, scrollHeight: sH } = container.value;

  let heightSum = 0;
  let startIndex = 0;
  let endIndex = 1000;
  itemHeights.value.forEach((height, index) => {
    heightSum += height;
    if (heightSum < sT) {
      startIndex = index;
    }
    if (heightSum < sT + containerHeight.value) {
      endIndex = index;
    }
  });

  if (sT + cH >= sH) {
    // Get new items when at the bottom and a request is not pending yet
    if (props.totalItems > props.Items.length && !props.pendingState) {
      emit("getNewItems");
    }
  } else {
    // Set start and end index only when there's still space to scroll down
    visibleEndIndex.value = Math.min(
      props.Items.length,
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
