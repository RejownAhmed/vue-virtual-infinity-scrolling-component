<script setup>
import {ref} from "vue";

import virtualInfinityScroll from './components/virtualInfinityScroll.vue'

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

const getNewItems = ()=> {
  preloader.value = true;
  generate(50, currentPage.value).then(res=> {
    items.value.push(...res);
    currentPage.value++;
  }).catch(error => {
    console.error("Error occurred:", error);
  }).finally(()=> preloader.value = false);

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
      <div>
        {{item.title}}
      </div>
    </template>
    <template #noData>
      Nothing found!
    </template>
    <template #loader>
      Loading ..... 
    </template>
  </virtual-infinity-scroll>
  </div>
</template>

<style scoped>
.scroller {
  min-height: 150px;
  max-height: 200px;
  width: 400px;
}

.container {
  display: flex;
  flex-direction: column;
  gap: 0.765rem;
}
</style>
