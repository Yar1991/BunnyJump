<script setup lang="ts">
import { onMounted } from "vue";
import { $ref } from "vue/macros";
import { animate } from "motion";

const emit = defineEmits<{ (e: "start"): void }>();
const heading = $ref<HTMLHeadingElement>();
const cloud = $ref<HTMLElement>();

function startGame() {
  emit("start");
}

onMounted(() => {
  const letters = [...heading.children];
  letters.forEach((letter, index) => {
    const delayTime = Number(`0.${index}`);
    animate(letter, {
      y: ["-200%", "200%", "0%"],
      rotateZ: ["400deg", "180deg", "0deg"],
      opacity: [0, 1],
    }, { delay: delayTime, duration: 0.5, easing: "ease-out" });
  });

  animate(cloud, {
    y: ["200%", "150%", "100%", "50%", "0%"],
    opacity: [0, 1]
  }, { duration: 1, delay: 1, easing: "ease" });

});
</script>

<template>
  <section id="home-section">
    <h1 ref="heading" class="heading">
      <span>B</span><span>u</span><span>n</span><span>n</span><span>y&nbsp;</span>
      <span>J</span><span>u</span><span>m</span><span>p</span>
    </h1>
    <div ref="cloud" class="start-box">
      <svg viewBox="0 0 100 100" fill="none" xmlns="http://www.w3.org/2000/svg">
        <path
          d="M82.8901 43.8779C88.52 37.3797 90.4733 29.8115 88.3916 22.3265C85.9 13.3766 78.0367 6.08676 68.357 3.75179C59.0054 1.50354 49.5839 4.35844 42.279 11.6016C33.759 4.8352 24.0357 2.75687 15.3925 5.9018C7.94934 8.61333 2.56768 14.9067 1.34951 22.3265C0.59287 26.9347 0.881153 35.7197 10.6078 44.4078C3.1628 49.5744 -0.565533 56.4245 0.0694308 63.8893C0.827832 72.8093 7.79934 81.3441 17.4158 85.1291C26.6406 88.7623 36.1789 87.2908 43.8505 81.1257C51.9271 91.4939 61.4869 96.9988 71.0668 96.9988C73.7285 96.9988 76.3934 96.5738 79.0283 95.7088C90.9915 91.7822 100.006 79.3857 99.9997 66.8759C99.9933 56.5061 93.795 48.2361 82.8901 43.8779Z"
          fill="black" />
      </svg>
      <div class="btn-box">
        <div class="btn-layer"></div>
        <button @click="startGame">Play</button>
      </div>
    </div>
  </section>
</template>

<style lang="scss" scoped>
#home-section {
  position: relative;
  height: 100%;
  min-width: 100vw;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;


  .heading {
    position: absolute;
    top: 3.5rem;
    left: 50%;
    translate: -50% 0;
    margin: auto;
    font-size: 5rem;
    color: hsl(0, 0%, 90%);
    text-shadow: 1px 2px 10px hsla(0, 0%, 0%, 0.3);
    display: flex;
    align-items: center;
    gap: 3px;

    span {
      display: inline-block;
    }
  }

  .start-box {
    position: relative;
    width: 25rem;
    height: 20rem;
    display: flex;
    align-items: center;
    justify-content: center;

    svg {
      position: absolute;
      width: 100%;
      height: 100%;
      inset: 0;
      scale: 1.2;
      filter: drop-shadow(0 0 10px hsla(0, 0%, 0%, 0.3)) drop-shadow(0 0 15px hsla(0, 0%, 0%, 0.2));

      path {
        fill: hsl(0, 0%, 90%);
      }
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
  }

}
</style>