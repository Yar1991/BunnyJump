<script setup lang="ts">
import { ref, onMounted, onUnmounted, watch } from "vue";
import Phaser, { Scene } from "phaser";
import Instruction from "./Instruction.vue";

const props = defineProps<{ status?: string }>();
const emit = defineEmits<{ (e: "play"): void, (e: "pause"): void }>();


// main settings:
let gameInit = $ref<Phaser.Game>();
let gameRef = $ref<Phaser.Scene>();
let gameBox = $ref<HTMLDivElement>();
let gameConfig = $ref<Phaser.Types.Core.GameConfig>();
let keysPressed = $ref<Phaser.Types.Input.Keyboard.CursorKeys>();
let platformScrollIndex = $ref<number>();
let scrollStep = $ref<number>(180);
let score = $ref<number>(0);
let scoreStep = $ref<number>(50);
let lives = $ref<number>(3);
let gameOver = $ref<boolean>(false);
let gameSound = $ref<boolean>(true);

// to add small platforms and golden carrots:
let addGoldenCarrots = $ref<boolean>(false);
let pickedGold = $ref<boolean>(false);
let goldenCarrotsInterval = $ref<number | null>();
// to add snowy platform:
let addSnowyPlatform = $ref<boolean>(false);
let snowyPlatformInterval = $ref<number | null>();
// to add broken platform:
let addBrokenPlatform = $ref<boolean>(false);
let brokenPlatformInterval = $ref<number | null>();
let brokenParticles = $ref<Phaser.GameObjects.Particles.ParticleEmitterManager>();
let emitBrokenParticles = $ref<Phaser.GameObjects.Particles.ParticleEmitter>();
// check the platform type:
let platformType = $ref<string>();


// game objects:
let platforms = $ref<Phaser.Physics.Arcade.StaticGroup>();
let player = $ref<Phaser.Physics.Arcade.Sprite>();
let jetIcon = $ref<Phaser.Physics.Arcade.Image>();
let jet = $ref<Phaser.Physics.Arcade.Sprite>();
let jetUseTime = $ref<number>(0);
let jetParticles = $ref<Phaser.GameObjects.Particles.ParticleEmitterManager>();
let emitJetParticles = $ref<Phaser.GameObjects.Particles.ParticleEmitter>();
let enemiesWings = $ref<Phaser.Physics.Arcade.Group>();
let enemiesWingsTimer = $ref<number | null>();
let enemiesSpike = $ref<Phaser.Physics.Arcade.Group>();
let enemiesSpikeTimer = $ref<number | null>();
let enemySpikePlatformPair = $ref<{ enemy: number, platform: { left: number, right: number } }[]>([]);
let carrots = $ref<Phaser.Physics.Arcade.StaticGroup>();
let spikes = $ref<Phaser.Physics.Arcade.StaticGroup>();
let hitTween = $ref<Phaser.Tweens.Tween>();
// sounds:
let jumpSound = $ref<Phaser.Sound.BaseSound>();
let carrotSound = $ref<Phaser.Sound.BaseSound>();
let goldSound = $ref<Phaser.Sound.BaseSound>();
let hurtSound = $ref<Phaser.Sound.BaseSound>();
let breakSound = $ref<Phaser.Sound.BaseSound>();
let jetSound = $ref<Phaser.Sound.BaseSound>();
let pickJetSound = $ref<Phaser.Sound.BaseSound>();

function preload(this: Phaser.Scene): void {

  // images:
  this.load.image("bg", "/assets/bg_layer1.png");
  this.load.atlas("bunny", "/assets/bunny_sprite2.png", "/assets/bunny_sprite2.json");
  this.load.image("jet_icon", "/assets/jetpack_item.png");
  this.load.image("jet", "/assets/jetpack.png");
  this.load.image("platform", "/assets/ground_grass.png");
  this.load.image("platform_small", "/assets/ground_grass_small.png");
  this.load.image("platform_snowy", "/assets/ground_snow.png");
  this.load.image("platform_broken", "/assets/ground_wood_small_broken.png");
  this.load.image("broken_pieces", "/assets/broken_pieces.png");
  this.load.image("carrot", "/assets/carrot.png");
  this.load.image("carrot_gold", "/assets/carrot_gold.png");
  this.load.image("spikes_bottom", "/assets/spikes_bottom.png");
  this.load.atlas("enemy_wings", "/assets/enemy_wings.png", "/assets/enemy_wings.json");
  this.load.atlas("enemy_spike", "/assets/enemy_spike.png", "/assets/enemy_spike.json");
  this.load.image("jet_particles", "/assets/jet_particles.png");

  // sounds:
  this.load.audio("jump", "/assets/sounds/jump.mp3");
  this.load.audio("carrot", "/assets/sounds/pick_carrot.mp3");
  this.load.audio("gold", "/assets/sounds/pick_gold.mp3");
  this.load.audio("hurt", "/assets/sounds/hurt.mp3");
  this.load.audio("break", "/assets/sounds/break.mp3");
  this.load.audio("jet", "/assets/sounds/jet.mp3");
  this.load.audio("pick_jet", "/assets/sounds/pick_jet.mp3");

}

function create(this: Phaser.Scene): void {

  gameRef = this;
  props.status !== "play" && this.scene.pause();

  // adding background:
  this.add.image(100, 100, "bg").setScrollFactor(1, 0);

  // adding platforms:
  platforms = this.physics.add.staticGroup();
  addPlatforms(5);
  platformScrollIndex = platforms.children.entries.length - 2;

  // broken platform particles:
  brokenParticles = this.add.particles("broken_pieces");
  emitBrokenParticles = brokenParticles.createEmitter({
    on: false,
    speed: 100,
    frequency: 300,
    gravityY: 200,
    accelerationY: 300,
    blendMode: "NORMAL",
    lifespan: 1000,
    scale: 0.5,
    quantity: 20,
  });
  // adding player:
  player = this.physics.add.sprite(this.renderer.width / 2, this.renderer.height / 2, "bunny", "bunny_jump");
  player.setScale(0.4);
  player.setDepth(2);


  jetParticles = this.add.particles("jet_particles");
  jetParticles.setDepth(1);
  emitJetParticles = jetParticles.createEmitter({
    on: false,
    speed: 50,
    frequency: 50,
    blendMode: "NORMAL",
    lifespan: 100,
    quantity: 1,
    angle: 90,
    alpha: { start: 1, end: 0 },
    scale: { start: 0.7, end: 1.5 },
  });

  // adding enemies:
  enemiesWings = this.physics.add.group({
    allowGravity: false
  });
  enemiesSpike = this.physics.add.group();
  addEnemies();

  // adding carrots:
  carrots = this.physics.add.staticGroup();
  addCarrots();

  // adding spikes:
  spikes = this.physics.add.staticGroup();


  // player animations:
  this.anims.create({
    key: "jump",
    frames: [
      { key: "bunny", frame: "bunny_stand" },
      { key: "bunny", frame: "bunny_ready" },
      { key: "bunny", frame: "bunny_jump" },
    ],
    frameRate: 10
  });
  hitTween = this.tweens.add({
    targets: player,
    duration: 0.5,
    alpha: 0.5,
    ease: "Power2",
    yoyo: true,
    repeat: 10
  });

  // wings enemies animations:
  this.anims.create({
    key: "fly",
    frames: [
      { key: "enemy_wings", frame: "wingMan1" },
      { key: "enemy_wings", frame: "wingMan2" },
      { key: "enemy_wings", frame: "wingMan3" },
      { key: "enemy_wings", frame: "wingMan4" },
      { key: "enemy_wings", frame: "wingMan5" },
    ],
    frameRate: 15,
    repeat: -1
  });
  // spike enemies animations:
  this.anims.create({
    key: "walk_left",
    frames: [
      { key: "enemy_spike", frame: "spikeMan_walk_left" },
      { key: "enemy_spike", frame: "spikeMan_walk_left2" }
    ],
    frameRate: 5,
    repeat: -1
  });
  this.anims.create({
    key: "walk_right",
    frames: [
      { key: "enemy_spike", frame: "spikeMan_walk_right" },
      { key: "enemy_spike", frame: "spikeMan_walk_right2" }
    ],
    frameRate: 5,
    repeat: -1
  });

  // camera moving with the player:
  this.cameras.main.startFollow(player, false, 0, 1);

  // adding colliders:
  player.setCollideWorldBounds(true);
  this.physics.world.checkCollision.up = false;
  this.physics.world.checkCollision.down = false;
  this.physics.add.collider(player as Phaser.Physics.Arcade.Sprite, platforms as Phaser.Physics.Arcade.StaticGroup, jump);
  player.body.checkCollision.up = false;
  player.body.checkCollision.left = false;
  player.body.checkCollision.right = false;
  this.physics.add.collider(carrots as Phaser.Physics.Arcade.StaticGroup, platforms as Phaser.Physics.Arcade.StaticGroup);
  this.physics.add.collider(player as Phaser.Physics.Arcade.Sprite, spikes as Phaser.Physics.Arcade.StaticGroup, hitSpike);
  this.physics.add.collider(enemiesSpike as Phaser.Physics.Arcade.Group, platforms as Phaser.Physics.Arcade.StaticGroup, enemySpikeCollide);

  // adding overlaps:
  this.physics.add.overlap(player as Phaser.Physics.Arcade.Sprite, carrots as Phaser.Physics.Arcade.StaticGroup, collectCarrots);
  this.physics.add.overlap(player as Phaser.Physics.Arcade.Sprite, enemiesWings as Phaser.Physics.Arcade.Group, hitEnemy);
  this.physics.add.overlap(player as Phaser.Physics.Arcade.Sprite, enemiesSpike as Phaser.Physics.Arcade.Group, hitEnemy);


  // adding keys:
  keysPressed = this.input.keyboard.createCursorKeys();
  keysPressed.space.on("down", () => {
    if (lives <= 0 && gameOver) {
      gameOver = false;
      scrollStep = 180;
      score = 0;
      scoreStep = 50;
      lives = 3;
      this.scene.restart();
    }
  });

  // adding sounds:
  jumpSound = this.sound.add("jump", {
    volume: 0.7
  });
  carrotSound = this.sound.add("carrot", {
    volume: 0.7
  });
  goldSound = this.sound.add("gold", {
    volume: 0.3
  });
  hurtSound = this.sound.add("hurt", {
    volume: 0.7
  });
  breakSound = this.sound.add("break", {
    volume: 0.3
  });
  pickJetSound = this.sound.add("pick_jet", {
    volume: 0.7
  });
  jetSound = this.sound.add("jet", {
    volume: 0.5,
    loop: true
  });
}

function update(this: Phaser.Scene): void {

  jumpMoves();

  updatePlatform(this);

  fallDown(this);

  removeOffscreenObjects(this);

  checkLives(this);

  moveEnemyWings(this);

  moveEnemySpike();

  jetFly();

}

function addPlatforms(num: number): void {

  for (let i = 0; i < num; i++) {

    let x: number = Phaser.Math.Between(100, 600);
    let y: number = 150 * i;

    const platform: Phaser.Physics.Arcade.Sprite = platforms.create(x, y, "platform");
    platform.scale = 0.7;
    platform.body.updateFromGameObject();

    smallPlatform();
    snowyPlatform();
    brokenPlatforms();
  }

}

function smallPlatform(): void {
  goldenCarrotsInterval = setInterval(() => {
    addGoldenCarrots = props.status === "play" && true;
  }, 30000);
}

function snowyPlatform(): void {
  snowyPlatformInterval = setInterval(() => {
    addSnowyPlatform = props.status === "play" && true;
  }, 25000);
}

function brokenPlatforms(): void {
  brokenPlatformInterval = setInterval(() => {
    addBrokenPlatform = props.status === "play" && true;
  }, 27000);
}

function updatePlatform(scene: Phaser.Scene): void {
  const scrollY: number = scene.cameras.main.scrollY * -1;
  let x: number = Phaser.Math.Between(150, 500);
  let y: number = scene.cameras.main.scrollY - Phaser.Math.Between(20, 40);

  if (scrollY >= scrollStep && !addGoldenCarrots && !addSnowyPlatform && !addBrokenPlatform) {
    const newPlatform: Phaser.Physics.Arcade.Sprite = platforms.create(x, y, "platform");
    newPlatform.scale = 0.7;
    newPlatform.body.updateFromGameObject();
    let newCarrot: Phaser.Physics.Arcade.Sprite = carrots.create(x, y - 70, "carrot");
    newCarrot.setScale(0.6);
    scrollStep += 180;
  } else if (scrollY >= scrollStep && addGoldenCarrots) {
    const newPlatform: Phaser.Physics.Arcade.Sprite = platforms.create(x, y, "platform_small");
    const spike: Phaser.Physics.Arcade.Sprite = spikes.create(x, y + newPlatform.height / 2 - 5, "spikes_bottom");
    spike.scale = 0.5;
    spike.angle = 5;
    newPlatform.scale = 0.7;
    newPlatform.body.updateFromGameObject();
    spike.body.updateFromGameObject();
    let newCarrot: Phaser.Physics.Arcade.Sprite = carrots.create(x, y - 70, "carrot_gold");
    newCarrot.setScale(0.6);
    scrollStep += 180;
    addGoldenCarrots = false;
  } else if (scrollY >= scrollStep && addSnowyPlatform) {
    const newPlatform: Phaser.Physics.Arcade.Sprite = platforms.create(x, y, "platform_snowy");
    newPlatform.scale = 0.7;
    newPlatform.body.updateFromGameObject();
    let newCarrot: Phaser.Physics.Arcade.Sprite = carrots.create(x, y - 70, "carrot");
    newCarrot.setScale(0.6);
    scrollStep += 180;
    addSnowyPlatform = false;
  } else if (scrollY >= scrollStep && addBrokenPlatform) {
    const newPlatform: Phaser.Physics.Arcade.Sprite = platforms.create(x, y, "platform_broken");
    newPlatform.scale = 0.7;
    newPlatform.body.updateFromGameObject();
    let newCarrot: Phaser.Physics.Arcade.Sprite = carrots.create(x, y - 70, "carrot");
    newCarrot.setScale(0.6);
    scrollStep += 180;
    addBrokenPlatform = false;
  } else {
    return;
  }
}


function removeOffscreenObjects(scene: Phaser.Scene) {

  let targetPlatform;
  let targetSpike;
  platforms.children.entries.forEach((platform) => {
    const platformInView = scene.cameras.main.worldView.contains(platform.body.position.x, platform.body.position.y);
    if (!platformInView && platform.body.position.y - 500 > player.body.position.y) {
      targetPlatform = platform as Phaser.Physics.Arcade.Sprite;
      targetPlatform.destroy();
    }
  });

  spikes.children.entries.forEach((spike) => {
    const spikeInView = scene.cameras.main.worldView.contains(spike.body.position.x, spike.body.position.y);
    if (!spikeInView && spike.body.position.y - 500 > player.body.position.y) {
      targetSpike = spike as Phaser.Physics.Arcade.Sprite;
      targetSpike.destroy();
    }
  });

  enemiesWings.children.entries.forEach((enemy) => {
    const enemyInView = scene.cameras.main.worldView.contains(enemy.body.position.x, enemy.body.position.y);
    if (!enemyInView && enemy.body.position.y - 500 > player.body.position.y) {
      enemy.destroy();
    }
  });

  enemiesSpike.children.entries.forEach((enemy, index) => {
    const enemyInView = scene.cameras.main.worldView.contains(enemy.body.position.x, enemy.body.position.y);
    if (!enemyInView && enemy.body.position.y - 500 > player.body.position.y) {
      enemy.destroy();
      enemySpikePlatformPair = enemySpikePlatformPair.filter((pair) => pair.enemy !== index);
    }
  });

}

function addCarrots() {

  platforms.children.iterate((platform) => {

    let x: number = platform.body.position.x + 100;
    let y: number = platform.body.position.y - 30;

    let newCarrot: Phaser.Physics.Arcade.Sprite = carrots.create(x, y, "carrot");
    newCarrot.setScale(0.6);

  });

}

function collectCarrots(player: any, carrot: any) {

  if (carrot!.texture!.key === "carrot_gold") {
    goldSound.play();
    score += 10;
    pickedGold = true;
  } else {
    carrotSound.play();
    score++;
  }
  carrot.destroy();
  setTimeout(() => {
    pickedGold = false;
  }, 500);
}

function addEnemies() {

  enemiesWingsTimer = setInterval(() => {
    if (props.status === "play") {
      const posY = enemiesWings?.children?.entries.at(-1) ? enemiesWings.children.entries.at(-1)!.body.position.y - 500 : player.y - 500;
      let newEnemy: Phaser.Physics.Arcade.Sprite = enemiesWings.create(50, posY, "enemy_wings", "wingMan1");
      newEnemy.setScale(0.4);
      newEnemy.setCollideWorldBounds(true);
      newEnemy.setDepth(1);
      newEnemy.anims.play("fly");
    }
  }, 5000);

  enemiesSpikeTimer = setInterval(() => {
    if (props.status === "play") {
      const targetPlatform = platforms.children.entries.at(-1) as Phaser.Physics.Arcade.Sprite;
      const lastEnemy = enemiesSpike.children.entries?.at(-1);
      if ((targetPlatform.texture.key === "platform" || targetPlatform.texture.key === "platforms_snowy") && !lastEnemy) {
        const newEnemy: Phaser.Physics.Arcade.Sprite = enemiesSpike.create(targetPlatform.x - 100, targetPlatform.y - 100, "enemy_spike", "spikeMan_stand");
        newEnemy.setScale(0.4);
        newEnemy.setCollideWorldBounds(true);
        newEnemy.anims.play("walk_right");
      }
    }
  }, 4000);
}

function moveEnemyWings(game: Phaser.Scene) {
  if (enemiesWings?.children) {
    enemiesWings.children.entries.forEach((child) => {

      let enemy = child as Phaser.Physics.Arcade.Sprite;
      if (enemy.x <= 50) {
        enemy.setVelocityX(130);
      } else if (enemy.x >= game.renderer.width - 50) {
        enemy.setVelocityX(-130);
      } else {
        return;
      }

    });
  }
}

function moveEnemySpike() {
  if (enemiesSpike?.children && enemiesSpike.children.entries.length > 0) {
    enemySpikePlatformPair.forEach(({ enemy, platform }) => {
      const targetEnemy = enemiesSpike.children.entries[enemy] as Phaser.Physics.Arcade.Sprite;

      if (targetEnemy && targetEnemy.x + 40 >= platform.right) {
        targetEnemy.setVelocityX(-90);
        targetEnemy.anims.play("walk_left");
      }
      else if (targetEnemy && targetEnemy.x - 40 <= platform.left) {
        targetEnemy.setVelocityX(90);
        targetEnemy.anims.play("walk_right");
      }
    });
  }

}

function enemySpikeCollide(enemy: Phaser.Types.Physics.Arcade.ArcadeColliderType, platform: Phaser.Types.Physics.Arcade.ArcadeColliderType) {

  const targetEnemy = enemy as Phaser.Physics.Arcade.Sprite;
  const targetPlatform = platform as Phaser.Physics.Arcade.Sprite;
  const enemyIndex = enemiesSpike.children.entries.findIndex((item) => item === targetEnemy);
  const platformBounds = targetPlatform.getBounds();
  enemySpikePlatformPair.push({ enemy: enemyIndex, platform: { left: platformBounds.left, right: platformBounds.right } });
}

function hitEnemy(playerRef: Phaser.Types.Physics.Arcade.ArcadeColliderType, enemyRef: Phaser.Types.Physics.Arcade.ArcadeColliderType) {

  const enemy = enemyRef as Phaser.Physics.Arcade.Sprite;
  enemy.body.enable = false;
  player.setVelocityY(-10);
  player.setTexture("bunny", "bunny_hurt");
  hurtSound.play();
  hitTween.play();
  lives -= 1;

  setTimeout(() => {
    if (enemy?.body) enemy.body.enable = true;
  }, 2000);

}


function jump(playerRef: Phaser.Types.Physics.Arcade.ArcadeColliderType, platformRef: Phaser.Types.Physics.Arcade.ArcadeColliderType) {

  const platform = platformRef as Phaser.Physics.Arcade.Sprite;
  const x = platform.x;
  const y = platform.y;

  platformType = platform.texture.key;
  const touchingDown: boolean = player.body.touching.down;

  if (touchingDown && platformType === "platform_broken") {
    player.anims.play("jump");
    player.setVelocityY(-300);
    jumpSound.play();
    breakSound.play();
    platform.destroy();
    brokenParticles.emitParticleAt(x, y);
  } else if (touchingDown) {
    player.anims.play("jump");
    player.setVelocityY(-300);
    jumpSound.play();
  } else {
    return;
  }

}

function jumpMoves(): void {

  const touchingDown: boolean = player.body.touching.down;
  const wasTouching: boolean = player.body.wasTouching.down;
  const randNum: number = Phaser.Math.Between(-1, 1);
  const roundRandNum: number = randNum >= 0 ? -300 : 300;
  if (keysPressed.left.isDown && !touchingDown) {
    player.setVelocityX(-200);
  } else if (keysPressed.right.isDown && !touchingDown) {
    player.setVelocityX(200);
  } else if (!keysPressed.left.isDown && !keysPressed.right.isDown && wasTouching && platformType === "platform_snowy") {
    player.setVelocity(roundRandNum, -300);
  } else if (!keysPressed.left.isDown && !keysPressed.right.isDown && platformType !== "platform_snowy") {
    player.setVelocityX(0);
  } else {
    return;
  }

}

function hitSpike(targetPlayer: Phaser.Types.Physics.Arcade.ArcadeColliderType, targetSpike: Phaser.Types.Physics.Arcade.ArcadeColliderType) {

  const spike = targetSpike as Phaser.Physics.Arcade.Sprite;
  spike.body.enable = false;

  player.setVelocityY(10);
  player.setTexture("bunny", "bunny_hurt");
  hurtSound.play();
  hitTween.play();
  lives -= 1;

  setTimeout(() => {
    if (spike?.body) spike.body.enable = true;
  }, 2000);

}

function addJet() {
  if (jetUseTime === 0 && props.status === "play") {
    const posX = platforms.children.entries.at(-1)!.body.position.x + 20;
    const posY = player.y - 500;
    jetIcon = gameRef.physics.add.sprite(posX, posY, "jet_icon");
    gameRef.physics.add.collider(jetIcon as Phaser.Physics.Arcade.Sprite, platforms as Phaser.Physics.Arcade.StaticGroup);
    gameRef.physics.add.overlap(player as Phaser.Physics.Arcade.Sprite, jetIcon as Phaser.Physics.Arcade.Sprite, pickJet);
    jetIcon.setCollideWorldBounds(true);
    jetIcon.setScale(0.5);
  }
}

function pickJet(targetPlayer: Phaser.Types.Physics.Arcade.ArcadeColliderType, targetJet: Phaser.Types.Physics.Arcade.ArcadeColliderType) {
  const jet_icon = targetJet as Phaser.Physics.Arcade.Sprite;
  jetUseTime = 10;
  let timer: number;
  jet_icon.destroy();
  pickJetSound.play();
  jet = gameRef.physics.add.sprite(player.x, player.y, "jet");
  jet.setScale(0.4);
  jet.setDepth(1);
  timer = setInterval(() => {
    if (jetUseTime > 0) {
      jetSound.play();
      jetUseTime -= 1;
    } else {
      jetSound.pause();
      return clearInterval(timer);
    }
  }, 1000);
}

function jetFly() {
  if (jet) {
    if (jetUseTime > 0 && jet.active) {
      player.anims.play("jump");
      player.setVelocityY(-200);
      jet.setVelocityY(-200);
      jetParticles.emitParticleAt(jet.x, jet.y + 50);
      if (keysPressed.left.isDown) {
        player.setVelocityX(-200);
        jet.setVelocityX(-200);
      } else if (keysPressed.right.isDown) {
        player.setVelocityX(200);
        jet.setVelocityX(200);
      } else {
        player.setVelocityX(0);
        jet.setVelocityX(0);
      }
    } else {
      jet.destroy();
    }
  }
}

function fallDown(game: Phaser.Scene): void {

  const distance: number = Math.abs(platforms.children.entries.at(-1)!.body.position.y - player.y);

  if (distance > 1000) {
    const platform = platforms.children.entries.at(-1) as Phaser.Physics.Arcade.Sprite;
    platform.tint = 0x5F134B;
    player.setAlpha(0.3);
    platforms.setAlpha(0.3);
    carrots.setAlpha(0.3);
    spikes.setAlpha(0.3);
    enemiesWings.setAlpha(0, 3);
    enemiesWingsTimer = null;
    enemiesSpike.setAlpha(0.3);
    enemiesSpikeTimer = null;
    goldenCarrotsInterval = null;
    snowyPlatformInterval = null;

    game.physics.pause();
    gameRef.sound.stopAll();
    gameOver = true;
    score = 0;
    scoreStep = 50;
    lives = 3;

    if (keysPressed.space.isDown) {
      gameOver = false;
      scrollStep = 180;
      player.setAlpha(1);
      game.scene.restart();
    }
  }
}

function toggleSound() {

  if (gameSound) {
    gameSound = false;
    gameRef.sound.mute = true;
  } else {
    gameSound = true;
    gameRef.sound.mute = false;
  }

}

function pauseGame() {
  if (gameRef.scene.isPaused()) {
    gameRef.scene.resume()
    emit("play");
  } else {
    gameRef.scene.pause();
    emit("pause");
  }
}

function checkLives(game: Phaser.Scene) {

  if (lives <= 0) {
    game.physics.pause();
    gameRef.sound.stopAll();
    player.alpha = 0.3;
    platforms.setAlpha(0.3);
    carrots.setAlpha(0.3);
    spikes.setAlpha(0.3);
    enemiesWings.setAlpha(0.3);
    enemiesWingsTimer = null;
    enemiesSpike.setAlpha(0.3);
    enemiesSpikeTimer = null;
    jet && jet.setAlpha(0.3);
    goldenCarrotsInterval = null;
    snowyPlatformInterval = null;
    gameOver = true;
  }

}

const addJetWatcher = watch(() => score, () => {
  if (score >= scoreStep) {
    scoreStep += 50;
    addJet();
  }

});

const statusWatcher = watch(() => props.status, () => {
  if (props.status === "play") gameRef.scene.resume();
});


onMounted(() => {


  gameConfig = {
    title: "Bunny Jump",
    type: Phaser.AUTO,
    width: 800,
    height: 600,
    antialias: true,
    parent: gameBox,
    physics: {
      default: "arcade",
      arcade: {
        gravity: {
          y: 200
        },
        debug: false
      }
    },
    scene: {
      preload,
      create,
      update
    }
  }

  gameInit = new Phaser.Game(gameConfig as Phaser.Types.Core.GameConfig);

});

onUnmounted(() => {
  clearInterval(goldenCarrotsInterval as number);
  clearInterval(snowyPlatformInterval as number);
  clearInterval(brokenPlatformInterval as number);
  clearInterval(enemiesWingsTimer as number);
  clearInterval(enemiesSpikeTimer as number);
  // clearInterval(jetInterval as number);
  statusWatcher();
  addJetWatcher();
});

</script>

<template>
  <section id="game-section">
    <div ref="gameBox" class="game-box">
      <div class="score">
        <img src="/assets/carrot_score.png" alt="carrot">
        <span :class="pickedGold && 'gold'">{{ score }}</span>
      </div>
      <div class="lives">
        <svg v-for="life in lives" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink"
          version="1.1" id="Layer_1" x="0px" y="0px" viewBox="0 0 489.6 489.6"
          style="enable-background:new 0 0 489.6 489.6;" xml:space="preserve">
          <path style="fill:#27D16B;"
            d="M119.2,30.8c52,0,97.6,28.8,113.6,72.8c1.6,4,6.4,6.4,12,5.6c4.8,1.6,10.4-0.8,12-5.6  c16-43.2,61.6-72.8,113.6-72.8c65.6,0,119.2,47.2,119.2,104.8c0,20.8-7.2,41.6-20.8,59.2c0,0,0,0,0,0.8l-0.8,0.8L244.8,459.6  l-223.2-264l-0.8-0.8l0,0C7.2,177.2,0,157.2,0,135.6C0,78,53.6,30.8,119.2,30.8z" />
          <path style="fill:#25C763;"
            d="M244.8,108.4c4.8,1.6,10.4-0.8,12-5.6c16-43.2,61.6-72.8,113.6-72.8c65.6,0,119.2,47.2,119.2,104.8  c0,20.8-7.2,41.6-20.8,59.2c0,0,0,0,0,0.8l-0.8,0.8l-223.2,264" />
          <path style="fill:#95F9B3;"
            d="M37.6,157.2c-6.4,0-12-5.6-12-12c0-49.6,44.8-89.6,100-89.6c6.4,0,12,5.6,12,12s-5.6,12-12,12  c-41.6,0-76,29.6-76,65.6C49.6,152.4,44,157.2,37.6,157.2z" />
        </svg>
      </div>
      <div v-if="gameOver" class="game-over">
        <p>Game Over</p>
        <span>press "Space" to start over</span>
      </div>
      <button class="sound-btn" @click="toggleSound" title="sound">
        <svg v-if="gameSound" viewBox="0 0 22 20" fill="none" xmlns="http://www.w3.org/2000/svg">
          <path
            d="M15 6.50001C16.333 8.27801 16.333 11.722 15 13.5M18 3.00001C21.988 6.80801 22.012 13.217 18 17M1 12.959V7.04001C1 6.46601 1.448 6.00001 2 6.00001H5.586C5.71833 5.99954 5.8492 5.97228 5.97071 5.91986C6.09222 5.86744 6.20185 5.79095 6.293 5.69501L9.293 2.30701C9.923 1.65101 11 2.11601 11 3.04301V16.957C11 17.891 9.91 18.352 9.284 17.683L6.294 14.314C6.20259 14.2153 6.09185 14.1365 5.96867 14.0825C5.84549 14.0285 5.71251 14.0004 5.578 14H2C1.448 14 1 13.534 1 12.959Z"
            stroke="black" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
        </svg>
        <svg v-else viewBox="0 0 22 20" fill="none" xmlns="http://www.w3.org/2000/svg">
          <path d="M21 13L15 7M21 7L15 13" stroke="black" stroke-width="2" stroke-linecap="round" />
          <path
            d="M1 12.959V7.04001C1 6.46601 1.448 6.00001 2 6.00001H5.586C5.71833 5.99954 5.8492 5.97228 5.97071 5.91986C6.09222 5.86744 6.20185 5.79095 6.293 5.69501L9.293 2.30701C9.923 1.65101 11 2.11601 11 3.04301V16.957C11 17.891 9.91 18.352 9.284 17.683L6.294 14.314C6.20259 14.2153 6.09185 14.1365 5.96867 14.0825C5.84549 14.0285 5.71251 14.0004 5.578 14H2C1.448 14 1 13.534 1 12.959Z"
            stroke="black" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" />
        </svg>
      </button>
      <button class="pause-btn" @click="pauseGame" :title="props.status === 'play' ? 'pause' : 'play'">
        <svg v-if="props.status === 'pause'" viewBox="0 0 44 44" fill="none" xmlns="http://www.w3.org/2000/svg">
          <path
            d="M21.9323 0.0995483C17.6537 0.0995483 13.4712 1.3683 9.91362 3.74537C6.35609 6.12244 3.58333 9.50105 1.94598 13.454C0.308629 17.4069 -0.119777 21.7566 0.714937 25.953C1.54965 30.1494 3.61 34.004 6.63543 37.0294C9.66086 40.0549 13.5155 42.1152 17.7119 42.9499C21.9083 43.7846 26.258 43.3562 30.2109 41.7189C34.1638 40.0815 37.5424 37.3088 39.9195 33.7512C42.2965 30.1937 43.5653 26.0112 43.5653 21.7326C43.5653 18.8917 43.0057 16.0786 41.9186 13.454C40.8314 10.8293 39.2379 8.44452 37.2291 6.43571C35.2203 4.4269 32.8355 2.83342 30.2109 1.74626C27.5862 0.659103 24.7732 0.0995483 21.9323 0.0995483V0.0995483ZM21.9323 39.039C18.5094 39.039 15.1634 38.024 12.3174 36.1223C9.47133 34.2207 7.25312 31.5178 5.94324 28.3554C4.63336 25.1931 4.29064 21.7134 4.95841 18.3563C5.62618 14.9991 7.27445 11.9154 9.6948 9.49508C12.1151 7.07474 15.1989 5.42646 18.556 4.75869C21.9131 4.09092 25.3928 4.43364 28.5552 5.74352C31.7175 7.05341 34.4204 9.27161 36.322 12.1176C38.2237 14.9637 39.2387 18.3097 39.2387 21.7326C39.2387 26.3225 37.4153 30.7245 34.1698 33.97C30.9242 37.2156 26.5222 39.039 21.9323 39.039V39.039Z" />
          <path
            d="M22.6678 11.8896C22.1377 11.4011 21.4758 11.0791 20.7642 10.9637C20.0527 10.8484 19.323 10.9446 18.6657 11.2406C18.0276 11.4987 17.481 11.9411 17.0957 12.5114C16.7103 13.0817 16.5038 13.754 16.5024 14.4423V29.0229C16.5038 29.7112 16.7103 30.3834 17.0957 30.9537C17.481 31.524 18.0276 31.9665 18.6657 32.2246C19.1346 32.4373 19.6434 32.5479 20.1584 32.5491C21.0862 32.545 21.9799 32.1983 22.6678 31.5756L30.5855 24.2853C30.9396 23.961 31.2224 23.5666 31.4158 23.1271C31.6093 22.6877 31.7092 22.2127 31.7092 21.7326C31.7092 21.2524 31.6093 20.7775 31.4158 20.338C31.2224 19.8986 30.9396 19.5042 30.5855 19.1799L22.6678 11.8896ZM20.8506 27.3572V16.108L26.9295 21.7326L20.8506 27.3572Z" />
        </svg>
        <svg v-else-if="props.status === 'play'" viewBox="0 0 42 42" fill="none" xmlns="http://www.w3.org/2000/svg">
          <path
            d="M21 0.166687C16.8796 0.166687 12.8517 1.38854 9.42565 3.67774C5.99963 5.96693 3.32937 9.22065 1.75254 13.0274C0.175714 16.8342 -0.236856 21.0231 0.567003 25.0644C1.37086 29.1057 3.35504 32.8178 6.26864 35.7314C9.18223 38.645 12.8944 40.6292 16.9357 41.433C20.9769 42.2369 25.1658 41.8243 28.9726 40.2475C32.7794 38.6707 36.0331 36.0004 38.3223 32.5744C40.6115 29.1484 41.8334 25.1205 41.8334 21C41.8334 18.2641 41.2945 15.5551 40.2475 13.0274C39.2006 10.4998 37.666 8.20318 35.7314 6.26863C33.7969 4.33408 31.5002 2.7995 28.9726 1.75253C26.445 0.705557 23.7359 0.166687 21 0.166687V0.166687ZM21 37.6667C17.7037 37.6667 14.4813 36.6892 11.7405 34.8578C8.99971 33.0265 6.8635 30.4235 5.60204 27.3781C4.34058 24.3326 4.01052 20.9815 4.65361 17.7485C5.2967 14.5155 6.88404 11.5458 9.21492 9.21491C11.5458 6.88403 14.5155 5.29669 17.7485 4.6536C20.9815 4.01051 24.3327 4.34057 27.3781 5.60203C30.4235 6.86349 33.0265 8.9997 34.8579 11.7405C36.6892 14.4813 37.6667 17.7037 37.6667 21C37.6667 25.4203 35.9108 29.6595 32.7851 32.7851C29.6595 35.9107 25.4203 37.6667 21 37.6667V37.6667Z" />
          <path
            d="M27.25 12.6667C26.6975 12.6667 26.1676 12.8862 25.7769 13.2769C25.3862 13.6676 25.1667 14.1975 25.1667 14.75V27.25C25.1667 27.8026 25.3862 28.3325 25.7769 28.7232C26.1676 29.1139 26.6975 29.3334 27.25 29.3334C27.8026 29.3334 28.3325 29.1139 28.7232 28.7232C29.1139 28.3325 29.3334 27.8026 29.3334 27.25V14.75C29.3334 14.1975 29.1139 13.6676 28.7232 13.2769C28.3325 12.8862 27.8026 12.6667 27.25 12.6667Z" />
          <path
            d="M14.75 12.6667C14.1975 12.6667 13.6676 12.8862 13.2769 13.2769C12.8862 13.6676 12.6667 14.1975 12.6667 14.75V27.25C12.6667 27.8026 12.8862 28.3325 13.2769 28.7232C13.6676 29.1139 14.1975 29.3334 14.75 29.3334C15.3026 29.3334 15.8325 29.1139 16.2232 28.7232C16.6139 28.3325 16.8334 27.8026 16.8334 27.25V14.75C16.8334 14.1975 16.6139 13.6676 16.2232 13.2769C15.8325 12.8862 15.3026 12.6667 14.75 12.6667Z" />
        </svg>
      </button>
      <Instruction :status="props.status" @play="emit('play')" />

    </div>
  </section>
</template>

<style lang="scss">
#game-section {
  position: relative;
  height: 100%;
  min-width: 100vw;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;


  .game-box {
    position: relative;
    width: 800px;
    height: 600px;
    background-color: hsl(0, 0%, 10%);
    border-radius: 1rem;
    box-shadow: 0 0 15px hsla(0, 0%, 5%, 0.3), 0 0 15px hsla(0, 0%, 5%, 0.3);
    overflow: hidden;

    .score {
      position: absolute;
      top: 0.5rem;
      left: 0.5rem;
      display: flex;
      align-items: center;

      img {
        display: inline-block;
        width: 2.2rem;
        height: 2.2rem;
        object-fit: contain;
      }

      span {
        display: inline-block;
        font-family: inherit;
        font-size: 1.5rem;
        color: #159852;

        &.gold {
          animation: gold-score 0.4s ease-out reverse;
        }

        @keyframes gold-score {
          from {
            transform: scale(1);
          }

          to {
            transform: scale(1.5);
          }
        }
      }
    }


    .lives {
      position: absolute;
      top: 1rem;
      right: 0.5rem;
      display: flex;
      align-items: center;
      gap: 0.3rem;

      svg {
        display: inline-block;
        height: 1.5rem;
        width: 1.5rem;
      }
    }

    .game-over {
      position: absolute;
      inset: 0;
      margin: auto;
      width: fit-content;
      height: fit-content;
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 1rem;

      p {
        font-size: 5rem;
        font-weight: 600;
        color: #137D44;
        text-shadow: 1px 2px 2px hsla(0, 0%, 5%, 0.3);
      }

      span {
        display: inline-block;
        font-size: 1.1rem;
        color: #135F36;
        letter-spacing: 1px;
      }
    }

    .sound-btn {
      position: absolute;
      bottom: 1rem;
      left: 1rem;
      cursor: pointer;
      background: none;
      border: none;
      width: 2rem;
      height: 2rem;
      outline: none;
      will-change: scale;
      transition: scale 0.2s ease;

      &:hover svg path {
        stroke: #0DB159;
      }

      &:active {
        scale: 0.8;
      }

      svg {
        width: 100%;
        height: 100%;

        path {
          stroke: #149B53;
          transition: stroke 0.2s ease;
        }
      }
    }

    .pause-btn {
      position: absolute;
      bottom: 0.8rem;
      right: 0.8rem;
      cursor: pointer;
      background: none;
      border: none;
      width: 2.4rem;
      height: 2.4rem;
      outline: none;
      will-change: scale;
      transition: scale 0.2s ease;

      &:hover svg path {
        stroke: #0DB159;
        fill: #0DB159;
      }

      &:active {
        scale: 0.8;
      }

      svg {
        width: 100%;
        height: 100%;

        path {
          stroke: #149B53;
          stroke-width: 1px;
          fill: #149B53;
          transition: stroke 0.2s ease, fill 0.2s ease;
        }
      }
    }
  }
}
</style>
