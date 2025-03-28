<script setup>
import { ref, watch } from "vue";

const step = ref(1);
const videoRef = ref(null);
const canvasRef = ref(null);
const capturedPhotos = ref([]); // Store 3 photos
const selectedFrame = ref(null);
let stream = null;

const frames = [
  { id: 1, name: "Neon Glow", class: "border-10 border-blue-500 shadow-neon" },
  { id: 2, name: "Pink Glow", class: "border-10 border-pink-500 shadow-pink-neon" },
  { id: 3, name: "Gold Royal", class: "border-10 border-yellow-500 shadow-xl" },
];

// Start camera
const startCamera = async () => {
  try {
    if (stream) {
      stream.getTracks().forEach((track) => track.stop());
    }
    stream = await navigator.mediaDevices.getUserMedia({ video: true });
    if (videoRef.value) {
      videoRef.value.srcObject = stream;
    }
  } catch (error) {
    console.error("Error accessing camera:", error);
  }
};

// Capture photo from webcam (Allow capturing 3 photos)
const capturePhoto = () => {
  const canvas = canvasRef.value;
  const video = videoRef.value;
  if (canvas && video && capturedPhotos.value.length < 3) {
    const ctx = canvas.getContext("2d");
    ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
    capturedPhotos.value.push(canvas.toDataURL("image/png"));
  }
};

// Select a frame
const selectFrame = (frame) => {
  selectedFrame.value = frame;
};

// Move to the next step
const nextStep = () => {
  if (step.value === 2 && capturedPhotos.value.length < 3) {
    alert("Please capture 3 photos before proceeding.");
    return;
  }
  if (step.value < 3) step.value++;
};

// Move to the previous step
const prevStep = () => {
  if (step.value > 1) step.value--;
};

// Watch for step changes and restart the camera on Step 2
watch(step, (newStep) => {
  if (newStep === 2) {
    capturedPhotos.value = []; // Reset photos when revisiting
    startCamera();
  }
});

// Download section
const finalCanvasRef = ref(null);

const downloadPhoto = () => {
  if (capturedPhotos.value.length < 3 || !selectedFrame.value) {
    alert("Please capture 3 photos and select a frame before downloading.");
    return;
  }

  const canvas = finalCanvasRef.value;
  const ctx = canvas.getContext("2d");

  // Set dimensions for image and borders
  const imgWidth = 512;
  const imgHeight = 512;
  const outerBorder = 50; // Thicker outer border
  const innerBorder = 30; // Thicker inner border
  const spacingBetween = 30; // Space between frames
  const canvasWidth = imgWidth + outerBorder * 2;
  const canvasHeight = (imgHeight + outerBorder * 2) * 3 + spacingBetween * 2;

  canvas.width = canvasWidth;
  canvas.height = canvasHeight;

  // Dynamically set frame color based on selected frame
  const outerFrameColor =
    selectedFrame.value.class.includes("border-yellow") ? "#FFD700" : // Gold
    selectedFrame.value.class.includes("border-blue") ? "#00FFFF" : // Cyan
    selectedFrame.value.class.includes("border-pink") ? "#FF1493" : // Deep Pink
    "#000"; // Default to black if no match

  const innerFrameColor = "#FFFFFF"; // White inner frame

  // Load all images
  const imagePromises = capturedPhotos.value.map((photo) => {
    return new Promise((resolve) => {
      const img = new Image();
      img.src = photo;
      img.onload = () => resolve(img);
    });
  });

  Promise.all(imagePromises).then((images) => {
    let yPos = 0; // Start at the top

    images.forEach((image) => {
      // Outer frame (uses selected frame color)
      ctx.fillStyle = outerFrameColor;
      ctx.fillRect(0, yPos, canvasWidth, imgHeight + outerBorder * 2);

      // Inner white frame
      ctx.fillStyle = innerFrameColor;
      ctx.fillRect(outerBorder / 2, yPos + outerBorder / 2, canvasWidth - outerBorder, imgHeight + outerBorder);

      // Draw the image inside the frame
      ctx.drawImage(image, outerBorder + innerBorder / 2, yPos + outerBorder + innerBorder / 2, imgWidth - innerBorder, imgHeight - innerBorder);

      // Move down for next image
      yPos += imgHeight + outerBorder * 2 + spacingBetween;
    });

    // Convert canvas to image and trigger download
    const link = document.createElement("a");
    link.href = canvas.toDataURL("image/png");
    link.download = "framed-photo.png";
    link.click();
  });
};

</script>



<template>
  <div class="min-h-screen flex flex-col justify-center items-center bg-gray-100">
    <div class="container mx-auto max-w-4xl p-5 bg-amber-50 rounded-lg shadow-lg">
      
      <div v-if="step === 1" class="text-center">
        <h1 class="text-3xl font-bold">Welcome to the Photo Booth!</h1>
        <p class="mt-2">Follow the steps to capture and style your photo.</p>
      </div>

      <div v-if="step === 2" class="text-center container mx-auto p-5">
        <h1 class="text-3xl font-bold">Step 2: Capture Your Photos</h1>
        <p class="mt-2">Capture up to 3 photos.</p>

        <div class="relative flex justify-center items-center mt-6">
          <video ref="videoRef" class="w-full max-w-md h-auto rounded-lg border-4 border-gray-400 shadow-lg" autoplay playsinline></video>
          <canvas ref="canvasRef" class="hidden"></canvas>
        </div>

        <button 
          @click="capturePhoto"
          class="bg-green-500 text-white px-6 py-3 rounded-lg mt-4 shadow-lg hover:bg-green-600 transition">
          Capture Photo
        </button>

        <div v-if="capturedPhotos.length > 0" class="mt-6">
          <h2 class="text-xl font-bold">Captured Photos:</h2>
          <div class="flex justify-center flex-wrap gap-4 mt-2">
            <img v-for="(photo, index) in capturedPhotos" :key="index" :src="photo" 
                 class="w-32 h-32 border-4 rounded-lg shadow-lg">
          </div>
        </div>
      </div>

      <div v-if="step === 3" class="text-center container mx-auto p-5 min-h-screen flex flex-col justify-center">
        <h1 class="text-3xl font-bold">Step 3: Choose a Frame</h1>
        <p class="mt-2">Pick a frame for your photos.</p>

        <div class="grid grid-cols-2 sm:grid-cols-3 gap-4 mt-6">
          <div 
            v-for="frame in frames" 
            :key="frame.id" 
            @click="selectFrame(frame)"
            class="border p-3 rounded-lg cursor-pointer hover:shadow-lg text-center transition-transform transform hover:scale-105"
            :class="{ 'border-blue-500 shadow-md': selectedFrame?.id === frame.id }"
          >
            <div :class="'w-24 h-24 mx-auto rounded-lg ' + frame.class"></div>
            <p class="mt-2 font-medium">{{ frame.name }}</p>
          </div>
        </div>

        <div v-if="selectedFrame && capturedPhotos.length > 0" class="mt-8">
  <h2 class="text-xl font-bold">Your Final Photo:</h2>
  <div class="flex flex-col items-center mt-4">
    <div 
      v-for="(photo, index) in capturedPhotos" 
      :key="index" 
      class="relative p-6 border-[32px] rounded-lg" 
      :class="selectedFrame.class"
    >
      <img :src="photo" class="w-64 h-[200px]">
    </div>
  </div>
</div>


        <canvas ref="finalCanvasRef" class="hidden"></canvas>

        <button 
          @click="downloadPhoto"
          class="bg-green-500 text-white px-6 py-3 rounded-lg mt-6 shadow-lg hover:bg-green-600 transition">
          Download Photos
        </button>
      </div>

      <div class="flex justify-between mt-6">
        <button v-if="step > 1" @click="prevStep" class="bg-gray-500 text-white px-6 py-3 rounded-lg shadow-lg hover:bg-gray-600 transition">
          Back
        </button>
        <button v-if="step < 3" @click="nextStep" class="bg-blue-500 text-white px-6 py-3 rounded-lg ml-auto shadow-lg hover:bg-blue-600 transition">
          Continue
        </button>
      </div>
    </div>
  </div>
</template>


<style scoped>
/* Custom shadow for neon effect */
.shadow-neon {
  box-shadow: 0 0 10px rgba(0, 255, 255, 0.8), 0 0 20px rgba(0, 255, 255, 0.6);
}

/* New Pink Glow Frame */
.shadow-pink-neon {
  box-shadow: 0 0 10px rgba(255, 20, 147, 0.8), 0 0 20px rgba(255, 20, 147, 0.6);
}
</style>
