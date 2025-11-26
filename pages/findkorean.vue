<style>
.problem {
  border: 1px solid black;
}
.word {
  display: inline-block;
}
</style>

<template>
  <Topbar></Topbar>
  <input type="text" v-model="search_term" @keyup.enter="entered" />
  <div id="problem_lists">
    <div class="problem" v-for="problem of problems" :key="problem">
      <div>
        <b>{{ problem.year }} {{ problem.month }} {{ problem.number }}</b>
      </div>
      <div>
        <span
          class="word"
          v-for="word of problem.article.split(' ')"
          :key="word"
        >
          <div
            v-if="
              (() => {
                console.log(search_term, word);
                return word.includes(search_term);
              })()
            "
            style="background-color: green"
          >
            {{ word }}&nbsp;
          </div>
          <div v-else>{{ word }}&nbsp;</div>
        </span>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch } from 'vue';

let problems = ref([]);
let search_term = ref('');

function entered() {
  fetch(serverURL + `findenglish?string=${search_term.value}`)
    .then((res) => res.json())
    .then((data) => {
      problems.value = data.problems;
    });
}
</script>
