
<template>
    <div id="container" @keyup="setControls" tabindex="0" ref = 'game'>
      <div class="score">Your Score: {{score}}</div>
    </div>
</template>

<script>
import * as Three from 'three';

export default {
  data() {
    return {
      camera: null,
      scene: null,
      renderer: null,
      snake: [],
      mesh: null,
      mesh2: null,
      mesh3: null,
      speed: 0.04,
      score: 0,
      direction: 'forwards',
      oldDirection: null,
      paused: false
    }
  },
  methods: {
    init() {
        let container = document.getElementById('container');

        this.scene = new Three.Scene();

        let geometry = new Three.BoxGeometry(0.5, 0.5, 0.5);
        let material = new Three.MeshLambertMaterial ();

        this.mesh = new Three.Mesh(geometry, material);
        this.snake.push(this.mesh);
        this.snake[0].direction = 'forward';
        this.snake[0].move = (lastCube) => {return this.move(lastCube)};
        this.snake[0].name = 0;
        this.generateRandomPositionFor(this.snake[0]);
        this.scene.add(this.snake[0]);

        this.mesh2 = new Three.Mesh(geometry, material);
        this.generateRandomPositionFor(this.mesh2);
        this.scene.add(this.mesh2);

        let light = new Three.PointLight( 0xFFFFFF, 1, 100 );
        light.position.set( 0, 5, 0 );
        this.scene.add( light );

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
    },
    animate() {
        if(!this.paused){
          requestAnimationFrame(this.animate);
        }

        for (let i = 0; i < this.snake.length; i++) {
          this.snake[i].move(this.snake[i]);
        }

        for (let i = 1; i < this.snake.length; i++) {
          if(this.twoProvidedObjectsCollide(this.snake[0], this.snake[i]) && i>1){
            this.restartGame();
          };
        }

        this.changeCameraAngle();

        if(this.twoProvidedObjectsCollide(this.snake[0], this.mesh2)){
          this.adjustScore();
          this.generateRandomPositionFor(this.mesh2);
          this.growSnakeBehind(this.snake[this.snake.length - 1]);
        };
        
        this.activateGameBorders();
          
        this.camera.lookAt(this.snake[0].position.x, this.snake[0].position.y, this.snake[0].position.z);
        this.camera.position.y = this.snake[0].position.y + 1.5;

        this.renderer.render(this.scene, this.camera);
    },
    generateRandomPositionFor(object){
      object.position.x = Math.random()*10 -5;
      object.position.z = Math.random()*10 -5;
      object.position.y = Math.random()*10 -5;
    },
    restartGame(){
      this.generateRandomPositionFor(this.snake[0]);
      for (let i = 1; i < this.snake.length; i++) {
        this.scene.remove(this.snake[i]);
      }
      this.snake = this.snake.slice(0, 1);
      this.score = 0; 
    },
    growSnakeBehind(lastCube){
      let geometry = new Three.BoxGeometry(0.5, 0.5, 0.5);
      let material = new Three.MeshLambertMaterial ();

      let cube = new Three.Mesh(geometry, material);

      cube.direction = lastCube.direction;
      cube.move = (lastSnakeCube) => {return this.move(lastSnakeCube)};
      cube.name = this.snake.length;

      switch(lastCube.direction){
        case 'left':
          cube.position.x = this.snake[this.snake.length - 1].position.x + 0.5;
          cube.position.y = this.snake[this.snake.length - 1].position.y;
          cube.position.z = this.snake[this.snake.length - 1].position.z;
          break;
        case 'backward':
          cube.position.x = this.snake[this.snake.length - 1].position.x;
          cube.position.y = this.snake[this.snake.length - 1].position.y;
          cube.position.z = this.snake[this.snake.length - 1].position.z - 0.5; 
          break;
        case 'right':
          cube.position.x = this.snake[this.snake.length - 1].position.x - 0.5;
          cube.position.y = this.snake[this.snake.length - 1].position.y;
          cube.position.z = this.snake[this.snake.length - 1].position.z;
          break;
        case 'forward':
          cube.position.x = this.snake[this.snake.length - 1].position.x;
          cube.position.y = this.snake[this.snake.length - 1].position.y;
          cube.position.z = this.snake[this.snake.length - 1].position.z + 0.5;
          break;
        case 'up':
          cube.position.x = this.snake[this.snake.length - 1].position.x;
          cube.position.y = this.snake[this.snake.length - 1].position.y - 0.5;
          cube.position.z = this.snake[this.snake.length - 1].position.z;
          break;
        case 'down':
          cube.position.x = this.snake[this.snake.length - 1].position.x;
          cube.position.y = this.snake[this.snake.length - 1].position.y + 0.5;
          cube.position.z = this.snake[this.snake.length - 1].position.z;
          break;
      }
      this.snake.push(cube);
      this.scene.add(cube);
    },
    twoProvidedObjectsCollide(mesh, mesh2) {
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
      switch(this.snake[0].direction){
        case 'left':
          let newCameraPositionX = this.snake[0].position.x+2;
          let newCameraPositionZ = this.snake[0].position.z; 
          
          this.changeCameraAngleRenderer(newCameraPositionX, newCameraPositionZ);
        break;

        case 'backward': 
          newCameraPositionX = this.snake[0].position.x;
          newCameraPositionZ = this.snake[0].position.z-2;

          this.changeCameraAngleRenderer(newCameraPositionX, newCameraPositionZ);
        break;

        case 'right': 
          newCameraPositionX = this.snake[0].position.x-2;
          newCameraPositionZ = this.snake[0].position.z;

          this.changeCameraAngleRenderer(newCameraPositionX, newCameraPositionZ);
        break;

        case 'forward': 
          newCameraPositionX = this.snake[0].position.x;
          newCameraPositionZ = this.snake[0].position.z+2;

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
        this.snake[0].position.x>5 || this.snake[0].position.x<-5 ||
        this.snake[0].position.y>5 || this.snake[0].position.y<-5 || 
        this.snake[0].position.z>5 || this.snake[0].position.z<-5
        ){
          {this.restartGame()}
        } 
    },
    setControls(event){
      switch(event.keyCode){
        case 37:
          if(this.snake[0].direction === 'up') this.snake[0].direction = this.oldDirection;
          if(this.snake[0].direction === 'down') this.snake[0].direction = this.oldDirection;
          if(this.snake[0].direction === 'right') {this.snake[0].direction = 'forward'; break;}
          if(this.snake[0].direction === 'forward') {this.snake[0].direction = 'left'; break;}
          if(this.snake[0].direction === 'backward') {this.snake[0].direction = 'right'; break;}
          if(this.snake[0].direction === 'left') {this.snake[0].direction = 'backward'; break;}
          break;
        case 39:
          if(this.snake[0].direction === 'up') this.snake[0].direction = this.oldDirection;
          if(this.snake[0].direction === 'down') this.snake[0].direction = this.oldDirection;
          if(this.snake[0].direction === 'right') {this.snake[0].direction = 'backward'; break;}
          if(this.snake[0].direction === 'forward') {this.snake[0].direction = 'right'; break;}
          if(this.snake[0].direction === 'backward') {this.snake[0].direction = 'left'; break;}
          if(this.snake[0].direction === 'left') {this.snake[0].direction = 'forward'; break;}
          break;
        case 38:
          if(this.snake[0].direction === 'down') {
            this.snake[0].direction = this.oldDirection;
            break;
          }
          if(this.snake[0].direction === 'up') {
            this.snake[0].direction = this.oldDirection;
            break;
          }else if(this.snake[0].direction !== 'up'){
            this.oldDirection = this.snake[0].direction;
            this.snake[0].direction = 'up';
            break;
          };
        case 40:
          if(this.snake[0].direction === 'down') {
            if(this.oldDirection === 'forward') {this.snake[0].direction = 'backward'; break;}
            if(this.oldDirection === 'backward') {this.snake[0].direction = 'forward'; break;}
            if(this.oldDirection === 'right') {this.snake[0].direction = 'left'; break;}
            if(this.oldDirection === 'left') {this.snake[0].direction = 'right'; break;}
            break;
          }
          if(this.snake[0].direction === 'up'){
            if(this.oldDirection === 'forward') {this.snake[0].direction = 'backward'; break;}
            if(this.oldDirection === 'backward') {this.snake[0].direction = 'forward'; break;}
            if(this.oldDirection === 'right') {this.snake[0].direction = 'left'; break;}
            if(this.oldDirection === 'left') {this.snake[0].direction = 'right'; break;}
            break;
          }
          if(this.snake[0].direction !== 'down'){
            this.oldDirection = this.snake[0].direction;
            this.snake[0].direction = 'down';
            break;
          };
        case 27:
          if(this.paused){
            this.paused = false; 
            this.animate(); 
            break;
          }else{
            this.paused = true; break;
          }
      }
      if(this.snake.length>1){this.sendCoordinates();}
    },
    sendCoordinates(){
      this.snake[1].coordinates = {
        x: this.snake[0].position.x, 
        y: this.snake[0].position.y, 
        z: this.snake[0].position.z
      };
      this.snake[1].newDirection =  this.snake[0].direction;
    },
    setFocus() {
      this.$refs.game.focus();
    },
  },
  mounted() {
      this.init();
      this.animate();
      this.setFocus();
  }
}
</script>

<style scoped>
    #container{
      width: 100%;
      height: 93vh;
    }
    .score{
      font-size: 3em;
    }
</style>
