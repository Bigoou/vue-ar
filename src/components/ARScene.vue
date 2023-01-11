<script>
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { ArToolkitProfile, ArToolkitSource, ArToolkitContext, ArMarkerControls} from 'ar-js-org/three.js/build/ar-threex.js';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import gsap from "gsap";
import {EffectComposer} from "three/examples/jsm/postprocessing/EffectComposer.js"
import {RenderPass} from "three/examples/jsm/postprocessing/RenderPass.js"
import {ShaderPass} from "three/examples/jsm/postprocessing/ShaderPass.js"
import {OutlinePass} from "three/examples/jsm/postprocessing/OutlinePass.js"
import {FXAAShader} from "three/examples/jsm/shaders/FXAAShader.js"

export default {
  name: "ARScene",
  data() {
    return {
      isScanned: false,
      cameraIsLoaded : false,
      mesh : {}
    };
  },
  watch: {
    isScanned(newVal, oldVal) {
      console.log(`variable was updated from ${oldVal} to ${newVal}`);
    },
  },

  mounted() {
    ArToolkitContext.baseURL = './'
    console.log("ARScene mounted");

    // init renderer
    var renderer	= new THREE.WebGLRenderer({
      antialias	: true,
      alpha: true
    });

    renderer.setClearColor(new THREE.Color('lightgrey'), 0)
    // renderer.setPixelRatio( 2 );
    renderer.setSize( window.innerWidth, window.innerHeight );
    renderer.domElement.style.position = 'absolute'
    renderer.domElement.style.top = '0px'
    renderer.domElement.style.left = '0px'
    document.body.appendChild( renderer.domElement ); // We should be able to specify an html element to append AR.js related elements to.

    // array of functions for the rendering loop
    var onRenderFcts= [];

    // init scene and camera
    var scene	= new THREE.Scene();


    //////////////////////////////////////////////////////////////////////////////////
    //  Add orbit control
    //////////////////////////////////////////////////////////////////////////////////

    // var controls = new OrbitControls( camera, renderer.domElement );

    //////////////////////////////////////////////////////////////////////////////////
    //		Initialize a basic camera
    //////////////////////////////////////////////////////////////////////////////////

    // Create a camera
    var camera = new THREE.PerspectiveCamera();
    camera.position.z = 4;
    camera.position.y = 0.25
    scene.add(camera);
    const artoolkitProfile = new ArToolkitProfile();
    artoolkitProfile.sourceWebcam(); // Is there good reason for having a function to set the sourceWebcam but not the displayWidth/Height etc?
    this.cameraIsLoaded = true;
    // add existing parameters, this is not well documented
    let additionalParameters = {
        // Device id of the camera to use (optional)
        deviceId: null,
        // resolution of at which we initialize in the source image
        sourceWidth: window.innerWidth,
        sourceHeight: window.innerHeight,
        // resolution displayed for the source
        displayWidth: window.innerWidth,
        displayHeight: window.innerHeight,
    }
    
    Object.assign(artoolkitProfile.sourceParameters, additionalParameters);
    console.log(artoolkitProfile.sourceParameters); // now includes the additionalParameters

    const arToolkitSource = new ArToolkitSource(artoolkitProfile.sourceParameters);

    arToolkitSource.init(function onReady(){
      onResize();
    })

    console.log(arToolkitSource);

    // handle resize
    window.addEventListener('resize', function(){
      onResize(); 
    })

    // resize is not called for the canvas on init. The canvas with the cube seems to be resized correctly at start.
    // Is that maybe a vue-specific problem?
    function onResize(){
      arToolkitSource.onResizeElement()
      arToolkitSource.copyElementSizeTo(renderer.domElement)
      if(arToolkitContext.arController !== null){
        arToolkitSource.copyElementSizeTo(arToolkitContext.arController.canvas)
      }
    }
      

    ////////////////////////////////////////////////////////////////////////////////
    //          initialize arToolkitContext
    ////////////////////////////////////////////////////////////////////////////////

    // create atToolkitContext
    var arToolkitContext = new ArToolkitContext({
      debug: false,
      cameraParametersUrl: ArToolkitContext.baseURL + 'data/camera_para.dat',
      detectionMode: 'mono',
      canvasWidth: window.innerWidth,
      canvasHeight: window.innerHeight,
      imageSmoothingEnabled : true, // There is still a warning about mozImageSmoothingEnabled when using Firefox
    })
    
    // initialize it
    arToolkitContext.init(function onCompleted(){
      // copy projection matrix to camera
      camera.projectionMatrix.copy( arToolkitContext.getProjectionMatrix() );
    })

    // update artoolkit on every frame
    onRenderFcts.push(function(){
      if( arToolkitSource.ready === false )	return

      arToolkitContext.update( arToolkitSource.domElement )
    })

    ////////////////////////////////////////////////////////////////////////////////
    //          Create a ArMarkerControls
    ////////////////////////////////////////////////////////////////////////////////

    var markerGroup = new THREE.Group()
    scene.add(markerGroup)

    var markerControls = new ArMarkerControls(arToolkitContext, markerGroup, {
      type: 'pattern',
      patternUrl: ArToolkitContext.baseURL + 'data/patt.hiro',
      smooth: true,
      smoothCount: 5,
      smoothTolerance: 0.01,
      smoothThreshold: 2
    })
    var markerPosition = markerControls.object3d.position

    //////////////////////////////////////////////////////////////////////////////////
    //		add an object in the scene
    //////////////////////////////////////////////////////////////////////////////////

    var markerScene = new THREE.Scene()
    markerGroup.add(markerScene)

    // var mesh = new THREE.AxesHelper()
    // markerScene.add(mesh)

    // add a torus knot
    // var geometry	= new THREE.BoxGeometry(1,1,1);
    // var material	= new THREE.MeshNormalMaterial({
    //   transparent : true,
    //   opacity: 0.5,
    //   side: THREE.DoubleSide
    // });
    // var mesh	= new THREE.Mesh( geometry, material );
    // mesh.position.y	= geometry.parameters.height/2
    var mesh;

    var loader = new GLTFLoader();
    loader.load('/models/CanvasMuseum.glb', function (gltf) {
      gltf.scene.children[0].scale.set(0.3, 0.3, 0.3);
      console.log(gltf.scene.children[0]);
      mesh = gltf.scene.children[0];
      console.log(mesh);
      markerScene.add(mesh);
     
    }, undefined, function (error) {
      console.error(error);
    });
    console.log(markerScene);
    // markerScene.add(mesh);
      
    // Raycaster

    const mouse = new THREE.Vector2();

    const raycaster = new THREE.Raycaster();
    let currentIntersect = null

    window.addEventListener('click', () => {
        // if (currentIntersect) {
            // console.log(scene.children[1]);
            console.log(arToolkitContext);

            mesh = markerScene.children[0]
            const position = scene.children[1].position
            const rotation = scene.children[1].rotation

            console.log(position);
            console.log(rotation);
            
            // Set position
            // mesh.scale.set(0.5, 0.5, 0.5)
            markerGroup.remove(scene.children[1])
            mesh.position.set(position.x, position.y, position.z)
            mesh.rotation.set(rotation._x, rotation._y, rotation._z)
            scene.add(mesh)
            // scene.children[3].position = position
            // scene.children[3].rotation.set(rotation._x, rotation._y, rotation._z)
            console.log(scene)

            // Animate with GSAP Timeline

            this.isScanned = true;

            const tl = gsap.timeline();
            tl.to(mesh.position, {
              duration: 
              1, 
              y: 0.5, 
              x: 0,
              ease: 'power2.out'
            })
            .to(mesh.rotation, {
              duration: 
              1, 
              y: 0, 
              x: 0,
              z: 0,
              ease: 'power2.out',
              delay : -1
            })
            .to(document.getElementById('arjs-video'), {
              duration: 
              1, 
              opacity: 0,
              ease: 'power2.out',
              delay : 0
            })
            .to(document.getElementsByClassName('ARFooter'), {
              duration: 
              1, 
              opacity: 0,
              ease: 'power2.out',
              delay : -1
            })
            .fromTo(document.getElementsByClassName('painting-name'), {
              opacity: 0
            }, {
              duration: 
              1, 
              opacity: 1,
              ease: 'power2.out',
              delay : -1
            })
            // .to(document.getElementsByClassName('scenario'), {
            //   duration: 
            //   1, 
            //   opacity: 1,
            //   ease: 'power2.out',
            //   delay : 0
            // })
            console.log(mesh)
        // }
    })    

    // const composer = new EffectComposer(renderer)
    // const renderPass = new RenderPass(scene, camera)
    // composer.addPass(renderPass)

    // let outlinePass = new OutlinePass(new THREE.Vector2(window.innerWidth, window.innerHeight), scene, camera)
    // outlinePass.edgeStrength = 6
    // outlinePass.edgeGlow = 1
    // outlinePass.edgeThickness = 1
    // console.log(outlinePass);
    // composer.addPass(outlinePass)

    // const fxaaPass = new ShaderPass(FXAAShader)
    // fxaaPass.material.uniforms["resolution"].value.set(1 / window.innerWidth, 1 / window.innerHeight)
    // composer.addPass(fxaaPass)

    //////////////////////////////////////////////////////////////////////////////////
    //		render the whole thing on the page
    //////////////////////////////////////////////////////////////////////////////////

    onRenderFcts.push(function(){
      renderer.render( scene, camera );
    })

    console.log(markerScene.children);

         // On click the cube, the marker is removed and the cube is added to the scene
        
      const intervalId = setInterval(() => {
      if (this.isScanned) {
    //     document.getElementById('arjs-video').classList.add('hidden')
    //     loader.load('/models/wooden-crate.glb', function (gltf) {
    //       mesh = gltf.scene.children[2];
    //       mesh.position.y = 0.5;
    //       mesh.scale.set(0.5, 0.5, 0.5);
    //       scene.add(mesh);
    //       onRenderFcts.push(function(delta){
    //       mesh.rotation.y += delta * Math.PI / 8
    // })
    //     }, undefined, function (error) {
    //       console.error(error);
    //     });

        console.log(scene.children)
        clearInterval(intervalId);
      } else {
        console.log('not scanned');
      }
    }, 1000);
    // this.$once('beforeDestroy', () => clearInterval(intervalId));
    // Adding lights
    const ambientLight = new THREE.AmbientLight(0xffffff, 1);
    scene.add(ambientLight);
      // run the rendering loop
    var lastTimeMsec = null;

    requestAnimationFrame(function animate(nowMsec){
      // keep looping
      requestAnimationFrame(animate);
      // measure time
      lastTimeMsec	= lastTimeMsec || nowMsec-1000/60
      var deltaMsec	= Math.min(200, nowMsec - lastTimeMsec)
      lastTimeMsec	= nowMsec

      raycaster.setFromCamera(mouse, camera)

      // const intersects = raycaster.intersectObjects(markerScene.children[0], true);
      const intersects = raycaster.intersectObjects(scene.children, true);

      if(intersects.length)
    {
        if(!currentIntersect)
        {
            console.log('mouse enter')
        }

        currentIntersect = intersects[0]
    }
    else
    {
        if(currentIntersect)
        {
            console.log('mouse leave')
        }
        
        currentIntersect = null
    }
      

      // call each update function
      onRenderFcts.forEach(function(onRenderFct){
        onRenderFct(deltaMsec/1000, nowMsec/1000)
      })
    })
    }
 
}
</script>

<template>
  
  <div id="ARScene">
    
  </div>

  <div v-if="isScanned" class="scenario">
      <div class="scenario-header">
        <svg fill="none" height="34" viewBox="0 0 44 34" width="44" xmlns="http://www.w3.org/2000/svg"><g fill="#b29957"><path d="m16.1444 2.41602c.9524 0 1.966.75437 1.966 2.64385 0 2.40973-1.6227 3.65127-3.2877 3.65127v-.15845c1.7803-.16791 2.5916-1.08074 2.5375-2.9702 0-.02128-.0494-.02128-.0517 0-.1223.60303-.6867.88682-1.3052.88682-1.2158 0-1.6932-1.00269-1.6932-1.99355 0-1.16348.7596-2.05974 1.8343-2.05974z"/><path d="m28.6507 4.78955c-5.4795 0-11.9349 5.09145-11.9349 13.17195 0 7.5461 5.0773 13.1815 11.9372 13.1791 6.8905-.0023 11.9326-5.6353 11.9326-13.1791 0-8.07815-6.4578-13.16722-11.9349-13.17195zm0 26.08385c-2.9632.0023-6.3473-.7425-6.3473-12.9119-.0023-11.88313 3.577-12.9402 6.3473-12.93784 2.7679-.00236 6.3566 1.05471 6.3566 12.93784-.0023 12.1694-3.3582 12.9119-6.3566 12.9119z"/><path d="m16.1423 29.8042h-12.88964v-23.98633h-.09407l-.25869.00237v24.29136h13.2448z"/></g></svg>
        <svg width="24" height="25" viewBox="0 0 24 25" fill="none" xmlns="http://www.w3.org/2000/svg">
          <path d="M11 5.6748L6 9.6748H2V15.6748H6L11 19.6748V5.6748Z" stroke="#B29957" stroke-linecap="round" stroke-linejoin="round"/>
          <path d="M15.54 9.13477C16.4774 10.0724 17.004 11.3439 17.004 12.6698C17.004 13.9956 16.4774 15.2671 15.54 16.2048" stroke="#B29957" stroke-linecap="round" stroke-linejoin="round"/>
          <path d="M19.0699 5.60498C20.9447 7.48026 21.9978 10.0233 21.9978 12.675C21.9978 15.3266 20.9447 17.8697 19.0699 19.745" stroke="#B29957" stroke-linecap="round" stroke-linejoin="round"/>
        </svg>
      </div>
      <div class="title">
        <p>Avant 1914</p>
        <div class="chapter-name">
          <h3>I - Les Arts à Paris </h3>
          <svg width="183" height="28" viewBox="0 0 183 28" fill="none" xmlns="http://www.w3.org/2000/svg">
            <path d="M97.0468 0.850098C-20.4492 13.8132 -41.7777 31.8298 94.3633 26.205C230.504 20.5802 202.912 -5.25214 47.4009 4.97477" stroke="#FFC01E" stroke-width="0.6" stroke-linecap="round" stroke-linejoin="round"/>
          </svg>
        </div>
      </div>
      <div class="painting-name">
        <h1>
          Domenica
        </h1>
      </div>
      <div class="main-text">
        <h2>
          Portrait de Madame Paul Guillaume au grand chapeau
        </h2>
        <p>1924</p>
      </div>
      <div class="ellipse e-1"></div>
      <div class="ellipse e-2"></div>
    </div>
  
  <div v-if="!isScanned" class="ARFooter">
      <p>
        Scannez le QR Code pour valider le checkpoint et découvrir le prochain chapitre !
      </p>
  </div>
</template>


<style scope>

@import url('https://fonts.googleapis.com/css2?family=Kristi&display=swap');

.loader {
  background-color: #ffecd7;
  color : #090705;
  display : flex;
  justify-content : center;
  align-items : center;
}

h2 {
  font-family: Branch;
  font-size : 26px;
  text-align: center;
}

h1 {
  font-size : 6em;
}

  body {
    background-color: #090705;
    color : #ffecd7
  }

  video {
    z-index : 1 !important
  }

  .title {
    text-align : center;
  }

  .chapter-name {
    position : relative;
  }

  .chapter-name svg {
    position : absolute;
    top : 0;
    left : 50%;
    transform : translateX(-50%);
  }

  .chapter-name h3 {
    margin : 0;
  }

  .title {
    position : absolute ;
    top : 10%;
    left : 50%;
    transform : translateX(-50%);
  }

  .title p {
    margin : 0;
    
  }

  .painting-name {
    font-family: 'Kristi';
    color : #FFC01E;
    text-align: center;
    transform : translateY(100%) rotate(-10deg);
    position : relative;
    z-index : 10;
  }

  .scenario-header {
    display : flex;
    justify-content: space-between;
    align-items: center;
    width : 95%;
    position : absolute;
    top : 2%;
    left : 50%;
    transform : translateX(-50%);
  }

  .ellipse {
    width : 424px;
    height : 424px;
    border-radius: 50%;
    background: radial-gradient(50% 50% at 50% 50%, rgba(70, 55, 34, 0.626) 0%, rgba(53, 42, 27, 0.154) 95.92%);
    opacity: 0.8;
    position : absolute;
    z-index : 0;
  }

  .e-1 {
    right : -25%;
    top : -10%;
  }

  .e-2 {
    left : -25%;
    bottom : -10%;
  }

  .ARFooter {
    position: absolute;
    z-index : 2;
    bottom: 2%;
    left : 50%;
    transform : translateX(-50%);
    width: 90%;
    height: 80px;
    background: rgba(77, 77, 77, 0.4);
    border: 1px solid #FFFFFF;
    backdrop-filter: blur(6px);
    border-radius: 8px;    
    color: white;
    text-align: center;
  }

  .main-text {
    width : 80%;
    margin : auto;
    transform : translateY(75%);
    position : relative;
    z-index : 1
  }

  canvas {
    z-index : 3 !important
  }

  .hidden {
    opacity: 0;
  }

</style>