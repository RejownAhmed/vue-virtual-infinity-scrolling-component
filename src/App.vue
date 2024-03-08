<script setup>
import { ref } from "vue";

import virtualInfinityScroll from "./components/virtualInfinityScroll.vue";

const startId = ref(1);

function generate(perPage = 10) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      const items = [];
      for (let i = 0; i < perPage; i++) {
        const id = startId.value;
        const title = `Item ${id}`;
        items.push({ id, title });
        startId.value++;
      }

      resolve(items);
    }, 1000); // Delay of 1 second (1000 milliseconds)
  });
}

const items = ref([]);
const preloader = ref(false);
const totalItems = ref(1000);
const deletedItems = ref([]);
const currentPage = ref(1);

const getNewItems = () => {
  preloader.value = true;
  generate(50, currentPage.value)
    .then((res) => {
      items.value.push(...res);
      currentPage.value++;
    })
    .catch((error) => {
      console.error("Error occurred:", error);
    })
    .finally(() => (preloader.value = false));
};

const deleteItem = (item)=> {
  items.value.splice(items.value.findIndex(i=> i.id == item.id), 1);
}
</script>

<template>
  <div>
    <virtual-infinity-scroll
      class="scroller"
      container-class="container"
      :Items="items"
      unique-key="id"
      :total-items="totalItems"
      :pending-state="preloader"
      :buffer-size="5"
      :deleted-items="deletedItems"
      @getNewItems="getNewItems()"
    >
      <template #item="{ item }">
        <div class="item">
          {{ item.title }}

          <button @click="deleteItem(item)">Delete</button>
        </div>
      </template>
      <template #noData> Nothing found! </template>
      <template #loader> Loading ..... </template>
    </virtual-infinity-scroll>
  </div>
</template>

<style>
.scroller {
  min-height: 150px;
  max-height: 300px;
  width: 300px;
  border: 1px solid rgb(84, 95, 85);
}

.container {
  display: flex;
  flex-direction: column;
  gap: 0.765rem;
  padding: 20px 10px;
}

.item {
  display: flex;
  gap: 1rem;
  align-items: center;
  justify-content: space-between;
  border: 1px solid #999;
  padding: 3px 10px;
  border-radius: 10px;
}

</style>
