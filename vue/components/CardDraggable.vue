<template>
  <a-card :bordered="false" class="card-div">
    <div class="card-header">{{data.title}}</div>
    <div v-for="index in data.name.length" :key="index">
      <div class="card-text-header">{{data.name[index-1]}}</div>
      <div v-if="data.title === 'System' && index === 4" class="card-text-content"><a-progress type="line" :percent="data.data[index-1]"/></div>
      <div v-else-if="data.title === 'System' && index === 5" class="card-text-content"><a-progress type="line" :percent="data.data[index-1]"/></div>
      <div v-else class="card-text-content">{{data.data[index-1]}}</div>
    </div>
    <div v-if="data.title === 'Recent system events'" class="card-link"><a href="#/status/syslog">See full system log</a></div>
    <div v-if="data.title === 'Recent network events'" class="card-link"><a href="#/status/dmesg">See full network log</a></div>
  </a-card>
</template>

<script>
export default {
  name: 'CardDraggable',
  props: ['data'],
  data () {
    return {
      changedAfterAction: 0,
      selectItem: ''
    }
  }
}
</script>
<style>
</style>
