<template>
  <div>
    <!-- Photo preview -->
    <div v-if="showPhotoPreview" class="absolute top-0 z-30">
      <img class="fixed top-0 right-0 w-16 h-16 p-3" src="~/assets/img/close-white-18dp.svg" alt="close" @click="onClickClosePhotoPreview">
      <img :src="photo" alt="photo">
    </div>

    <!-- Video preview -->
    <div v-if="showVideoPreview" class="absolute top-0 z-30" >
    <img class="fixed top-0 right-0 z-40 w-16 h-16 p-3" src="~/assets/img/close-white-18dp.svg" alt="close" @click="onClickCloseVideoPreview">
    <video :src="video" 
      autoplay 
      loop
    >
    </video>        
    </div>

    <canvas class="deepar" id="deepar-canvas" ref="canvas" oncontextmenu="event.preventDefault()"></canvas>
    <div class="fixed bottom-0 z-20 w-full py-3 mb-10 slick-background">
      <VueSlickCarousel @afterChange="onElementChange" v-bind="settings">
        <effect-icon-button
            v-for="effect in effectList"
            :key="effect.id"
            :icon="effect.name"
            @click="capturePhoto(effect.name)"
            @long-press-start="onLongPressStart(effect.name)"
            @long-press-stop="onLongPressStop(effect.name)"
        />
      </VueSlickCarousel>
    </div>
  </div>
</template>

<script>
import VueSlickCarousel from "vue-slick-carousel";
import "vue-slick-carousel/dist/vue-slick-carousel.css";
import EffectIconButton from '../components/EffectIconButton'


export default {
  components: { VueSlickCarousel, EffectIconButton },
  data() {
    return {
      canvasHeight: window.innerHeight,
      canvasWidth: window.innerWidth,
      deepARInstance: null,
      settings: {
        arrows: false,
        centerMode: true,
        centerPadding: '15%',
        focusOnSelect: true,
        infinite: true,
        slidesToShow: 3,
        swipeToSlide: true,
        speed: 150,
      },
      photoTaken: null,
      showVideoPreview: false,
      showPhotoPreview: false,
      mediaRecorder: null,
      longPressActive: false,
      effectList:[
          {id:0, name:'lion'},
          {id:1, name:'aviators'},
          {id:2, name:'beard'},
          {id:3, name:'dalmatian'},
          {id:4, name:'flowers'},
          {id:5, name:'koala'},
          {id:6, name:'background_segmentation'},
          {id:7, name:'teddycigar'}
        ]
    };
  },
  computed:{
    photo(){
      return this.$store.state.photo
    },
    video(){
      return this.$store.state.video
    }
  },
  methods: {
    onLongPressStart(key){
      if(key !== this.$store.state.activeEffectIcon){
        console.log('long press espaced');
        return
      }
      //Apply start record CSS to button
      console.log('onLongPressStart',key);
      this.longPressActive = true
      this.initRecord()
    },
    async onLongPressStop(key){
      if(key !== this.$store.state.activeEffectIcon || !this.longPressActive){
        console.log('long press espaced');
        return
      }
      console.log('onLongPressStop',key);
      this.longPressActive = false;
      if(this.mediaRecorder && this.mediaRecorder.state !== 'inactive')
      await this.mediaRecorder.stop()
    },
    initRecord(){
      console.log("initRecord..");
      var video = document.querySelector("video")
      console.log('video ref', video);
      var canvas = document.getElementById("deepar-canvas")
      var videoStream = canvas.captureStream(30);
      this.mediaRecorder = new MediaRecorder(videoStream);
      console.log('mediaRecorder', this.mediaRecorder);

      var chunks = [];
      this.mediaRecorder.ondataavailable = function(e) {
        chunks.push(e.data);
      };

      this.mediaRecorder.onstop = (e) => {
        console.log("video onStop");
        var blob = new Blob(chunks, { 'type' : 'video/mp4' }); // other types are available such as 'video/webm' for instance, see the doc for more info
        chunks = [];
        var videoURL = URL.createObjectURL(blob);
        this.$store.commit('setVideo', videoURL)
        this.showVideoPreview = true
      };
      this.mediaRecorder.start();
    },
    onClickClosePhotoPreview(){
      console.log("close photo");
      this.showPhotoPreview = false
      this.$store.commit('setPhoto', null)
    },
    onClickCloseVideoPreview(){
      console.log("close video");
      this.showVideoPreview = false;
      //  TODO: Temp. work around
      this.showPhotoPreview = false;
      this.$store.commit('setVideo', null)
    },
    capturePhoto(key){
      if(this.$store.state.video){
        return
      }
      console.log(`capturePhoto: ${key} icon is pressed`);
      //  If center icon is clicked
      if(key === this.$store.state.activeEffectIcon && this.longPressActive === false){
        console.log(`capturePhoto: ${key} is at center, take screen shot`);
        this.deepARInstance.takeScreenshot();
      }
    },
    onElementChange(index){
      console.log("onElementChange: change to ", index);
      this.changeEffect(index)
    },
    initialize() {
      // desktop, the width of the canvas is 0.66 * window height and on mobile it's fullscreen
      if (window.innerWidth > window.innerHeight) {
        this.canvasWidth = Math.floor(window.innerHeight * 0.66);
      }

      const deepAR = DeepAR({
        canvasWidth: this.canvasWidth,
        canvasHeight: this.canvasHeight,
        licenseKey:
          "b31c00ca5e518316fd9c2edc7a4ed4507f272498998f7c5ff91a42ddf3afe0bce83fb15abd1d8a43",
        canvas: document.getElementById("deepar-canvas"),
        numberOfFaces: 1,
        libPath: "/lib",
        segmentationInfoZip: "segmentation.zip",
        onInitialize: function () {
          console.log("onInitialize callback", deepAR);
          // start video immediately after the initalization, mirror = true
          deepAR.startVideo(true);

          // or we can setup the video element externally and call deepAR.setVideoElement (see startExternalVideo function below)

          deepAR.switchEffect(0, "slot", "/effects/lion", function () {
            // effect loaded
          });
        },
      });

      deepAR.onCameraPermissionAsked = function () {
        console.log("camera permission asked");
      };

      deepAR.onCameraPermissionGranted = function () {
        console.log("camera permission granted");
      };

      deepAR.onCameraPermissionDenied = function () {
        console.log("camera permission denied");
      };

      deepAR.onScreenshotTaken = (photo) => {
        console.log("screenshot taken");
        this.$store.commit('setPhoto', photo)
        this.showPhotoPreview = true
      };

      deepAR.onImageVisibilityChanged = function (visible) {
        console.log("image visible", visible);
      };

      deepAR.onFaceVisibilityChanged = function (visible) {
        console.log("face visible", visible);
      };

      deepAR.onVideoStarted = function () {
        // var loaderWrapper = document.getElementById("loader-wrapper");
        // loaderWrapper.style.display = "none";
      };

      deepAR.downloadFaceTrackingModel("/lib/models-68-extreme.bin");

      this.deepARInstance = deepAR;
    },
    changeEffect(key) {
      console.log(`${key} selected`);
      switch (key) {
        case 0:
          this.$store.commit('setActiveEffectIcon','lion')
          this.deepARInstance.switchEffect(
            0,
            "slot",
            "/effects/lion",
            function () {
              console.log("changeEffect: changed effect to lion");
            }
          );
          break;
        case 1:
          this.$store.commit('setActiveEffectIcon','aviators')
          this.deepARInstance.switchEffect(
            0,
            "slot",
            "/effects/aviators",
            function () {
              console.log("changeEffect: changed effect to aviators");
            }
          );
          break;
        case 2:
          this.$store.commit('setActiveEffectIcon','beard')
          this.deepARInstance.switchEffect(
            0,
            "slot",
            "/effects/beard",
            function () {
              console.log("changeEffect: changed effect to beard");
            }
          );
          break;
        case 3:
          this.$store.commit('setActiveEffectIcon','dalmatian')
          this.deepARInstance.switchEffect(
            0,
            "slot",
            "/effects/dalmatian",
            function () {
              console.log("changeEffect: changed effect to dalmation");
            }
          );
          break;
        case 4:
          this.$store.commit('setActiveEffectIcon','flowers')
          this.deepARInstance.switchEffect(
            0,
            "slot",
            "/effects/flowers",
            function () {
              console.log("changeEffect: changed effect to flowers");
            }
          );
          break;
        case 5:
          this.$store.commit('setActiveEffectIcon','koala')
          this.deepARInstance.switchEffect(
            0,
            "slot",
            "/effects/koala",
            function () {
              console.log("changeEffect: changed effect to koala");
            }
          );
          break;
        case 6:
          this.$store.commit('setActiveEffectIcon','background_segmentation')
          this.deepARInstance.switchEffect(
            0,
            "slot",
            "/effects/background_segmentation",
            function () {
              console.log("changeEffect: changed effect to background_segmentation");
            }
          );
          break;
        case 7:
          this.$store.commit('setActiveEffectIcon','teddycigar')
          this.deepARInstance.switchEffect(
            0,
            "slot",
            "/effects/teddycigar",
            function () {
              console.log("changeEffect: changed effect to teddycigar");
            }
          );
          break;
        default:
          // code block
          console.log("effect not found");
      }
    },
  },
  mounted() {
    this.initialize();
  },
};
</script>

<style>
canvas.deepar {
  border: 0px none;
  background-color: black;
  display: block;
  margin: auto;
  -webkit-tap-highlight-color: rgba(0, 0, 0, 0);
  z-index: 10;
}

.thumb {
  display: flex;
  justify-content: center;
  align-items: center;
}

.slick-background {
  background-color: transparent
}

.center-circle {

}

.longpress-prevent-default {
  -webkit-user-select: none; /* disable selection/Copy of UIWebView */
  -webkit-touch-callout: none; /* disable the IOS popup when long-press on a link */
}   

.longPressEffect {
    @apply transform scale-150
}

</style>
