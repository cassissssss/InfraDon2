<script lang="ts">

export default {
  data() {
    return {
      datas: [],
      databaseReference: null,
    };
  },
  methods: {
    inc() {
      // old
      // this.total++;
    },
    initDatabase() {
    },
    fetchData() {
    }
  },
  mounted() {
    this.initDatabase()
  }
}
</script>
<template>
  <h1>InfraDon2</h1>
  <p>Counter: {{ datas }}</p>
  <button type="button" role="button" @click="inc">
    +1
  </button>
</template>