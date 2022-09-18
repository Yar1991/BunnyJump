<script setup lang="ts">
import { $ref } from "vue/macros";
import { defineAsyncComponent, onMounted, onUnmounted } from "vue";
import { useResizeObserver } from "@vueuse/core";
import Home from './components/Home.vue';

const appPage = $ref<HTMLElement>();
let showGame = $ref<boolean>(false);
let showGameTimeout = $ref<number | null>();
const Game = defineAsyncComponent(() => import("./components/Game.vue"));
let gameStatus = $ref<string>();

function start() {
  // TODO: figure out why the page scrolls when resizing the window...
  const pageWidth = appPage.scrollWidth;
  appPage.scroll({
    top: 0,
    left: pageWidth,
    behavior: "smooth"
  });
  gameStatus = "start";
}



onMounted(() => {
  showGameTimeout = setTimeout(() => {
    showGame = true;
  }, 2000);

  useResizeObserver(appPage, (entries) => {
    if (appPage.scrollLeft > 500) {
      appPage.scroll({
        top: 0,
        left: 10000,
      });
    }

  });
});

onUnmounted(() => {
  clearTimeout(showGameTimeout as number);
});

</script>

<template>
  <section ref="appPage" id="main-section">
    <div class="bg-box">
      <svg xmlns="http://www.w3.org/2000/svg" version="1.1" xmlns:xlink="http://www.w3.org/1999/xlink"
        xmlns:svgjs="http://svgjs.com/svgjs" preserveAspectRatio="none" viewBox="0 0 1440 560">
        <g mask="url(&quot;#SvgjsMask1009&quot;)" fill="none">
          <rect width="1440" height="560" x="0" y="0" fill="rgba(30, 102, 72, 1)"></rect>
          <path
            d="M1536 560L0 560 L0 366.64Q21.58 268.22, 120 289.8Q197.44 247.24, 240 324.67Q286.06 298.73, 312 344.78Q330.58 291.36, 384 309.94Q414.61 268.55, 456 299.16Q526.84 250, 576 320.83Q593.45 266.28, 648 283.74Q691.14 254.88, 720 298.02Q770.53 276.55, 792 327.08Q872.78 287.86, 912 368.64Q966.79 303.43, 1032 358.22Q1058.66 312.88, 1104 339.55Q1158.71 322.26, 1176 376.97Q1228.01 308.99, 1296 361Q1346.02 291.02, 1416 341.04Q1494.33 299.37, 1536 377.69z"
            fill="rgba(24, 93, 67, 1)"></path>
          <path
            d="M1512 560L0 560 L0 441.36Q15.99 337.35, 120 353.33Q193.59 306.92, 240 380.52Q286.15 354.67, 312 400.83Q357.22 374.05, 384 419.27Q396.53 359.8, 456 372.33Q497.81 342.14, 528 383.95Q573.97 309.91, 648 355.88Q713.12 301, 768 366.11Q841.79 319.9, 888 393.7Q940.42 326.12, 1008 378.54Q1058.35 308.88, 1128 359.23Q1183.25 342.48, 1200 397.73Q1284.35 362.08, 1320 446.43Q1325.59 380.02, 1392 385.61Q1445.87 319.48, 1512 373.34z"
            fill="rgba(37, 125, 92, 1)"></path>
          <path
            d="M1464 560L0 560 L0 513.84Q-3.72 438.13, 72 434.41Q152.26 394.68, 192 474.94Q201.45 412.39, 264 421.84Q337.09 374.93, 384 448.02Q464.77 408.79, 504 489.57Q573.55 439.12, 624 508.67Q648.85 413.52, 744 438.37Q772.85 395.22, 816 424.06Q910.18 398.24, 936 492.42Q947.14 431.56, 1008 442.71Q1051.6 414.31, 1080 457.91Q1156.26 414.18, 1200 490.44Q1201.81 420.25, 1272 422.05Q1338.19 368.25, 1392 434.44Q1465.48 435.93, 1464 509.41z"
            fill="rgba(53, 177, 111, 1)"></path>
          <path
            d="M1512 560L0 560 L0 555.11Q45.71 528.82, 72 574.52Q107.42 489.94, 192 525.37Q267.01 480.38, 312 555.4Q362.62 534.02, 384 584.64Q413.7 494.34, 504 524.04Q526.38 474.41, 576 496.79Q619.46 468.25, 648 511.72Q745.61 489.33, 768 586.94Q797.36 544.3, 840 573.65Q890.41 504.06, 960 554.47Q990.88 465.35, 1080 496.23Q1147.47 443.7, 1200 511.17Q1286.63 477.81, 1320 564.44Q1374.28 498.72, 1440 553Q1465.61 506.61, 1512 532.22z"
            fill="rgba(241, 241, 241, 1)"></path>
        </g>
        <defs>
          <mask id="SvgjsMask1009">
            <rect width="1440" height="560" fill="#ffffff"></rect>
          </mask>
        </defs>
      </svg>
    </div>
    <Home @start="start" />
    <div v-if="showGame">
      <Game :status="gameStatus" @play="gameStatus = 'play'" @pause="gameStatus = 'pause'" />
    </div>
  </section>
</template>

<style lang="scss">
#main-section {
  position: relative;
  height: 100%;
  display: flex;
  overflow: hidden;

  &::-webkit-scrollbar {
    display: none;
  }

  .bg-box {
    position: absolute;
    inset: 0;
    width: 200vw;
    height: 100%;
    transform-origin: 100% 50%;
    animation: anim-bg 1.2s ease-out both;

    @keyframes anim-bg {
      from {
        scale: 2;
      }

      to {
        scale: 1;
      }
    }

    svg {
      width: 100%;
      height: 100%;
    }
  }
}
</style>
