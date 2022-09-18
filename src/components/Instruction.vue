<script setup lang="ts">

const props = defineProps<{ status?: string }>();
const emit = defineEmits<{ (e: "play"): void }>();

function playGame() {
  emit("play");
}
</script>

<template>
  <div :class="['play', 'pause'].includes(props.status as string) ? 'instruction-box hidden' : 'instruction-box'">
    <div class="content-box">
      <h3>Use left and right arrow keys to move bunny</h3>
      <div class="content-arrows" />
    </div>
    <div class="btn-box">
      <div class="btn-layer"></div>
      <button @click="playGame">Ok</button>
    </div>
  </div>
</template>


<style lang="scss" scoped>
.instruction-box {
  position: absolute;
  width: 100%;
  height: 100%;
  inset: 0;
  background-color: hsla(0, 0%, 0%, 0.8);
  z-index: 1;
  opacity: 1;
  visibility: visible;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2rem;
  justify-content: center;
  transition: opacity 0.5s ease-in, visibility 0.5s ease-in;

  &.hidden {
    opacity: 0;
    visibility: hidden;
  }

  .btn-box {
    position: relative;
    border-radius: 1rem;

    .btn-layer {
      position: absolute;
      inset: 0;
      width: 100%;
      height: 100%;
      border-radius: 1rem;
      background-color: #0C6234;
      translate: 0 0.3rem;
      box-shadow: 1px 2px 10px hsla(0, 0%, 0%, 0.5);
    }

    button {
      position: relative;
      border: none;
      border-radius: 1rem;
      z-index: 1;
      background-color: #118044;
      font-size: 1.5rem;
      font-weight: 500;
      letter-spacing: 1px;
      font-family: inherit;
      color: #fff;
      padding: 1rem 2rem;
      cursor: pointer;
      transition: background-color 0.2s ease, translate 0.2s ease;

      &:hover {
        background-color: #0C914A;
      }

      &:active {
        translate: 0 0.3rem;
        transition: background-color 0.2s ease, translate 0.1s ease;
      }
    }
  }

  .content-box {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 1rem;

    h3 {
      color: hsl(0, 0%, 97%);
      font-size: 2rem;
      font-weight: 400;
      max-width: 70%;
      text-align: center;
      letter-spacing: 1px;
    }

    .content-arrows {
      width: 10rem;
      height: 10rem;
      background-image: url("/assets/keys.webp");
      background-repeat: no-repeat;
      background-position: center center;
      background-size: cover;
    }
  }
}
</style>