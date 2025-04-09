<script setup lang="ts">
import { onMounted, ref } from 'vue';
import axios from 'axios';


const items = ref([]);

onMounted(async () => {
  try
  {
  let _response = await axios.get('https://pokeapi.co/api/v2/pokemon/ditto')
/*  let _response = await axios.post('http://127.0.0.1:8000/register', {
      email: 'hello@baby.com'
    }, {
      headers: {
        'Accept': 'application/json'
      }
    })
    console.log(_response.data)
  /*/
  .then((data) => {
    console.log(data)
    items.value = data.data;
  })
  .catch((data) => {
    console.error(data)
  })
    //*/
  } catch (error) {
    console.error(error)
  }

})
</script>

<template>
    <div>
      <h1>Hello</h1>




      {{ items }}
      <hr>

      <ul>
        <li v-for="item in items" :key="item.id">
          {{ item.name }}
        </li>
      </ul>
      
    </div>

</template>
