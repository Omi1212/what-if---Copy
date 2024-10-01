<template>
  <v-app>
    <v-container fluid>
      <v-row justify="center">
        <v-col cols="12" class="fullscreen">
          <v-alert v-if="noFrontCamera" type="error">You don't seem to have a front camera on your device</v-alert>
          <v-alert v-if="noRearCamera" type="error">You don't seem to have a rear camera on your device</v-alert>

          <v-card class="camera-container">
            <qrcode-stream
              :paused="paused"
              :torch="torchActive"
              :constraints="{ deviceId: facingMode }"
              @detect="onDetect"
              @error="onError"
              @camera-on="onCameraOn"
            >
              <v-alert v-if="validationSuccess && paused" type="success">This is a valid code</v-alert>
              <v-alert v-if="validationFailure && paused" type="error">This is NOT a valid code!</v-alert>
              <v-alert v-if="validationPending && paused" type="info">Long validation in progress...</v-alert>

              <div class="qr-frame"></div>

              <!-- Botones de íconos en la parte superior -->
              <div class="top-button-container">
                <v-btn @click="switchCamera" icon>
                  <v-icon>mdi-camera-switch</v-icon>
                </v-btn>
                <v-btn @click="toggleTorch" :disabled="torchNotSupported" icon>
                  <v-icon>mdi-flashlight</v-icon>
                </v-btn>
                <v-btn @click="close" icon>
                  <v-icon>mdi-close</v-icon>
                </v-btn>
              </div>

              <!-- Botones default en la parte inferior -->
              <div class="bottom-button-container">
                <v-btn @click="pasteFromClipboard" class="upload-button">Paste</v-btn>
                <v-btn @click="showMyQR" class="upload-button">Mi QR</v-btn>
              </div>
            </qrcode-stream>
          </v-card>

          <v-alert v-if="alertVisible" type="error" dismissible @click:close="alertVisible = false">
            {{ alertMessage }}
          </v-alert>
        </v-col>
      </v-row>
    </v-container>
  </v-app>
</template>

<script>
import { QrcodeStream, QrcodeCapture } from 'vue-qrcode-reader'
import { shallowRef, ref, onMounted } from 'vue'
import { VApp, VContainer, VRow, VCol, VCard, VAlert, VBtn, VIcon, VFileInput } from 'vuetify/components'

const PTypes = {
  BOLT11: 'BOLT11',
  LnAddress: 'LnAddress',
  LNURLP: 'LNURLP'
}

const regexs = shallowRef([
  { name: PTypes.BOLT11, regex: /^(lnbc|lntb|lnsb|lnbcrt)([0-9]{1,}[munp]?)?(1)([02-9ac-hj-np-z]{1,}){6,}$/i },
  { name: PTypes.LnAddress, regex: /^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$/ },
  { name: PTypes.LNURLP, regex: /^(lnurl1){1}[0-9ac-hj-np-z]+$/i }
])

export default {
  components: { QrcodeStream, QrcodeCapture, VApp, VContainer, VRow, VCol, VCard, VAlert, VBtn, VIcon, VFileInput },

  data() {
    return {
      isValid: undefined,
      paused: false,
      result: null,
      facingMode: 'environment',
      noRearCamera: false,
      noFrontCamera: false,
      torchActive: false,
      torchNotSupported: false,
      videoInputDevices: [],
      currentDeviceIndex: 0,
      alertVisible: false,
      alertMessage: ''
    }
  },

  computed: {
    validationPending() {
      return this.isValid === undefined && this.paused
    },
    validationSuccess() {
      return this.isValid === true
    },
    validationFailure() {
      return this.isValid === false
    }
  },

  mounted() {
    this.getVideoInputDevices()
    this.requestFullscreen()
  },

  methods: {
    async getVideoInputDevices() {
      const devices = await navigator.mediaDevices.enumerateDevices()
      this.videoInputDevices = devices.filter(device => device.kind === 'videoinput')
    },
    onError(error) {
      if (error.name === 'OverconstrainedError') {
        this.facingMode === 'environment' ? this.noRearCamera = true : this.noFrontCamera = true
      }
      console.error(error)
      this.showAlert(error.message)
    },
    resetValidationState() {
      this.isValid = undefined
    },
    async onDetect([firstDetectedCode]) {
      this.result = firstDetectedCode.rawValue
      this.paused = true
      console.log(this.result)
      this.isValid = regexs.value.some(({ regex }) => regex.test(this.result))
      await this.timeout(2000)
      this.paused = false
    },
    timeout(ms) {
      return new Promise(resolve => setTimeout(resolve, ms))
    },
    switchCamera() {
      if (this.videoInputDevices.length > 0) {
        this.currentDeviceIndex = (this.currentDeviceIndex + 1) % this.videoInputDevices.length
        this.facingMode = this.videoInputDevices[this.currentDeviceIndex].deviceId
        console.log(`Switched to camera: ${this.videoInputDevices[this.currentDeviceIndex].label}`)
      }
    },
    onCameraOn(capabilities) {
      console.log(capabilities)
      this.torchNotSupported = !(capabilities && capabilities.torch)
    },
    async pasteFromClipboard() {
      try {
        const text = await navigator.clipboard.readText()
        this.result = text
        console.log(this.result)
        this.isValid = regexs.value.some(({ regex }) => regex.test(this.result))
        this.paused = true
        await this.timeout(2000)
        this.paused = false
      } catch (err) {
        console.error('Failed to read clipboard contents: ', err)
        this.showAlert('Failed to read clipboard contents')
      }
    },
    handleFileUpload(event) {
      const file = event.target.files[0]
      if (file) {
        const reader = new FileReader()
        reader.onload = (e) => {
          const img = new Image()
          img.onload = () => {
            const canvas = document.createElement('canvas')
            canvas.width = img.width
            canvas.height = img.height
            const ctx = canvas.getContext('2d')
            ctx.drawImage(img, 0, 0, img.width, img.height)
            const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height)
            this.$refs.qrcodeCapture.decode(imageData)
          }
          img.src = e.target.result
        }
        reader.readAsDataURL(file)
      }
    },
    close() {
      console.log('Close button clicked')
    },
    showMyQR() {
      console.log('Mi QR button clicked')
    },
    triggerFileUpload() {
      this.$el.querySelector('input[type="file"]').click()
    },
    toggleTorch() {
      this.torchActive = !this.torchActive
    },
    showAlert(message) {
      this.alertMessage = message
      this.alertVisible = true
    },
    requestFullscreen() {
      const elem = this.$refs.wrapper
      if (elem.requestFullscreen) {
        elem.requestFullscreen()
      } else if (elem.mozRequestFullScreen) { // Firefox
        elem.mozRequestFullScreen()
      } else if (elem.webkitRequestFullscreen) { // Chrome, Safari y Opera
        elem.webkitRequestFullscreen()
      } else if (elem.msRequestFullscreen) { // IE/Edge
        elem.msRequestFullscreen()
      }
    }
  }
}
</script>

<style scoped>
.camera-container {
  width: 100vw; /* Ancho completo */
  height: 100vh; /* Alto completo */
  margin: 0;
  padding: 0;
  position: fixed; /* Para que ocupe todo el viewport */
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  border: none;
  border-radius: 0;
  overflow: hidden;
}

.qr-frame {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 250px; /* Tamaño fijo del marco del QR */
  height: 250px; /* Tamaño fijo del marco del QR */
  transform: translate(-50%, -50%);
  border: 4px solid rgba(255, 255, 255, 0.8);
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.5);
  z-index: 10;
}

.fullscreen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw; /* Ancho completo */
  height: 100vh; /* Alto completo */
  margin: 0;
  padding: 0;
  background-color: black;
  z-index: 1000;
}

/* Contenedor de botones de íconos en la parte superior */
.top-button-container {
  position: absolute;
  top: 20px; /* Ajusta la distancia desde la parte superior */
  left: 0;
  right: 0;
  display: flex;
  justify-content: center; /* Centra los botones horizontalmente */
  gap: 10px; /* Espacio entre los botones */
  z-index: 1000;
  padding: 0;
}

/* Contenedor de botones default en la parte inferior */
.bottom-button-container {
  position: absolute;
  bottom: 20px; /* Ajusta la distancia desde la parte inferior */
  left: 0;
  right: 0;
  display: flex;
  justify-content: center; /* Centra los botones horizontalmente */
  gap: 10px; /* Espacio entre los botones */
  z-index: 1000;
  padding: 0;
}

.upload-button {
  margin: 10px;
}
</style>