<template>
  <div id="container" @keydown="setControls" @mouseup="setIsSpeedingUpTo(false, $event)" @mousedown="setIsSpeedingUpTo(true, $event)" tabindex="0" ref='game'>
    <paused-wrapper v-if="paused">
      <standard-paused-menu 
        @resume="resumeGame" 
        @new-game="newGame"
        @save-game="saveGame"
        @load-game="loadGame"
      />
    </paused-wrapper>
    <div class="score">Your Score: {{score}} Speed: {{Math.floor(speed * 600) + "Km/h"}}</div>
    <div class="weapons">
      <div class="selected-weapon">
        <span>{{selectedWeapon.name}}</span>
        <img :src="selectedWeapon.iconUrl" alt="">
      </div>
      <div class="all-weapons">
        <div class="each-weapon-icon" v-for="(weapon, index) in weapons" :key="index">
          <img :src="weapon.iconUrl" alt="">
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import * as Three from 'three';
import PausedWrapper from './pausedMenus/PausedWrapper'
import StandardPausedMenu from './pausedMenus/StandardPausedMenu'
import axios from 'axios';

export default {
  components: {
    PausedWrapper,
    StandardPausedMenu
  },
  data() {
    return {
      baseUrl: 'http://319g567.mars-t2.mars-hosting.com/api',
      camera: null,
      scene: null,
      renderer: null,
      snake: [],
      snakeHead: {},
      food: null,
      speed: 0.04,
      originalSpeed: 0.04,
      maxSpeed: 0.2,
      isSpeedingUp: false,
      weapons: [
        {
          name: 'Bombs',
          iconUrl: require('@/assets/bomb.png'),
          activate: () => {
            let geometry = new Three.BoxGeometry(0.5, 0.5, 0.5);
            let material = new Three.MeshLambertMaterial({
              color: 0xFF0000
            })

            let bomb = new Three.Mesh(geometry, material);
            this.generateRandomPositionFor(bomb)
            this.scene.add(bomb);
          }
        },
        {
          name: 'Bombs2',
          iconUrl: require('@/assets/bomb.png')
        },
        {
          name: 'Bombs3',
          iconUrl: require('@/assets/bomb.png')
        }
      ],
      selectedWeapon: {
        name: 'Bombs',
        iconUrl: require('@/assets/bomb.png'),
        activate: () => {
          let previousBomb = this.scene.getObjectByName('bomb', true);
          if (!previousBomb) {
            let geometry = new Three.BoxGeometry(3, 3, 3);
            let material = new Three.MeshLambertMaterial({
              color: 0xFF0000,
              transparent: true,
              opacity: 1
            })
            let bomb = new Three.Mesh(geometry, material);
            bomb.name = 'bomb'
            this.generateRandomPositionFor(bomb)
            this.scene.add(bomb);
          } else {
            alert("Bomb has already been planted.")
          }
        }
      },
      score: 0,
      direction: 'forwards',
      oldDirection: null,
      paused: false,
      clock: null
    }
  },
  methods: {
    init() {
      let container = document.getElementById('container');

      this.scene = new Three.Scene();

      let geometry = new Three.BoxGeometry(0.5, 0.5, 0.5);
      let material = new Three.MeshLambertMaterial ();

      let tempSnakeHead = new Three.Mesh(geometry, material);
      this.snake.push(tempSnakeHead);
      this.snakeHead = this.snake[0];
      this.snakeHead.direction = 'forward';
      this.snakeHead.move = lastCube => this.move(lastCube);
      this.snakeHead.name = 0;
      this.generateRandomPositionFor(this.snakeHead);
      this.scene.add(this.snakeHead);

      this.food = new Three.Mesh(geometry, material);
      this.generateRandomPositionFor(this.food);
      this.scene.add(this.food);

      let light = new Three.PointLight( 0xFFFFFF, 1, 100 );
      light.position.set( 0, 5, 0 );
      this.scene.add(light);

      this.camera = new Three.PerspectiveCamera(70, container.clientWidth/container.clientHeight, 0.01, 1000);

      let skyboxGeometry = new Three.BoxGeometry(10, 10, 10);
      let skyboxMats = [
        new Three.MeshBasicMaterial({map: new Three.TextureLoader().load('skybox/yellowcloud_ft.jpg'), side: Three.BackSide}),
        new Three.MeshBasicMaterial({map: new Three.TextureLoader().load('skybox/yellowcloud_bk.jpg'), side: Three.BackSide}),
        new Three.MeshBasicMaterial({map: new Three.TextureLoader().load('skybox/yellowcloud_up.jpg'), side: Three.BackSide}),
        new Three.MeshBasicMaterial({map: new Three.TextureLoader().load('skybox/yellowcloud_dn.jpg'), side: Three.BackSide}),
        new Three.MeshBasicMaterial({map: new Three.TextureLoader().load('skybox/yellowcloud_rt.jpg'), side: Three.BackSide}),
        new Three.MeshBasicMaterial({map: new Three.TextureLoader().load('skybox/yellowcloud_lf.jpg'), side: Three.BackSide})
      ];

      let skybox = new Three.Mesh(skyboxGeometry, skyboxMats);
      this.scene.add(skybox);
      
      this.renderer = new Three.WebGLRenderer({antialias: true});
      this.renderer.setSize(container.clientWidth, container.clientHeight);
      this.renderer.setClearColor('#e5e5e5');
      container.appendChild(this.renderer.domElement);

      this.clock = new Three.Clock();
      this.clock.start();
    },
    animate() {
      if(!this.paused){
        requestAnimationFrame(this.animate);
      }

      for (let i = 0; i < this.snake.length; i++) {
        this.snake[i].move(this.snake[i]);
      }

      for (let i = 1; i < this.snake.length; i++) {
        if(this.isCollisionBetween(this.snakeHead, this.snake[i]) && i > 1){
          this.restartGame();
        };
      }

      if (Math.floor(this.clock.getElapsedTime()) !== 0 && Math.floor(this.clock.getElapsedTime()) % 30 === 0) {
        this.selectedWeapon.activate();
        alert("Bomb has been planted.")
      }

      if (this.isSpeedingUp) {
        this.raiseSpeed();
      } else {
        this.dropSpeed();
      }

      this.handleBombCollision();

      this.changeCameraAngle();

      if(this.isCollisionBetween(this.snakeHead, this.food)){
        this.adjustScore();
        this.generateRandomPositionFor(this.food);
        this.growSnakeBehind(this.snake[this.snake.length - 1]);
      };
      
      this.activateGameBorders();
        
      this.camera.lookAt(this.snakeHead.position.x, this.snakeHead.position.y, this.snakeHead.position.z);
      this.camera.position.y = this.snakeHead.position.y + 1.5;

      this.renderer.render(this.scene, this.camera);
    },
    handleBombCollision () {
      let bomb = this.scene.getObjectByName('bomb', true); 
      if (bomb) {
        if (this.isCollisionBetween(this.snakeHead, bomb)) {
          this.score--;
          this.speed = this.originalSpeed;
          if (this.snake.length > 1) {
            this.scene.remove(this.snake[this.snake.length - 1])
            this.snake.pop();
          }
          this.scene.remove(bomb);
        }

        bomb.material.opacity -= 0.001;
        
        if (bomb.material.opacity <= 0) {
          this.scene.remove(bomb);
          alert("Bomb timed out!")
        }
      }
    },
    raiseSpeed () {
      if (this.speed > this.maxSpeed) {
        this.speed = this.maxSpeed;
      } else if (this.speed < this.maxSpeed) {
        this.speed += 0.002;
      }
    },
    dropSpeed () {
      if (this.speed < this.originalSpeed) {
        this.speed = this.originalSpeed;
      } else if (this.speed > this.originalSpeed) {
        this.speed -= 0.0005;
      }
    },
    generateRandomPositionFor(object){
      object.position.x = Math.random()*10 -5;
      object.position.z = Math.random()*10 -5;
      object.position.y = Math.random()*10 -5;
    },
    restartGame(){
      this.generateRandomPositionFor(this.snakeHead);
      for (let i = 1; i < this.snake.length; i++) {
        this.scene.remove(this.snake[i]);
      }
      let bomb = this.scene.getObjectByName('bomb', true)
      if (bomb) this.scene.remove(bomb)
      this.snake = this.snake.slice(0, 1);
      this.speed = this.originalSpeed;
      this.score = 0;
      this.clock.start()
    },
    growSnakeBehind(lastCube){
      let geometry = new Three.BoxGeometry(0.5, 0.5, 0.5);
      let material = new Three.MeshLambertMaterial ();

      let newSnakeCube = new Three.Mesh(geometry, material);

      newSnakeCube.direction = lastCube.direction;
      newSnakeCube.move = (lastSnakeCube) => {return this.move(lastSnakeCube)};
      newSnakeCube.name = this.snake.length;

      switch(lastCube.direction){
        case 'left':
          newSnakeCube.position.x = this.snake[this.snake.length - 1].position.x + 0.5;
          newSnakeCube.position.y = this.snake[this.snake.length - 1].position.y;
          newSnakeCube.position.z = this.snake[this.snake.length - 1].position.z;
          break;
        case 'backward':
          newSnakeCube.position.x = this.snake[this.snake.length - 1].position.x;
          newSnakeCube.position.y = this.snake[this.snake.length - 1].position.y;
          newSnakeCube.position.z = this.snake[this.snake.length - 1].position.z - 0.5; 
          break;
        case 'right':
          newSnakeCube.position.x = this.snake[this.snake.length - 1].position.x - 0.5;
          newSnakeCube.position.y = this.snake[this.snake.length - 1].position.y;
          newSnakeCube.position.z = this.snake[this.snake.length - 1].position.z;
          break;
        case 'forward':
          newSnakeCube.position.x = this.snake[this.snake.length - 1].position.x;
          newSnakeCube.position.y = this.snake[this.snake.length - 1].position.y;
          newSnakeCube.position.z = this.snake[this.snake.length - 1].position.z + 0.5;
          break;
        case 'up':
          newSnakeCube.position.x = this.snake[this.snake.length - 1].position.x;
          newSnakeCube.position.y = this.snake[this.snake.length - 1].position.y - 0.5;
          newSnakeCube.position.z = this.snake[this.snake.length - 1].position.z;
          break;
        case 'down':
          newSnakeCube.position.x = this.snake[this.snake.length - 1].position.x;
          newSnakeCube.position.y = this.snake[this.snake.length - 1].position.y + 0.5;
          newSnakeCube.position.z = this.snake[this.snake.length - 1].position.z;
          break;
      }
      this.snake.push(newSnakeCube);
      this.scene.add(newSnakeCube);
    },
    isCollisionBetween(mesh, mesh2) {
      let firstBB = new Three.Box3().setFromObject(mesh);
      let secondBB = new Three.Box3().setFromObject(mesh2);

      let collision = firstBB.intersectsBox(secondBB);
      return collision;
    },
    adjustScore(){
      this.score++;
    },
    move(cube){
      switch(cube.direction){
        case 'left':
          cube.position.x -= this.speed;
          if(cube.coordinates !== undefined){
             if(cube.position.x <= cube.coordinates.x){
               this.positionFollowingCube(cube);
               this.sendParametersBehind(cube);
               cube.direction = cube.newDirection;
               cube.coordinates = undefined;
             }
          }
          break;
        case 'backward':
          cube.position.z += this.speed; 
          if(cube.coordinates !== undefined){
             if(cube.position.z >= cube.coordinates.z){
               this.positionFollowingCube(cube);
               this.sendParametersBehind(cube);
               cube.direction = cube.newDirection;
               cube.coordinates = undefined;
             }
          }
          break;
        case 'right':
          cube.position.x += this.speed;
          if(cube.coordinates !== undefined){
             if(cube.position.x >= cube.coordinates.x){
               this.positionFollowingCube(cube);
               this.sendParametersBehind(cube);
               cube.direction = cube.newDirection;
               cube.coordinates = undefined;
             }
          }
          break;
        case 'forward': 
          cube.position.z -= this.speed;
          if(cube.coordinates !== undefined){
             if(cube.position.z <= cube.coordinates.z){
               this.positionFollowingCube(cube);
               this.sendParametersBehind(cube);
               cube.direction = cube.newDirection;
               cube.coordinates = undefined;
             }
          }
          break;
        case 'up': 
          cube.position.y += this.speed;
          if(cube.coordinates !== undefined){
             if(cube.position.y >= cube.coordinates.y){
               this.positionFollowingCube(cube);
               this.sendParametersBehind(cube);
               cube.direction = cube.newDirection;
               cube.coordinates = undefined;
             }
          }
          break;
        case 'down':
          cube.position.y -= this.speed;
          if(cube.coordinates !== undefined){
             if(cube.position.y <= cube.coordinates.y){
               this.positionFollowingCube(cube);
               this.sendParametersBehind(cube);
               cube.direction = cube.newDirection;
               cube.coordinates = undefined;
             }
          }
          break;
      }
    },
    sendParametersBehind(cube){
      if(this.snake[cube.name + 1] !== undefined){
        this.snake[cube.name + 1].coordinates = cube.position;
        this.snake[cube.name + 1].direction = cube.direction;
        this.snake[cube.name + 1].newDirection = cube.newDirection;
      }
    },
    positionFollowingCube(cube){
      switch(this.snake[cube.name-1].direction){
        case 'left':
          cube.position.x = this.snake[cube.name-1].position.x + 0.5;
          cube.position.y = this.snake[cube.name-1].position.y;
          cube.position.z = this.snake[cube.name-1].position.z;
          break;
        case 'backward':
          cube.position.x = this.snake[cube.name-1].position.x;
          cube.position.y = this.snake[cube.name-1].position.y;
          cube.position.z = this.snake[cube.name-1].position.z - 0.5; 
          break;
        case 'right':
          cube.position.x = this.snake[cube.name-1].position.x - 0.5;
          cube.position.y = this.snake[cube.name-1].position.y;
          cube.position.z = this.snake[cube.name-1].position.z;
          break;
        case 'forward':
          cube.position.x = this.snake[cube.name-1].position.x;
          cube.position.y = this.snake[cube.name-1].position.y;
          cube.position.z = this.snake[cube.name-1].position.z + 0.5;
          break;
        case 'up':
          cube.position.x = this.snake[cube.name-1].position.x;
          cube.position.y = this.snake[cube.name-1].position.y - 0.5;
          cube.position.z = this.snake[cube.name-1].position.z;
          break;
        case 'down':
          cube.position.x = this.snake[cube.name-1].position.x;
          cube.position.y = this.snake[cube.name-1].position.y + 0.5;
          cube.position.z = this.snake[cube.name-1].position.z;
          break;
      }
    },
    changeCameraAngle(){
      switch(this.snakeHead.direction){
        case 'left':
          let newCameraPositionX = this.snakeHead.position.x+2;
          let newCameraPositionZ = this.snakeHead.position.z; 
          
          this.changeCameraAngleRenderer(newCameraPositionX, newCameraPositionZ);
        break;

        case 'backward': 
          newCameraPositionX = this.snakeHead.position.x;
          newCameraPositionZ = this.snakeHead.position.z-2;

          this.changeCameraAngleRenderer(newCameraPositionX, newCameraPositionZ);
        break;

        case 'right': 
          newCameraPositionX = this.snakeHead.position.x-2;
          newCameraPositionZ = this.snakeHead.position.z;

          this.changeCameraAngleRenderer(newCameraPositionX, newCameraPositionZ);
        break;

        case 'forward': 
          newCameraPositionX = this.snakeHead.position.x;
          newCameraPositionZ = this.snakeHead.position.z+2;

          this.changeCameraAngleRenderer(newCameraPositionX, newCameraPositionZ);
        break;
      }
    },
    changeCameraAngleRenderer(newCameraPositionX, newCameraPositionZ){
      if(newCameraPositionX <= this.camera.position.x){
          this.camera.position.x -= 0.07;
          if(this.camera.position.x < newCameraPositionX){
            this.camera.position.x = newCameraPositionX;
          }
      }else if(newCameraPositionX >= this.camera.position.x){
          this.camera.position.x += 0.07;
          if(this.camera.position.x > newCameraPositionX){
            this.camera.position.x = newCameraPositionX;
          }
      }

      if(newCameraPositionZ <= this.camera.position.z){
          this.camera.position.z -= 0.07;
          if(this.camera.position.z < newCameraPositionZ){
            this.camera.position.z = newCameraPositionZ;
          }
      }else if(newCameraPositionZ >= this.camera.position.z){
          this.camera.position.z += 0.07;
          if(this.camera.position.z > newCameraPositionZ){
            this.camera.position.z = newCameraPositionZ;
          }
      }
    },
    activateGameBorders(){
      if(
        this.snakeHead.position.x>5 || this.snakeHead.position.x<-5 ||
        this.snakeHead.position.y>5 || this.snakeHead.position.y<-5 || 
        this.snakeHead.position.z>5 || this.snakeHead.position.z<-5
        ){
          {this.restartGame()}
        } 
    },
    setControls(event){
      switch(event.keyCode){
        //arrow left 37
        case 65:
          if(this.snakeHead.direction === 'up') this.snakeHead.direction = this.oldDirection;
          if(this.snakeHead.direction === 'down') this.snakeHead.direction = this.oldDirection;
          if(this.snakeHead.direction === 'right') {this.snakeHead.direction = 'forward'; break;}
          if(this.snakeHead.direction === 'forward') {this.snakeHead.direction = 'left'; break;}
          if(this.snakeHead.direction === 'backward') {this.snakeHead.direction = 'right'; break;}
          if(this.snakeHead.direction === 'left') {this.snakeHead.direction = 'backward'; break;}
          break;
        //arrow right 39
        case 68:
          if(this.snakeHead.direction === 'up') this.snakeHead.direction = this.oldDirection;
          if(this.snakeHead.direction === 'down') this.snakeHead.direction = this.oldDirection;
          if(this.snakeHead.direction === 'right') {this.snakeHead.direction = 'backward'; break;}
          if(this.snakeHead.direction === 'forward') {this.snakeHead.direction = 'right'; break;}
          if(this.snakeHead.direction === 'backward') {this.snakeHead.direction = 'left'; break;}
          if(this.snakeHead.direction === 'left') {this.snakeHead.direction = 'forward'; break;}
          break;
        //arrow up 38
        case 87:
          if(this.snakeHead.direction === 'down') {
            this.snakeHead.direction = this.oldDirection;
            break;
          }
          if(this.snakeHead.direction === 'up') {
            this.snakeHead.direction = this.oldDirection;
            break;
          }else if(this.snakeHead.direction !== 'up'){
            this.oldDirection = this.snakeHead.direction;
            this.snakeHead.direction = 'up';
            break;
          };
        //arrow down 40
        case 83:
          if(this.snakeHead.direction === 'down' || this.snakeHead.direction === 'up') {
            if(this.oldDirection === 'forward') {this.snakeHead.direction = 'backward'; break;}
            if(this.oldDirection === 'backward') {this.snakeHead.direction = 'forward'; break;}
            if(this.oldDirection === 'right') {this.snakeHead.direction = 'left'; break;}
            if(this.oldDirection === 'left') {this.snakeHead.direction = 'right'; break;}
            break;
          }
          if(this.snakeHead.direction !== 'down'){
            this.oldDirection = this.snakeHead.direction;
            this.snakeHead.direction = 'down';
            break;
          };
        case 27:
          if(this.paused){
            this.resumeGame();
            break;
          }else{
            this.paused = true; 
            break;
          }
        // case 32:
        //   this.isSpeedingUp = true;
      }
      if (this.snake.length > 1) this.sendCoordinates();
    },
    slowDown (e) {
      this.isSpeedingUp = false;
    },
    setIsSpeedingUpTo (newVal, e) {
      if (e.which === 1) this.isSpeedingUp = newVal;
    },
    resumeGame () {
      this.paused = false; 
      this.animate(); 
    },
    newGame () {
      this.restartGame();
      this.resumeGame();
    },
    async saveGame () {
      console.log(JSON.stringify(this.snake));
      // const res = await axios.post(this.baseUrl + '/savegame', {
      //   snake: JSON.stringify(this.snake),
      //   food: JSON.stringify(this.food),
      //   score: this.score
      // })

      // if (res.data.result === 'ok') {
      //   alert('Game saved successfully');
      // }
    },
    async loadGame () {
      const res = await axios.get(this.baseUrl + '/loadgame')
      this.snake = JSON.parse(res.data.saved_games[0].sav_snake)
      this.food = JSON.parse(res.data.saved_games[0].sav_food)
      this.score = res.data.saved_games[0].sav_score;
      console.log(this.snake);
      throw new Error("Something went badly wrong!");
    },
    sendCoordinates(){
      this.snake[1].coordinates = {
        x: this.snakeHead.position.x, 
        y: this.snakeHead.position.y, 
        z: this.snakeHead.position.z
      };
      this.snake[1].newDirection = this.snakeHead.direction;
    },
    setFocus() {
      this.$refs.game.focus();
    },
  },
  mounted() {
    this.init();
    this.animate();
    this.setFocus();

    window.oncontextmenu = () => {
      this.selectedWeapon.activate();
      return false;
    }
  }
}
</script>

<style scoped lang="scss">
#container{
  width: 100%;
  height: 100vh;
  overflow: hidden;
  position: relative;
}
.score{
  position: absolute; 
  left: 0; 
  right: 0; 
  margin: 0 auto;
  /* width: 500px; */
  font-size: 3em;
  user-select: none;
}
.weapons {
  width: 20%;
  display: flex;
  position: absolute; 
  top: 30%; 
  right: 10%; 
  margin: auto 0;
  font-size: 3em;
  user-select: none;

  .selected-weapon {
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 80%;
    font-size: 0.5em;
    img {
      width: 88px;
      height: 88px;
    }
  }
  .all-weapons {
    display: flex;
    flex-direction: column;
    justify-content: center;
    width: 20%;

    img {
      width: 44px;
      height: 44px;
    }
  }
}
</style>
