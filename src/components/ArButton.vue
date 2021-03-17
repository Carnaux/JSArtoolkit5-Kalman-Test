<template>
    <div id="app">
        <button id="enableButton" @click="enableKalman">Enable Kalman</button>
        <div :style="{display: startedAr ? 'block' : 'none'}">
          <video
              loop
              autoplay
              muted
              playsinline
              id="video">
          </video>
          <canvas id="canvas"></canvas>
        </div>
    </div>
</template>

<script>
import * as THREE from 'three';
import {GLTFLoader} from 'three/examples/jsm/loaders/GLTFLoader';
import { KTX2Loader } from 'three/examples/jsm/loaders/KTX2Loader.js';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';
import { MeshoptDecoder } from 'three/examples/jsm/libs/meshopt_decoder.module.js';

import KalmanFilter from 'kalmanjs';

export default {
    name: "ArButton",
    data(){
        return {
            startedAr: false,

            canvas: null,
            video: null,

            interpolationFactor: 24,
            trackedMatrix: {
                // for interpolation
                delta: [
                    0,0,0,0,
                    0,0,0,0,
                    0,0,0,0,
                    0,0,0,0
                ],
                interpolated: [
                    0,0,0,0,
                    0,0,0,0,
                    0,0,0,0,
                    0,0,0,0
                ]
            },
            
            root: new THREE.Object3D(),
            scene: null,
            camera: null,
            renderer: null,

            ambientLight: null,
            spotLightTop: null,
            spotLightLeft: null,
            spotLightRightAngled: null,

            sphere: null,
            cube: null,
            cone: null,

            input_width: null, 
            input_height: null,
            vw: null,
            vh: null,
            sw: null,
            sh: null,
            pscale: null, 
            sscale: null,
            w: null, 
            h: null,
            pw: null, 
            ph: null,
            ox: null, 
            oy: null,
            worker: null,

            world: null,
            index: null,

            lasttime: null,
            time: 0,
            clock: new THREE.Clock(),

            canvas_process: null,
            context_process: null,

            camera_para: '../camera_para.dat',
            worker_path: './js/artoolkit.multi_worker.js',

            kalman: null,
            enable_filter: false,

            modelTest: {
                name: "elephant",
                modelUrl: "./models/Flamingo.glb",
                markerUrl: "../markers/pinball",
                animations: null,
                loadedModel: null,
                loaded: false,
                position: new THREE.Vector3(100,100, 30),
                rotation: new THREE.Vector3(0,0,0),
                scale: new THREE.Vector3(1,1,1),
                mixer: null,
            },
      
            
        };
    },
    methods:{
        async initAR(){
            this.startedAr = !this.startedAr;
            this.createKalman();

            await this.loadGLTF();

            this.root.matrixAutoUpdate = false;

            if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
                var hint = {
                    audio: false,
                    video: true
                };
                if( window.innerWidth < 800 ) {
                    hint = {
                        audio: false,
                        video: {
                            width: { ideal:  1024 },
                            height: { ideal: 540 },
                            facingMode:
                                { exact:
                                    "environment"
                                }
                            },
                    };
                }
                let _this = this;
                navigator.mediaDevices.getUserMedia( hint ).then( function( stream ) {
                    _this.video.srcObject = stream;
                    _this.video.addEventListener( 'loadedmetadata', function() {
                        _this.video.play();

                        _this.input_width = _this.video.videoWidth;
                        _this.input_height = _this.video.videoHeight;

                        _this.start();

                    });
                });
            }
        },
        async start() {

            this.renderer = new THREE.WebGLRenderer({ canvas: this.canvas, alpha: true, antialias: true, precision: 'mediump'});
            this.renderer.setPixelRatio(window.devicePixelRatio);
            this.renderer.outputEncoding = THREE.sRGBEncoding;
            this.renderer.shadowMap.enabled = true;
            this.renderer.shadowMap.type = THREE.PCFSoftShadowMap;

            this.scene = new THREE.Scene();

            this.camera = new THREE.PerspectiveCamera( 45, window.innerWidth / window.innerHeight, 1, 10000 );
            this.camera.matrixAutoUpdate = false;

            this.ambientLight = new THREE.AmbientLight( 0x404040, 2 ); // soft white light
            this.scene.add( this.ambientLight );

            this.spotLight = new THREE.SpotLight(0xffffff, 3, 11, Math.PI * 0.3);
            this.spotLight.castShadow = true;
            this.spotLight.position.set(0, 10, 0);
            this.root.add(this.spotLight);

            this.spotLightLeft = new THREE.SpotLight(0xffffff, 3, 12, Math.PI * 0.3);
            this.spotLightLeft.castShadow = true;
            this.spotLightLeft.position.set(-10, 2, 2);
            this.root.add(this.spotLightLeft);

            this.spotLightRightAngled = new THREE.SpotLight(0xffffff, 3, 21, Math.PI * 0.3);
            this.spotLightRightAngled.castShadow = true;
            this.spotLightRightAngled.position.set(10, 10, 10);
            this.root.add(this.spotLightRightAngled);

            this.scene.add(this.camera);
            
            this.scene.add(this.root);

            this.load();
            this.tick();
            this.process();
        },
        setMatrix(matrix, value) {
            var array = [];
            for (var key in value) {
                array[key] = value[key];
            }
            if (typeof matrix.elements.set === "function") {
                matrix.elements.set(array);
                return matrix;
            } else {
                matrix.elements = [].slice.call(array);
                return matrix;
            }
        },
        createKalman(){
            let filterArr = [];
            for(let i = 0; i < 16; i++){
                let filter = new KalmanFilter({R: 0.0005, Q: 2});
                filterArr.push(filter);
            }
            filterArr[3] = null;
            filterArr[7] = null;
            filterArr[11] = null;
            filterArr[15] = null;

            this.kalman = filterArr;

        },
        applyKalman(matrix){
            let newElArr = matrix.elements.map((num, index)=>{
                if(this.kalman[index] != null){
                    return this.kalman[index].filter(num);
                }else{
                    return num;
                }
            })

            return new THREE.Matrix4().fromArray(newElArr);
        },
        found(msg) {
            if (!msg) {
                this.world = null;
            } else {
                this.world = JSON.parse(msg.matrixGL_RH);
          
                this.index = JSON.parse(msg.index);
            }
        },
        load(){
                this.vw = this.input_width;
                this.vh = this.input_height;

                this.pscale = 320 / Math.max(this.vw, this.vh / 3 * 4);
                this.sscale = this.isMobile() ? window.outerWidth / this.input_width : 1;

                this.sw = this.vw * this.sscale;
                this.sh = this.vh * this.sscale;

                this.w = this.vw * this.pscale;
                this.h = this.vh * this.pscale;
                this.pw = Math.max(this.w, this.h / 3 * 4);
                this.ph = Math.max(this.h, this.w / 4 * 3);
                this.ox = (this.pw - this.w) / 2;
                this.oy = (this.ph - this.h) / 2;
                // this.canvas_process.style.clientWidth = this.pw + "px";
                // this.canvas_process.style.clientHeight = this.ph + "px";
                this.canvas_process.width = this.pw;
                this.canvas_process.height = this.ph;

                this.renderer.setSize(this.sw, this.sh);

                this.worker = new Worker(this.worker_path);

                this.worker.postMessage({ type: "load", pw: this.pw, ph: this.ph, camera_para: this.camera_para, markerUrls: this.modelTest.markerUrl });
                
                let _this = this;
                this.worker.onmessage = function(ev) {
                    var msg = ev.data;
                    switch (msg.type) {
                        case "loaded": {
                            var proj = JSON.parse(msg.proj);
                            var ratioW = _this.pw / _this.w;
                            var ratioH = _this.ph / _this.h;
                            proj[0] *= ratioW;
                            proj[4] *= ratioW;
                            proj[8] *= ratioW;
                            proj[12] *= ratioW;
                            proj[1] *= ratioH;
                            proj[5] *= ratioH;
                            proj[9] *= ratioH;
                            proj[13] *= ratioH;
                            _this.camera.projectionMatrix = _this.setMatrix(_this.camera.projectionMatrix, proj);
                            break;
                        }
                        case "endLoading": {
                            if (msg.end == true) {
                                console.log("FINISHED LOADING!!!")
                                // // removing loader page if present
                                // var loader = document.getElementById('loading');
                                // if (loader) {
                                //     loader.querySelector('.loading-text').innerText = 'Start the tracking!';
                                //     setTimeout(function(){
                                //         loader.parentElement.removeChild(loader);
                                //     }, 2000);
                                // }
                            }
                            break;
                        }
                        case "found": {
                            _this.found(msg);
                            break;
                        }
                        case "not found": {
                            _this.found(null);
                            break;
                        }
                    }
                    _this.process();
                };
        },
        draw() {
            var now = Date.now();
            var dt = now - this.lasttime;
            this.time += dt;
            this.lasttime = now;

            if (!this.world) {
                this.modelTest.loadedModel.visible = false;
            } else {
                this.modelTest.loadedModel.visible = true;
                this.modelTest.animations.forEach( ( clip ) => { 
                    this.modelTest.mixer.clipAction( clip ).play();
                });
                // interpolate matrix
                // for (var i = 0; i < 16; i++) {
                //     this.trackedMatrix.delta[i] = this.world[i] - this.trackedMatrix.interpolated[i];
                //     this.trackedMatrix.interpolated[i] =
                //     this.trackedMatrix.interpolated[i] +
                //     this.trackedMatrix.delta[i] / this.interpolationFactor;
                // }

               
                // set matrix of 'root' by detected 'world' matrix
                // this.root.matrix = this.setMatrix(this.root.matrix, this.trackedMatrix.interpolated);
                this.setMatrix(this.root.matrix, this.world);

                if(this.enable_filter){
                    let filteredMatrix = this.applyKalman(this.root.matrix);
                    this.root.matrix = filteredMatrix;
                }
               
            }

            let delta = this.clock.getDelta();

            this.modelTest.mixer.update( delta )

            this.renderer.render(this.scene, this.camera);
        },
        process() {
            this.context_process.fillStyle = "black";
            this.context_process.fillRect(0, 0, this.pw, this.ph);
            this.context_process.drawImage(this.video, 0, 0, this.vw, this.vh, this.ox, this.oy, this.w, this.h);

            var imageData = this.context_process.getImageData(0, 0, this.pw, this.ph);
            this.worker.postMessage({ type: "process", imagedata: imageData }, [imageData.data.buffer]);
        },
        tick() {
            this.draw(); 
            requestAnimationFrame(this.tick);
        },
        isMobile() {
            return /Android|mobile|iPad|iPhone/i.test(navigator.userAgent);
        },
        async loadGLTF(){
        
            await this.modelLoader(this.modelTest.modelUrl).then(data =>{
                this.modelTest.loadedModel = data.scene;
                this.modelTest.loadedModel.name = this.modelTest.name;
                //transforms
                this.modelTest.loadedModel.position.copy(this.modelTest.position);
                this.modelTest.loadedModel.rotation.copy(this.modelTest.rotation);
                this.modelTest.loadedModel.scale.copy(this.modelTest.scale);

                //animation
                this.modelTest.animations = data.animations;
                this.modelTest.mixer = new THREE.AnimationMixer( data.scene );
                
                this.modelTest.animations.forEach( ( clip ) => { 
                    this.modelTest.mixer.clipAction( clip ).play();
                });

                this.modelTest.loadedModel.traverse((node) => {
                    if (node.isMesh) {
                        // console.log(node)
                        // TODO(https://github.com/mrdoob/three.js/pull/18235): Clean up.
                        // node.material = new THREE.MeshPhongMaterial();
                        // console.log(node.material)
                        // node.material.type = "MeshPhongMaterial";
                        node.material.depthWrite = !node.material.transparent;
                        node.material.side = THREE.DoubleSide;
                        node.material.needsUpdate = true;
                    }
                });

                this.root.add(this.modelTest.loadedModel);
            });
            
            console.log("Finished loading model")
        },
        modelLoader(url) {
            const loader = new GLTFLoader()
            .setDRACOLoader(new DRACOLoader().setDecoderPath( '../wasm/' ))
            .setKTX2Loader(new KTX2Loader()
                .setTranscoderPath( '../wasm/' )
            )
            .setMeshoptDecoder( MeshoptDecoder );

            return new Promise((resolve, reject) => {
                loader.load(url, data=> resolve(data), null, reject);
            });
        },
        enableKalman(){
            this.enable_filter = !this.enable_filter;
        }
    },
    mounted(){
        this.canvas = document.querySelector("#canvas");
        this.video = document.querySelector("#video");

        this.canvas_process = document.createElement('canvas');
        this.context_process = this.canvas_process.getContext('2d'); 

        this.initAR();
    },  
    beforeDestroy(){
        this.video.removeEventListener( 'loadedmetadata', function() {
            this.video.play();

            console.log( 'video',  this.video,  this.video.videoWidth,  this.video.videoHeight );

            this.start();

        });
    }
}
</script>
<style scoped>
    #video {
        position: absolute;
        top: 0;
        left: 0;

        display: block;
        width: 100vw !important;
        height: 100vh !important;
        object-fit: cover;
    }
    #canvas {
        position: absolute;
        left: 0;
        top: 0;
        z-index: 100;

        display: block;
        width: 100vw !important;
        height: 100vh !important;
        object-fit: cover;
    }
    #enableButton{
        position: absolute;
        z-index: 500;
    }
</style>