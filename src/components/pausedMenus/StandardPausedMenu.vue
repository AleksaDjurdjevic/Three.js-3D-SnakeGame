<template>
  <div class="menu-wrap">
    <h1>Paused</h1>
    <p @click="$emit('resume')">Resume</p>
    <p @click="$emit('new-game')">New Game</p>
    <p @click="$emit('save-game')">Save game <span style="font-size: 0.6em;">(not here yet!)</span></p>
    <p @click="$emit('load-game')">Load game <span style="font-size: 0.6em;">(not here yet!)</span></p>
    <p @click="$emit('resume')">Exit</p>
  </div>
</template>

<script>
export default {

}
</script>

<style scoped>
.menu-wrap {
  display: flex;
  flex-direction: column;
  color: white;
}
.menu-wrap p {
  cursor: pointer;
}
</style>