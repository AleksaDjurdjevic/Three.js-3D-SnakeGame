<template>
  <div class="paused-wrapper">
    <div class="menu-wrapper">
      <slot></slot>
    </div>
  </div>
</template>

<script>
export default {

}
</script>

<style scoped>
.paused-wrapper {
  display: flex;
  align-items: center;
  justify-content: center;
  position:fixed;
  padding:0;
  margin:0;

  top:0;
  left:0;

  z-index: 1;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.7);
  user-select: none;
}
.menu-wrapper {
  font-size: 1.6em;
  padding: 30px 70px;
  border-radius: 20px;
  background-color:  rgba(0, 0, 0, 0.8);
}
</style>