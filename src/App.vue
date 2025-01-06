<template>
  <div>
    <strong
      v-if="careerfirstChanges !== 'careerfirst'"
      style="color: #010127; font-size: medium"
    >
      <button
        text
        fab
        small
        size="3vw"
        style="margin-right: 7px"
        @click="showQrCodeScanner = true"
        :disabled="sppuInstituteIdFlag"
      >
        Start Scanning
      </button>
    </strong>

    <v-dialog
      v-model="showQrCodeScanner"
      v-if="showQrCodeScanner === true"
      width="auto"
    >
      <v-card>
        <v-card-title>
          Scanner--1
          <v-spacer></v-spacer>
          <button
            text
            fab
            small
            size="3vw"
            style="margin-right: 7px"
            @click="toggleCamera"
          >
            Flip camera
          </button>
        </v-card-title>

        <v-card-text>
          <div id="app">
            <div>
              <video
                ref="video"
                autoplay
                playsinline
                style="width: 100%"
              ></video>
            </div>
            <div>
              <label for="zoom">Zoom: </label>
              <input
                id="zoom"
                type="range"
                min="1"
                max="5"
                step="0.1"
                v-model="zoomLevel"
                @input="adjustZoom"
              />
              <span>{{ zoomLevel }}x</span>
            </div>
            <div v-if="qrCodeData">
              <h3>Latest QR Code: {{ qrCodeData }}</h3>
            </div>
            <div v-if="qrCodeScannerError" style="color: red">
              <p>Error: {{ qrCodeScannerError }}</p>
            </div>
          </div>
        </v-card-text>

        <v-card-actions>
          <v-spacer></v-spacer>
          <button
            color="#FFFFFF"
            text
            @click="showQrCodeScanner = false"
            style="background-color: #ff4f1f; margin: 0 auto; display: block"
          >
            Ok
          </button>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>

<script>
import { BrowserQRCodeReader } from "@zxing/library";
import "@mdi/font/css/materialdesignicons.css";
export default {
  data() {
    return {
      qrResultByLiveData: {},
      qrCodeStatusNumber: null,
      studentAccess: [],
      sppuInstituteIdFlag: false,
      isMobileView: false,
      selectedActivity: 1,
      userData: null,
      user: null,
      logoutDialog: false,
      drawer: false,
      formLink: {},
      brandingName: "",
      brandingLogo: "",
      isFormModalOpen: false,
      showQrCodeScanner: false,
      qrCodeScannerError: "",
      showQrAttendanceResult: false,
      attendanceResult: false,
      careerfirstChanges: "",
      camera: "auto",
      fingerprint: "",
      codeReader: null,
      currentStream: null,
      currentTrack: null,
      zoomLevel: 1,
      qrCodeData: null,
      currentFacingMode: "auto",
      controls: null,
      isInitializing: false,
    };
  },
  mounted() {
    // Automatically start the camera when the component is mounted
    if (this.showQrCodeScanner) {
      this.initializeScanner(); // Initialize the scanner
      this.toggleCamera(); // Start scanning and toggle the camera
    }
  },
  watch: {
    showQrCodeScanner(newValue) {
      if (newValue) {
        this.initializeScanner(); // Initialize the scanner first
        this.toggleCamera();
      } else {
        this.stopScanning();
        if (this.currentFacingMode == "environment") {
          this.toggleCamera();
        }
      }
    },
  },
  methods: {
    stopScanning() {
      if (this.controls) {
        this.controls.stop();
        this.controls = null;
      }
      if (this.currentStream) {
        this.currentStream.getTracks().forEach((track) => track.stop());
        this.currentStream = null;
        this.currentTrack = null;
      }
      console.log("Scanning stopped");
    },
    async toggleCamera() {
      try {
        await this.stopScanning();
        this.currentFacingMode =
          this.currentFacingMode === "environment" ? "user" : "environment";
        console.log("Switching to camera mode:", this.currentFacingMode);
        await new Promise((resolve) => setTimeout(resolve, 500));
        await this.startScanning();
      } catch (error) {
        console.error("Error toggling camera:", error);
        this.qrCodeScannerError = `Failed to switch camera: ${error.message}`;
      }
    },
    adjustZoom() {
      if (this.currentTrack) {
        const capabilities = this.currentTrack.getCapabilities();
        if (capabilities.zoom) {
          this.currentTrack
            .applyConstraints({
              advanced: [{ zoom: parseFloat(this.zoomLevel) }],
            })
            .catch((error) => {
              console.error("Error applying zoom:", error);
            });
        }
      }
    },
    async initializeScanner() {
      try {
        this.isInitializing = true;
        this.codeReader = new BrowserQRCodeReader();
        const hints = new Map();
        hints.set(2, true);
        this.codeReader.hints = hints;

        console.log("Scanner initialized:", this.codeReader); // Debug log
        await this.startScanning();
      } catch (error) {
        console.error("Failed to initialize scanner:", error);
        this.qrCodeScannerError = `Scanner initialization failed: ${error.message}`;
      }
    },
    async startScanning() {
      try {
        if (!this.codeReader) {
          throw new Error("Scanner is not initialized");
        }

        if (this.currentStream) {
          this.stopScanning();
        }

        const videoElement = this.$refs.video;
        if (!videoElement) {
          throw new Error("Video element not found");
        }

        const constraints = {
          video: {
            facingMode: { exact: this.currentFacingMode },
            width: { ideal: 1280 },
            height: { ideal: 720 },
          },
        };

        this.controls = await this.codeReader.decodeFromConstraints(
          constraints,
          videoElement,
          (result, error) => {
            if (result) {
              try {
                console.log("QR code result: ", result.getText());
                this.qrCodeData = result.getText(); // Save the decoded QR code data
              } catch (err) {
                console.error("Error processing QR code:", err);
                this.qrCodeScannerError = "Invalid QR code format";
              }
            }
            if (error && error.name !== "NotFoundException") {
              console.error("QR Code scanning error:", error);
            }
          }
        );

        if (videoElement.srcObject) {
          this.currentStream = videoElement.srcObject;
          this.currentTrack = this.currentStream.getVideoTracks()[0];
          console.log(
            "Scanner started with facing mode:",
            this.currentFacingMode
          );
        } else {
          throw new Error("Failed to initialize video stream");
        }
      } catch (error) {
        console.error("Error starting scanner:", error);
        this.qrCodeScannerError = `Failed to start scanner: ${error.message}`;
      }
    },
  },
  beforeDestroy() {
    this.stopScanning();
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
video {
  width: 100%;
  max-width: 600px;
  height: auto;
  border: 2px solid #ccc;
}

button {
  margin: 10px;
  padding: 10px 20px;
  font-size: 16px;
  cursor: pointer;
}

button:hover {
  background-color: #f0f0f0;
}

input[type="range"] {
  width: 200px;
}

label {
  font-size: 16px;
}

span {
  font-size: 18px;
  font-weight: bold;
  margin-left: 10px;
}
</style>
